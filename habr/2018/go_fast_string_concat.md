# Ускорение конкатенации строк в Go своими руками

![](https://habrastorage.org/webt/36/bi/u0/36biu0kvugh_maxmmyhmuvqlrs8.png)

Сегодня мы будем разгонять склеивание коротких строк в Go на 30%. Причём для этого нам не нужно будет модифицировать сам Go, всё это будет реализованно в виде [сторонней библиотеки](https://github.com/Quasilyte/concat).

Под катом вас ждут:

* Сравнение `+`, `strings.Builder` и собственной функции конкатенации
* Детали внутреннего устройства строк в Go
* Совсем немного ассемблера

Данную статью можно также считать предлогом обсудить [CL123256: runtime,cmd/compile: specialize concatstring2](https://go-review.googlesource.com/c/go/+/123256). Идеи по улучшению этого change list'а приветствуются.

<cut/>

# Сразу результаты

Сравнение производилось с `go tip` (master) версией компилятора. Аналогичные результаты вы можете получить на версиях примерно с Go 1.5. Последним значительным изменением функции `concatstrings` был [CL3120: cmd/gc: allocate buffers for non-escaped strings on stack](https://go-review.googlesource.com/c/3120/).

```
BenchmarkConcat2Operator-8   	20000000	        83.8 ns/op
BenchmarkConcat2Builder-8    	20000000	        70.9 ns/op
BenchmarkConcat2-8           	20000000	        62.1 ns/op
BenchmarkConcat3Operator-8   	20000000	        104  ns/op
BenchmarkConcat3Builder-8    	20000000	        89.9 ns/op
BenchmarkConcat3-8           	20000000	        82.1 ns/op
```

`ConcatOperator` использует `+`.
`ConcatBuilder` использует `strings.Builder` с правильной пред-аллокацией.
`Concat` использует функцию, которую мы реализуем в рамках этой истории.

Сравнение через [benchstat](https://godoc.org/golang.org/x/perf/cmd/benchstat):

```
name       old time/op  new time/op  delta
Concat2-8  84.2ns ± 1%  62.7ns ± 2%  -25.49%  (p=0.000 n=9+10)
Concat3-8   103ns ± 3%    83ns ± 4%  -19.83%  (p=0.000 n=10+9)
```

Ассемблерная реализация под `GOARCH=AMD64` немного быстрее и обладает дополнительной оптимизацией, которая присутствует у встроенного оператора `+`, но об этом ниже:

```
name       old time/op  new time/op  delta
Concat2-8  84.2ns ± 1%  57.1ns ± 3%  -32.20%  (p=0.000 n=9+9)
```

Ассемблерную функцию будем брать как 100% производительности (относительно остальных рассматриваемых реализаций).

> Результаты для более длинных строк можно увидеть в [README.md](https://github.com/Quasilyte/concat). Чем длиннее строка, тем менее выражена разница между реализациями.

# Наивная конкатенация

Самым простым решением является использование оператора `+`.

Семантика этого оператора такая: взять две строки и вернуть строку-результат, которая содержит сцепление обеих строк. При этом нет гарантии, что будет возвращена новая строка. Например, если происходит сцепление пустой строки и любой другой, runtime может вернуть непустой аргумент, избегая необходимости выделять новую память и копировать туда данные.

Но, как видно из результатов в начале статьи, это самый медленный способ.

```go
func concat2operator(x, y string) string { 
	return x + y
}
```

> Оценка производительности: **67.8%**.

# strings.Builder

Не так давно в Go добавили новый тип - [strings.Builder](https://golang.org/src/strings/builder.go). Это аналог `bytes.Buffer`, но при вызове метода `String()` не происходит повторного выделения памяти и копирования данных.

В отличие от `bytes.Buffer`, builder не имеет оптимизации [малого буфера](https://github.com/golang/go/blob/3a28a711db15efd97c7675fccf0d2d0f2245a99b/src/bytes/buffer.go#L20) и, следовательно, предварительно аллоцированной памяти под строку. Если не использовать метод `Grow`, производительность будет хуже, чем в случае с `bytes.Buffer`. Несколько регрессий в Go 1.11 вызваны именно этой особенностью (см. [CL113235](https://go-review.googlesource.com/c/go/+/113235)).

В нашем коде, для чистоты эксперимента, мы будем избегать этой ошибки.

```go
func concat2builder(x, y string) string {
	var builder strings.Builder
	builder.Grow(len(x) + len(y)) // Только эта строка выделяет память
	builder.WriteString(x)
	builder.WriteString(y)
	return builder.String()
}
```

> Оценка производительности: **80.5%** (+12.7).

# Кодогенерация для конкатенации

Если посмотреть, какой код генерирует компилятор для оператора `+`, мы увидим вызовы функций `concatstring2`, `concatstring3` и так далее (до `concatstring5` включительно).

```go
func concat2codegen(x, y) string { return x + y }
// => CALL	runtime.concatstring2(SB)

func concat3codegen(x, y, z) string { return x + y + z }
// => CALL	runtime.concatstring3(SB)
```

Заглянем в сам [runtime/string.go](https://github.com/golang/go/blob/master/src/runtime/string.go):

```go
func concatstring2(buf *tmpBuf, a [2]string) string {
	return concatstrings(buf, a[:])
}

func concatstring3(buf *tmpBuf, a [3]string) string {
	return concatstrings(buf, a[:])
}
```

Значит, осталось изучить функцию `concatstrings`.
Полный листинг доступен ниже под спойлером, а вот высокоуровневое описание:

1. Параметр `buf` может быть `nil`. Этот буфер выделяется компилятором, если строка не "убегает" из области своего определения. Если же строка живёт дольше, чем фрейм, то этот буфер всегда будет `nil` (как чаще всего и происходит). Однако если этот буфер доступен, получится избежать аллокации в случае, если результат в него влезает (его размер - 32 байта).
2. Если все строки, кроме одной, пустые, функция вернёт эту строку. Но при этом выделенные на стеке и покидающие свой фрейм строки минуют этой оптимизации, чтобы вызывающая сторона не получила уже освобождённую память.
3. Далее все строки копируются в новую память.

<spoiler title="Полный листинг функции concatstrings">
```go
// concatstrings implements a Go string concatenation x+y+z+...
// The operands are passed in the slice a.
// If buf != nil, the compiler has determined that the result does not
// escape the calling function, so the string data can be stored in buf
// if small enough.
func concatstrings(buf *tmpBuf, a []string) string {
	idx := 0
	l := 0
	count := 0
	for i, x := range a {
		n := len(x)
		if n == 0 {
			continue
		}
		if l+n < l {
			throw("string concatenation too long")
		}
		l += n
		count++
		idx = i
	}
	if count == 0 {
		return ""
	}

	// If there is just one string and either it is not on the stack
	// or our result does not escape the calling frame (buf != nil),
	// then we can return that string directly.
	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx]
	}
	s, b := rawstringtmp(buf, l)
	for _, x := range a {
		copy(b, x)
		b = b[len(x):]
	}
	return s
}
```
</spoiler>

Здесь мы видим сразу несколько мест, которые могут быть оптимизированы для частного случая:
- `buf` чаще всего пустой. Когда компилятор не смог доказать, что строку безопасно размещать на стеке, передача лишнего параметра и проверка его на `nil` внутри функции дают лишь накладные расходы.
- Для частного случая при `len(a) == 2` нам не нужен цикл и вычисления можно упростить. А это самый распространённый вид конкатенации.

<spoiler title="Статистика по использованию конкатенации">
При выполнении `./make.bash` (сборка Go компилятора и stdlib) видим 445 конкатенаций с двумя операндами:

* 398 результатов "убегают". В этом случае наша специализация имеет смысл.
* 47  результатов не покидают своего фрейма.

Итого **89%** конкатенаций от двух аргументов попадают пот оптимизацию.

Для утилиты `go` имеем:

* 501 вызовов concatstring2
* 194 вызовов concatstring3
* 55  вызовов concatstring4
</spoiler>

# Версия для всех архитектур

Для реализации специализации нам потребуется знать, как в Go представлены строки. Нам важна бинарная совместимость, при этом `unsafe.Pointer` можно заменить на `*byte` без каких-либо жертв.

```go
type stringStruct struct {
	str *byte
	len int
}
```

Второй важный вывод, который мы можем сделать из рантайма: строки начинают свою жизнь мутабельными. Выделяется участок памяти, на который ссылается `[]byte`, в который записывается содержимое новой строки, и только после этого `[]byte` выбрасывается, а память, на которую он ссылался, сохраняется в `stringStruct`. 

Для тех, кому хочется больше деталей, предлагается изучить функции `rawstringtmp` и `rawstring`.

<spoiler title="runtime.rawstring">
```go
// rawstring allocates storage for a new string. The returned
// string and byte slice both refer to the same storage.
// The storage is not zeroed. Callers should use
// b to set the string contents and then drop b.
func rawstring(size int) (s string, b []byte) {
	p := mallocgc(uintptr(size), nil, false)

	stringStructOf(&s).str = p
	stringStructOf(&s).len = size

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, size}

	return
}
```
</spoiler>

Мы можем провернуть примерно то же самое, воспользовавшись тёмной стороной пакета `unsafe`:

```go
func concat2(x, y string) string {
	length := len(x) + len(y)
	if length == 0 {
		return ""
	}
	b := make([]byte, length)
	copy(b, x)
	copy(b[len(x):], y)
	return goString(&b[0], length)
}
```

Мы выделяем `[]byte`, который используем для формирования содержимого новой строки. Затем нам остаётся лишь финализировать строку приведением её к ожидаемому рантаймом представлению. За это отвечает функция `goString`:

```go
func goString(ptr *byte, length int) string {
	s := stringStruct{str: ptr, len: length}
	return *(*string)(unsafe.Pointer(&s))
}
```

> Оценка производительности: **91.9%** (+10.9).

# Версия для AMD64


К сожалению, предыдущая версия функции не имеет оптимизации для конкатенации с пустой строкой, а ещё мы выполняем некоторое количество лишних вычислений из-за невозможности выделить память напрямую, приходится работать со слайсом байт.

Одной из интересных особенностей Go ассемблера является то, что он позволяет вызывать, например, неэкспортируемые функции рантайма. Мы можем вызвать `runtime·mallocgc` из ассемблерного кода даже если он не является частью пакета `runtime`. Этим свойством мы и воспользуемся.

Также мы можем проверять принадлежность строк стековой памяти, что делает безопасной оптимизацию возврата одного из аргументов в качестве результата.

Допустим, функция вызвана с аргументами `concat2("", "123")`. `x` - пустая строка, и если `y` не выделен на стеке, мы можем вернуть его в качестве результата конкатенации.

```asm
//; Считаем, что x и y имеют тип stringStruct.
//; CX - y.str.
//; SI - y.len.
maybe_return_y:
        //; Проверка на вхождения указателя в стек.
        MOVQ (TLS), AX //; *g
        CMPQ CX, (AX)
        JL return_y //; если y_str < g.stack.lo
        CMPQ CX, 8(AX)
        JGE return_y //; если y_str >= g.stack.hi
        JMP concatenate //; y на стеке, нужна новая аллокация
return_y:
        MOVQ CX, ret+32(FP) //; stringStruct.len
        MOVQ SI, ret+40(FP) //; stringStruct.str
        RET
```

`MOVQ (TLS), AX` переместит [*g](https://github.com/golang/go/blob/a80a7f0e77fab42cebe61c43b98e0959b740def2/src/runtime/runtime2.go#L338) в регистр `AX`. Чтение по нулевому смещению даст поле `g.stack.lo`, а с 8-го байта начинается `g.stack.hi` (для 64-битной платформы).

```go
type g struct {
        stack struct {
                lo uintptr // 0(AX)
                hi uintptr // 8(AX)
        }
        stackguard0 uintptr // 16(AX)
        stackguard1 uintptr // 24(AX)
        // ... другие поля
}
```

Тело `concatenate` выделяет память, заполняет его обеими строками, и возвращает новую строку.

<spoiler title="Полный листинг с комментариями">

```asm
#include "textflag.h"
#include "funcdata.h"

TEXT ·Strings(SB), 0, $48-48
        NO_LOCAL_POINTERS // Костыль для избежания ошибки.
        MOVQ x+0(FP), DX
        MOVQ x+8(FP), DI
        MOVQ y+16(FP), CX
        MOVQ y+24(FP), SI
        TESTQ DI, DI
        JZ maybe_return_y // x - пустая строка, попробуем вернуть y
        TESTQ SI, SI
        JZ maybe_return_x // y - пустая строка, попробуем вернуть x
concatenate:
        LEAQ (DI)(SI*1), R8 // len(x) + len(y)
        // Выделяем память для новой строки.
        MOVQ R8, 0(SP)
        MOVQ $0, 8(SP)
        MOVB $0, 16(SP)
        CALL runtime·mallocgc(SB)
        MOVQ 24(SP), AX // Указатель на выделенную память
        MOVQ AX, newstr-8(SP)
        // Копируем x.
        MOVQ x+0(FP), DX
        MOVQ x+8(FP), DI
        MOVQ AX, 0(SP)
        MOVQ DX, 8(SP)
        MOVQ DI, 16(SP)
        CALL runtime·memmove(SB)
        // Копируем y со смещения len(x).
        MOVQ x+8(FP), DI
        MOVQ y+16(FP), CX
        MOVQ y+24(FP), SI
        MOVQ newstr-8(SP), AX
        LEAQ (AX)(DI*1), BX
        MOVQ BX, 0(SP)
        MOVQ CX, 8(SP)
        MOVQ SI, 16(SP)
        CALL runtime·memmove(SB)
        // Возврат новой строки.
        MOVQ newstr-8(SP), AX
        MOVQ x+8(FP), R8
        ADDQ y+24(FP), R8
        MOVQ AX, ret+32(FP)
        MOVQ R8, ret+40(FP)
        RET
maybe_return_y:
        // Проверка на вхождения указателя в стек.
        MOVQ (TLS), AX // *g
        CMPQ CX, (AX)
        JL return_y // если y_ptr < stk.lo
        CMPQ CX, 8(AX)
        JGE return_y // если y_ptr >= stk.hi
        JMP concatenate // y на стеке, нужна новая аллокация
return_y:
        MOVQ CX, ret+32(FP)
        MOVQ SI, ret+40(FP)
        RET
maybe_return_x:
        // Проверка на вхождения указателя в стек.
        MOVQ (TLS), AX // *g
        CMPQ DX, (AX)
        JL return_x // если x_ptr < stk.lo
        CMPQ DX, 8(AX)
        JGE return_x // если x_ptr >= stk.hi
        JMP concatenate // x на стеке, нужна новая аллокация
return_x:
        MOVQ DX, ret+32(FP)
        MOVQ DI, ret+40(FP)
        RET
```

Если вам интересна природа `NO_LOCAL_POINTERS` в этом коде, можете почитать [Calling a Go function from asm ("fatal error: missing stackmap")](https://groups.google.com/forum/#!topic/golang-nuts/a6NKBbL9fX0).

</spoiler>

> Оценка производительности: **100%** (+8.6).

# В качестве заключения

Весь код предоставлен в качестве пакета [concat](https://github.com/Quasilyte/concat).

Готов ли мир к такой быстрой конкатенации? Who knows.

В начале статьи был упомянут [CL123256](https://go-review.googlesource.com/c/go/+/123256). У него есть несколько путей развития:
1. Вариадичная специализация для случая, когда компилятором не выделен временный буфер. Меньше прирост на каждый случай, зато покрывает больше видов конкатенации и практически не увеличивает размер кода (как машинного, так и кода на Go).
2. Больше специализаций для частных случаев. Выше прирост, но больше машинного кода, может навредить кешу инструкций.
3. Тонны машинного кода, для каждого особого случая и специализированный memmove, на манер того как это сделано в glibc. Здесь в основном встают вопросы целесообразности.

Текущий предложенный вариант ускоряет только наиболее частый и простой случай конкатенации пары строк (арность=2).

Если в Go не примут это изменение, то сравнимого ускорения можно будет добиться с помощью реализации операций над строками в виде сторонней библиотеки. Менее удобно, красиво и элегантно, но зато работает.
