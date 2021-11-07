# bytes.Buffer в Go: оптимизации, которые не работают

Многие [Go](https://golang.org/) программисты знакомы с [bytes.Buffer](https://golang.org/pkg/bytes/#Buffer). Одно из его преимуществ состоит в том, что он позволяет избегать выделений памяти в куче по той же схеме, что и "оптимизация коротких строк" ([small buffer/size optimization](https://nullprogram.com/blog/2016/10/07/)):

```go
type Buffer struct {
    bootstrap [64]byte // для избежания аллокации малых слайсов в куче
    // ... другие поля
}
```

Есть только одна проблема. Эта оптимизация [не работает](https://github.com/golang/go/issues/7921).

К концу этой статьи вы узнаете, почему эта оптимизация не работает и что мы можем с этим сделать.

<cut/>

# Как было по задумке, "small buffer optimization"

Введём немного упрощённое определение `bytes.Buffer`:

```go
const smallBufSize int = 64
type Buffer struct {
    bootstrap [smallBufSize]byte
    buf       []byte
}
```

Когда мы выполняем действия над `Buffer`, например, вызываем метод `Buffer.Write`, запись всегда производится в `buf`, однако перед этой записью внутри каждого подобного метода запускается `Buffer.grow(n)`, который делает так, чтобы в этом самом слайсе было достаточно места для очередных `n` байт.

Выглядеть `grow` может примерно так:

```go
func (b *Buffer) grow(n int) {
	// Код сильно упрощён по сравнению с настоящей реализацией bytes.Buffer.
	l := len(b.buf) // Фактическая длина Buffer
	need := n + l
	have := cap(b.buf) - l
	if have >= need {
		b.buf = b.buf[:need]
		return
	}
	if need <= smallBufSize {
		// Может быть только первой аллокацией,
		// копирование не нужно.
		b.buf = b.bootstrap[:]
	} else {
		// growFactor - функция подбора следующего размера аллокации.
		// В простейшем случае это need или need*2.
		newBuf := make([]byte, need, growFactor(need))
		copy(newBuf, b.buf)
		b.buf = newBuf
	}
}
```

<spoiler title="Допущения, которые использовалось в нашей реализации Buffer.grow"><hr>

Мы делаем допущение, что `len(b.buf)` - это фактическая длина данных в Buffer, что требовало бы от `Write` методов использования append для добавления новых байтов в слайс. В `bytes.Buffer` из стандартной библиотеки это не так, но для примера это является несущественной деталью реализации.

<hr></spoiler>

Если `b` выделен на стеке, то и `bootstrap` внутри него выделен на стеке, а, значит, слайс `b.buf` будет переиспользовать память внутри `b`, не требуя дополнительной аллокации.

Когда `grow` выявляет, что `bootstrap` массива уже недостаточно, будет выполнено создание нового, "настоящего" слайса, куда затем будут скопированы элементы из прошлого хранилища (из "малого буфера"). После этого `Buffer.bootstrap` утратит актуальность. Если вызывается `Buffer.Reset`, `cap(b.buf)` останется прежним, и необходимости в `bootstrap` массиве больше не будет.

# Память, убегающая в heap

Далее ожидается, что читатель хотя бы поверхностно знаком с тем, что такое [escape analysis](http://www.agardner.me/golang/garbage/collection/gc/escape/analysis/2015/10/18/go-escape-analysis.html) в Go.

Рассмотрим следующую ситуацию:

```go
func f() *Buffer {     
    var b bytes.Buffer // leak.go:11:6: moved to heap: b
    return &b          // leak.go:12:9: &b escapes to heap
}
```

Здесь `b` будет выделен на куче. Причина тому - "утекающий" указатель на `b`:

```bash
$ go tool compile -m leak.go
leak.go:12:9: &b escapes to heap
leak.go:11:6: moved to heap: b
```

<spoiler title="Терминология"><hr>

В данной статье "утекает" (leaking) и "убегает" (escapes) используются почти как синонимы.

В самом компиляторе есть некоторое различие, например, значение "убегает в кучу" (x escapes to heap), но параметры функции "утекающие" (leaking param x).

Утекающий параметр означает, что переданный аргумент под этот параметр будет выделения в куче. Другими словами, leaking параметр заставляет аргументы убегать в кучу.

<hr></spoiler>

Выше был очевидный случай, но как насчёт этого:

```go
func length() int {
	var b bytes.Buffer
	b.WriteString("1")
	return b.Len()
}
```

Здесь нам нужен всего лишь 1 байт, всё влезает в `bootstrap`, сам буфер локален и не "убегает" из функции. Возможно, вы будете удивлены, но результат будет тот же, аллокация `b` на куче.

<img src="https://habrastorage.org/webt/_d/3w/jr/_d3wjrildbyauunsvvv5oq7zqpo.jpeg">

Для уверенности можно проверить это с помощью бенчмарка:

```
BenchmarkLength-8  20000000  90.1 ns/op  112 B/op  1 allocs/op
```

<spoiler title="Листинг бенчмарка"><hr>

```go
package p

import (
	"bytes"
	"testing"
)

func length() int {
	var b bytes.Buffer
	b.WriteString("1")
	return b.Len()
}

func BenchmarkLength(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = length()
	}
}
```

<hr></spoiler>

<spoiler title="Объяснение 112 B/op"><hr>

Когда runtime просит у аллокатора `N` байт, не обязательно, что будет выделено ровно `N` байт. 

> Все результаты ниже указаны для комбинации `GOOS=linux` и `GOARCH=AMD64`.

```go
package benchmark

import "testing"

//go:noinline
func alloc9() []byte {
	return make([]byte, 9)
}

func BenchmarkAlloc9(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = alloc9()
	}
}
```

Если запустить `go test -bench=. -benchmem` с этим тестом:

```
BenchmarkAlloc9-8  50000000  33.5 ns/op  16 B/op  1 allocs/op
```

Запрошено 9 байтов, выделено 16. Теперь вернёмся к `bytes.Buffer`:

```go
fmt.Println(unsafe.Sizeof(bytes.Buffer{})) => 104
```

Посмотрим на [$GOROOT/src/runtime/sizeclasses.go](https://github.com/golang/go/blob/b444215116e8d4a63f70f1a1b5a31f60a41a7f75/src/runtime/sizeclasses.go#L13-L14):

```
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         32        8192      256           0     46.88%
//     4         48        8192      170          32     31.52%
//     5         64        8192      128           0     23.44%
//     6         80        8192      102          32     19.07%
//     7         96        8192       85          32     15.95%
//     8        112        8192       73          16     13.56%
// ... остальные
```

В 96 байт не влезает, выбирается 112.

<hr></spoiler>

Но почему же это происходит?

# Что происходит и почему

Некоторый анализ ситуации можно найти в упомянутом в самом начале [issue](https://github.com/golang/go/issues/7921).<br>
Там же есть [простой reproducer](https://github.com/golang/go/issues/7921#issuecomment-285855447).

Проблемное место как раз в присваивании `b.buf = b.bootstrap[:]`. Этот код заставляет escape analysis считать, что `b.bootstrap` "убегает", а поскольку это массив, то он хранится внутри самого объекта, а значит на куче нужно выделять весь `b`.

Если бы bootstrap был слайсом, а не массивом, то этого бы не происходило, потому что есть adhoc оптимизация для присваиваний слайсов из объекта в сам объект:

```go
// Несмотря на то, что здесь практически идентичная ситуация,
// object не обязателен к размещению на куче.
object.buf1 = object.buf2[a:b]
```

Ответ почему же эта оптимизация не работает для массивов уже был сформулирован выше, но вот выжимка из самого [esc.go#L835-L866](https://github.com/golang/go/blob/b444215116e8d4a63f70f1a1b5a31f60a41a7f75/src/cmd/compile/internal/gc/esc.go#L835-L866) (по ссылке выделен весь код оптимизации):

```go
// Note, this optimization does not apply to OSLICEARR,
// because it does introduce a new pointer into b that was not already there
// (pointer to b itself). After such assignment, if b contents escape,
// b escapes as well. If we ignore such OSLICEARR, we will conclude
// that b does not escape when b contents do.
```

Здесь стоит добавить, что для анализатора указателей есть несколько уровней "утечек", основные из них:
1. Убегает сам объект (b escapes). В этом случае выделять на куче нужно сам объект.
2. Убегают элементы объекта (b contents escape). В этом случае указатели в объекте считаются убегающими.

Случай с массивом особенен тем, что если утекает массив, должен утекать и сам объект его содержащий.

escape analysis выводит решение о том, можно ли расместить объект на стеке или нет, полагаясь только на информацию, которая доступна в теле анализируемой функции. Метод `Buffer.grow` принимает `b` по указателю, поэтому для него требуется вычислить пригодное место для размещения. Поскольку в случае массива мы не можем отличить `"b escape"` от `"b contents escape"`, приходится быть более пессимистичными и приходить к выводу, что `b` размещать на стеке не безопасно.

Предположим обратное, что `self-assignment` паттерн разрешает для массивов всё то же, что и для слайсов:

```go
package example

var sink interface{}

type bad struct {
	array [10]byte
	slice []byte
}

func (b *bad) bug() {
	b.slice = b.array[:] // ignoring self-assignment to b.slice
	sink = b.array       // b.array escapes to heap
	// b does not escape
}
```

Решение разместить `b` на стеке в этой ситуации приведёт к катастрофе: после выхода из функции, внутри которой был создан `b`, память, на которую будет ссылаться `sink` будет ничем иным, как мусором.

# Указатели на массивы

Представим, что наш `Buffer` был объявлен несколько иначе:

```diff
const smallBufSize int = 64
type Buffer struct {
-   bootstrap [smallBufSize]byte
+   bootstrap *[smallBufSize]byte
    buf       []byte
}
```

В отличие от обычного массива, указатель на массив не будет хранить все элементы внутри самого `Buffer`. Это означает, что если выделение `bootstrap` на куче не влечёт за собой выделение `Buffer` на куче. Поскольку escape analysis умеет выделять поля-указатели на стеке, когда это возможно, мы можем считать, что такое определение `Buffer` более удачно.

Но это в теории. На практике, указатель на массив не имеет особой обработки и попадает в ту же категорию, что и слайс от обычного массива, что не совсем правильно. [CL133375: cmd/compile/internal/gc: handle array slice self-assign in esc.go](https://golang.org/cl/133375) направлен на исправление этой ситуации.

Предположим, что это изменение было принято в компилятор Go.

# Zero value, который мы потеряли

К сожалению, переход от `[64]byte` к `*[64]byte` имеет проблему: мы теперь не можем использовать `bootstrap` без явной его инициализации, нулевое значение `Buffer` перестаёт быть полезным, нам нужен конструктор.

```go
func NewBuffer() Buffer {
    return Buffer{bootstrap: new(*[smallBufSize]byte)}
}
```

Мы возвращаем `Buffer`, а не `*Buffer`, для избежания проблем с анализом указателей (он в Go очень консервативен), а с учётом того, что `NewBuffer` всегда встраивается в место вызова, лишнее копирование происходить не будет.

После встраивания тела `NewBuffer` в место вызова escape analysis может попробовать доказать, что `new(*[smallBufSize]byte)` не превосходит по своему времени жизни время жизни фрейма функции, в котором он вызван. Если это так, то аллокация будет на стеке.

# Intel bytebuf

Описанная выше оптимизация применена в пакете [intel-go/bytebuf](https://github.com/intel-go/bytebuf).

Эта библиотека экспортирует тип `bytebuf.Buffer`, который дублирует `bytes.Buffer` на 99.9%. Все изменения сводятся к введению конструктора (`bytebuf.New`) и указателю на массив вместо обычного массива:

```diff
type Buffer struct {
	buf       []byte   // contents are the bytes buf[off : len(buf)]
	off       int      // read at &buf[off], write at &buf[len(buf)]
- 	bootstrap [64]byte // helps small buffers avoid allocation.
+ 	bootstrap *[64]byte // helps small buffers avoid allocation.
	lastRead  readOp   // last read operation (for Unread*).
}
```

Вот сравнение производительности с `bytes.Buffer`:

```
name            old time/op    new time/op    delta
String/empty-8     138ns ±13%      24ns ± 0%   -82.94%  (p=0.000 n=10+8)
String/5-8         186ns ±11%      60ns ± 1%   -67.82%  (p=0.000 n=10+10)
String/64-8        225ns ±10%     108ns ± 6%   -52.26%  (p=0.000 n=10+10)
String/128-8       474ns ±17%     338ns ±13%   -28.57%  (p=0.000 n=10+10)
String/1024-8      889ns ± 0%     740ns ± 1%   -16.78%  (p=0.000 n=9+10)

name            old alloc/op   new alloc/op   delta
String/empty-8      112B ± 0%        0B       -100.00%  (p=0.000 n=10+10)
String/5-8          117B ± 0%        5B ± 0%   -95.73%  (p=0.000 n=10+10)
String/64-8         176B ± 0%       64B ± 0%   -63.64%  (p=0.000 n=10+10)
String/128-8        368B ± 0%      256B ± 0%   -30.43%  (p=0.000 n=10+10)
String/1024-8     2.16kB ± 0%    2.05kB ± 0%    -5.19%  (p=0.000 n=10+10)

name            old allocs/op  new allocs/op  delta
String/empty-8      1.00 ± 0%      0.00       -100.00%  (p=0.000 n=10+10)
String/5-8          2.00 ± 0%      1.00 ± 0%   -50.00%  (p=0.000 n=10+10)
String/64-8         2.00 ± 0%      1.00 ± 0%   -50.00%  (p=0.000 n=10+10)
String/128-8        3.00 ± 0%      2.00 ± 0%   -33.33%  (p=0.000 n=10+10)
String/1024-8       3.00 ± 0%      2.00 ± 0%   -33.33%  (p=0.000 n=10+10)
```

Вся остальная информация доступна в [README](https://github.com/intel-go/bytebuf/blob/master/README.md).

Из-за невозможности использовать zero value и привязке к конструирующей функции `New`, применить эту оптимизацию к `bytes.Buffer` не представляется возможным.

Единственный ли это способ сделать более быстрый `bytes.Buffer`? Ответ - нет. Но это точно способ, требующий минимальных изменений в реализации.

# Планы на escape analysis

В текущем виде, escape analysis в Go довольно слаб. Почти любые операции со значениями-указателями ведут к выделениям на куче, даже если это не является оправданным решением.

Большую часть времени, которое я уделяю проекту [golang/go](https://github.com/golang/go), постараюсь направить на решение именно этих проблем, так что в ближайшем релизе (1.12) возможны некоторые улучшения.

О результатах и деталях внутреннего устройства этой части компилятора вы сможете прочитать в одной из следующих моих статей. Постараюсь также предоставить набор рекомендаций, который поможет в некоторых случаях структурировать код так, чтобы в нём было меньше нежелательных выделений памяти.
