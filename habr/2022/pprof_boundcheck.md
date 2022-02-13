# Анализируем bound checks в Go по CPU профилю

Сегодня мы будем анализировать бинарники на пару с CPU профилями, чтобы создать на их основе расширенные профили исполнения. Эти дополненные профили мы сможем использовать для оценки времени, которое программа тратит на проверки выхода за границу массивов и слайсов.

![](https://habrastorage.org/webt/z9/my/pu/z9mypusorv6u--j-d6kicx5i3xk.png)

<cut/>

## Введение в проблему

Профилирование в Go собирает стеки вызовов вместе с позицией текущей исполняемой инструкции. И хотя такая гранулярность очень полезна при просмотре в disasm режиме, она мало чего говорит нам о том, какова семантика исполняемых инструкций.

Проще всего работать с уровнем функций. По функциям мы можем агрегировать семплы, строить деревья вызовов и флеймграфы.

Некоторые встроенные в язык операции генерируют какой-то понятный вызов. Например, операторы для работы с каналами (`runtime.chanrecv`, `runtime.chansend`).

Другие могут превращаться в несколько вызовов: `append(b1, b2...)` создаст вызовы `runtime.growslice` и `runtime.memmove`. Это уже усложняет анализ профилей, но у нас хотя бы есть, за что зацепиться.

Самое сложное - это операции, которые встраивают свой код прямо в место использования, без какого-либо вызова. Это эффективно в плане скорости исполнения, но профилирование таких конструкций штатными средствами становится невозможным. Машинный код этих операций смешивается с использующей их функцией, а в профиле эти значения будут увеличивать личное (flat/self) время этой функции.

## Цена bound check'ов

Будем бенчмаркать довольно простую функцию получения последнего элемента слайса:

```go
package bcbench

import "testing"

//go:noinline
func getlast(xs []int) int {
  return xs[len(xs)-1]
}

func BenchmarkGetLast(b *testing.B) {
  xs := []int{0, 2, 3, 5}
  for i := 0; i < b.N; i++ {
    getlast(xs)
    getlast(xs)
    getlast(xs)
    getlast(xs)
  }
}
```

Чтобы бинарник с тестом был доступен после бенчмарков, соберём его явно через `-c`. Нам потребуется много запусков (count), так как сами тестируемые операции очень быстрые - семплирующая природа профилирования требует более длинных интервалов, чтобы у нас был шанс снять стеки в нужные моменты.

```bash
$ go test -c
$ ./bcbench.test -test.count=60 -test.cpuprofile=cpu.out -test.bench=. .
```

Теперь откроем профиль в браузере:

```bash
$ go tool pprof -http=:8080
```

![](https://habrastorage.org/webt/_p/pf/a4/_ppfa4ixc0vmbzanjhrebk1wgac.png)

Я отметил строки, которые напрямую относятся к проверкам выхода за пределы слайса. Мы ещё вернёмся к этой структуре, но сначала я хочу обратить ваше внимание на то, что большая часть семплов вообще не покрыла связанные инструкции. Это не значит, что bound check в данном случае бесплатен. Это скорее признак того, что записывая семплы 100 раз в секунду нам очень сложно поймать некоторые участки машинного кода.

Перепишем функцию `getlast` так, чтобы там не было проверок, вставленных компилятором:

```go
//go:noinline
func getlast(xs []int) int {
  if len(xs) != 0 {
    return xs[len(xs)-1]
  }
  return 0
}
```

Запустим аналогичные бенчмарки, соберём профиль, сравним быстродействие.

![](https://habrastorage.org/webt/cl/2c/qg/cl2cqgnlqi9sdalfa6xhvg2vei8.png)

Можем видеть, что вставленных компилятором проверок нет. Вместо CMPQ у нас в коде TESTQ и, может показаться, что это работает медленнее, ведь целых 4 секунды мы провели на этой инструкции. Но это не так, нам просто "повезло".

При сравнении быстродействия видно, что вторая версия функции быстрее, хоть её исходники и более многословные:

```bash
$ benchstat old.txt new.txt 
name       old time/op  new time/op  delta
GetLast-8  12.5ns ± 2%   9.3ns ± 1%  -25.93%  (p=0.000 n=10+10)
```

И, снова, это скорее побочный эффект: версия без вызова `panicindex` не имеет фрейма, поэтому сама по себе функция более легковесная. Наша проблема в том, что даже имея всю информацию, замерять время, потраченное на bound check - не очень просто.

> Сам бенчмарк можно переписать, убрав `go:noinline`, но тогда нужно сделать так, чтобы компилятор не догадался, что наш слайс - неизменяемый и проверять его длину внутри цикла нет смысла. Писать корректные бенчмарки - [это нетривиально](https://github.com/golang/go/issues/27400).

## Откуда появляется bound check

Проверки выхода за пределы массива бывают разные. Давайте попробуем узнать, что вообще в нашем коде может требовать этих проверок.

Для массивов, частью типа которых является их длина, проверки вставляются только в случае неконстантных индексов.

```go
func f(i int) {
  var a [8]int
  println(a[0]) // не требует проверок
  println(a[i]) // проверка i < len(a)
}
```

Для слайсов эти проверки вставляются почти по любому поводу. На практике, часть из этих проверок удаляется как избыточные, когда компилятор может доказать, что индекс точно лежит в допустимом интервале.

При слайсинге (взятии "срезов") массивов происходят те же самые проверки, поэтому для `a[i:]` компилятор будет вставлять проверку на `i <= len(a)`.

```go
func f(b []byte, i, j int) {
  println(b[i])   // проверка i < len(b)
  println(b[i:])  // проверка i <= len(b)
  println(b[:j])  // проверка j <= cap(b)
  println(b[i:j]) // проверка i <= len(b) && j <= cap(b)
}
```

Здесь важно отметить, что разные операции могут генерировать немного отличающиеся фрагменты. Результаты так же зависят от версии Go компилятора. Данная статья написана опираясь на go `1.17` и `1.18`.

## Смотрим на дизассемблер bound check'ов

Все примеры кода актуальны для x86-64 (GOARCH=amd64).

Начнём с самого простого случая:

```go
func indexing(i int, xs []int) int {
  return xs[i]
}
```

Посмотрим на сгенерированный код:

![](https://habrastorage.org/webt/zz/-v/7m/zz-v7mpikwbjbyu-ykvilsjhx10.png)

<spoiler title="То же самое, только текстом">

```asm
    SUBQ $24, SP
    MOVQ BP, 16(SP)
    LEAQ 16(SP), BP
    MOVQ BX, xs+40(FP)
    CMPQ CX, AX
    JLS bc_fail
    MOVQ (BX)(AX*8), AX
    MOVQ 16(SP), BP
    ADDQ $24, SP
    RET
bc_fail:
    CALL runtime.panicIndex(SB)
```

</spoiler>

Это и есть довольно общая схема для bound check'а:

1. Инструкция сравнения (CMPQ или TEST в случае сравнения с 0)
2. Условный прыжок (JLS выше - это JBE)
3. В месте прыжка идёт вызов одной из паникующих функций

До инструкции CALL могут быть несколько инструкций типа MOV или LEA для передачи нужных аргументов.

Возьмём ещё несколько функций для экспериментов:

```go
func slicingFrom(i int, xs []int) []int {
  return xs[i:]
}

func slicingTo(i int, xs []int) []int {
  return xs[:i]
}

func slicingFromTo(i, j int, xs []int) []int {
  return xs[i:j]
}
```

Если посмотреть через дизассемблер на получившийся код, мы заметим, что для этих случаев генерируется во многом похожий код, но функции "паники" отличаются.

* `indexing` - `runtime.panicIndex()`
* `slicingFrom` - `runtime.panicSliceB()`
* `slicingTo` - `runtime.panicSliceAcap()`
* `slicintFromTo` - `runtime.panicSliceB()` и `runtime.panicSliceAcap()`

На самом деле, даже это не полный набор. Весь список функций можно подсмотреть в [исходниках компилятора](https://github.com/golang/go/blob/bcee121ae4f67281450280c72399890a3c7a7d5b/src/cmd/compile/internal/ssagen/ssa.go#L170-L186).

Это означает, что есть более одной функции, которая может вызываться в случае неуспешной проверки. Нам нужно знать адреса всех этих функций.

Если же машинный код матчится по алгоритму выше и завершается вызовом специальной функции-паники, типа `panicSliceAcap`, то мы можем быть уверенными, что перед нами bound check.

При этом считать семплы мы будем только для двух инструкций из всей распознанной последовательности: CMPQ и JLS.

## Собираем адреса функций-паник

Я буду рассматривать пример с ELF бинарником.

В стандартной библиотеке Go есть пакет [debug/elf](https://pkg.go.dev/debug/elf). Им мы и воспользуемся.

```go
var bcFuncNames = map[string]struct{}{
  "runtime.panicIndex":        {},
  "runtime.panicIndexU":       {},
  "runtime.panicSliceAlen":    {},
  "runtime.panicSliceAlenU":   {},
  "runtime.panicSliceAcap":    {},
  "runtime.panicSliceAcapU":   {},
  "runtime.panicSliceB":       {},
  "runtime.panicSliceBU":      {},
  "runtime.panicSlice3Alen":   {},
  "runtime.panicSlice3AlenU":  {},
  "runtime.panicSlice3Acap":   {},
  "runtime.panicSlice3AcapU":  {},
  "runtime.panicSlice3B":      {},
  "runtime.panicSlice3BU":     {},
  "runtime.panicSlice3C":      {},
  "runtime.panicSlice3CU":     {},
  "runtime.panicSliceConvert": {},
}

bcFuncAddresses := make(map[int64]struct{})

// exeBytes - []byte из нашего бинарника
f, err := elf.NewFile(bytes.NewReader(exeBytes))
if err != nil {
  return fmt.Errorf("parse ELF: %v", err)
}
symbols, err := f.Symbols()
if err != nil {
  return fmt.Errorf("fetch ELF symbols: %v", err)
}

for _, sym := range symbols {
  if _, ok := bcFuncNames[sym.Name]; !ok {
    continue
  }
  addr := sym.Value // !!! <----------------------------
  bcFuncAddresses[int64(addr)] = struct{}{}
}
```

К сожалению, этот код не совсем корректен. Обратите внимание на строку с отметкой.

Чтобы адреса успешно отображались на то, что будет внутри CPU профиля, нам нужно выполнить некоторые преобразования. Пришло время парсить CPU профиль.

Пакет [github.com/google/pprof/profile](https://github.com/google/pprof) идеально подходит для нашей задачи.

```go
// Использую ReadFile, а не os.Open, чтобы не возиться с файлом,
// который потом нужно будет закрывать
data, err := os.ReadFile("cpu.out")
if err != nil {
  return fmt.Errorf("read profile: %v", err)
}
p, err := profile.Parse(bytes.NewReader(data))
if err != nil {
  return fmt.Errorf("parse profile: %v", err)
}
```

Теперь поменяем вычисления адреса:

```diff
- addr := sym.Value
+ addr := sym.Value + p.Mapping[0].Offset - p.Mapping[0].Start
```

Ура! Теперь мы можем определять `CALL <panicfunc>` по таблице адресов.

## Разбираем машинный код

Обходя семплы, мы используем `loc.Address`, чтобы получить адрес текущей исполняемой инструкции. Затем мы проверяем, можно ли с этой позиции распознать bound check в функции `markBoundCheck`.

```go
for _, sample := range p.Sample {
  if len(sample.Location) == 0 {
    continue
  }
  for _, loc := range sample.Location {
    if len(loc.Line) == 0 {
      continue
    }
    m := loc.Mapping
    addr := int64(loc.Address + m.Offset - m.Start)
    // ctx содержит подготовленный ранее bcFuncAddresses
    markBoundCheck(ctx, exeBytes, addr)
  }
}
```

Картинка ниже призвана помочь понять высокоуровневую структуру профилей. Семплы располагаются примерно так:

![](https://habrastorage.org/webt/qz/ur/ty/qzurtyjymrozdfsj24azcasz-hk.png)

Остаётся реализовать `markBoundCheck`. Для декодирования инструкций будем использовать [golang.org/x/arch/x86/x86asm](https://pkg.go.dev/golang.org/x/arch/x86/x86asm).

```go
func markBoundCheck(ctx *context, code []byte, addr int64) bool {
  // 1. Инструкция сравнения
  cmp, err := x86asm.Decode(code[addr:], 64)
  if err != nil {
    return false
  }
  if cmp.Op != x86asm.CMP && cmp.Op != x86asm.TEST {
    return false
  }

  // 2. Условный прыжок
  jmp, err := x86asm.Decode(code[addr+int64(cmp.Len):], 64)
  if err != nil {
    return false
  }
  jumpFrom := int64(addr) + int64(cmp.Len)
  var jumpTo int64
  switch jmp.Op {
  case x86asm.JBE, x86asm.JB, x86asm.JA:
    rel, ok := jmp.Args[0].(x86asm.Rel)
    if !ok {
      return false
    }
    jumpTo = jumpFrom + int64(rel) + int64(jmp.Len)
  default:
    return false
  }

  // 3. Вызов паникующей функции
  call, err := x86asm.Decode(code[jumpTo:], 64)
  if err != nil {
    return false
  }
  if inst.Op != x86asm.CALL {
    return false
  }
  funcRel, ok := call.Args[0].(x86asm.Rel)
  if !ok {
    return false
  }
  funcAddr := jumpTo + int64(funcRel) + int64(call.Len)
  if _, ok := ctx.bcFuncAddresses[funcAddr]; !ok {
    return false
  }

  // Записываем, что CMPQ (addr) и прыжок (addr+cmp.Len) относятся
  // к bound check операциям
  ctx.bcAddresses[addr] = struct{}{}
  ctx.bcAddresses[addr+int64(cmp.Len)] = struct{}{}

  return true
}
```

Этот код немного упрощён. Ссылку на полную реализацию я привожу в конце статьи.

Теперь можем по любому адресу инструкции из CPU профиля сказать, относится ли она к bound check или нет.

## Добавляем runtime.boundcheck в профиль

Самый простой способ - это добавить новые inline фреймы в уже существующие `profile.Location` объекты.

Но сначала нам нужно добавить в профиль информацию о новой функции. Назовём её `runtime.boundcheck`:

```go
// ID начинаются с 1, то есть p.Function[i].ID == i+1
id := len(p.Function) + 1
boundcheckFunc := &profile.Function{
  ID:         uint64(id),
  Name:       "runtime.boundcheck",
  SystemName: "runtime.boundcheck",
  Filename:   "builtins.go", // Не имеет значения
  StartLine:  1,             // Не имеет значения
}
p.Function = append(p.Function, boundcheckFunc)
```

Теперь пройдёмся по всем семплам ещё раз и будем вставлять `runtime.boundcheck` для всех семплов, которые сейчас исполняют связанный с этим код.

```go
for _, sample := range p.Sample {
  if len(sample.Location) == 0 {
    continue
  }
  loc := sample.Location[0]
  if len(loc.Line) == 0 {
    continue
  }
  m := loc.Mapping
  addr := int64(loc.Address + m.Offset - m.Start)
  if _, ok := ctx.bcAddresses[addr]; ok {
    insertLine(loc, boundcheckFunc)
  }
}
```

Функцию `insertLine` можно реализовать так:

```go
func insertLine(loc *profile.Location, fn *profile.Function) {
  if loc.Line[0].Function == fn {
    return // Видимо, мы уже вставляли сюда новый вызов
  }
  newLine := profile.Line{
    Function: fn,
    // Номер строки не важен, ведь функция runtime.boundcheck
    // по-настоящему нигде не определена и не реализована
    Line: fn.StartLine, 
  }
  loc.Line = append([]profile.Line{newLine}, loc.Line...)
}
```

Вставляя вызов `runtime.boundcheck` в `profile.Location.Line[0]`, смещая все остальные вызовы, мы расширяем стек inlined функций. Если бы мы добавляли новый `profile.Location`, то это бы означало non-inlined вызов.

```
Line[0] getlast => Line[0] runtime.boundcheck
                   Line[1] getlast
```

После работы с объектов профиля нам остаётся вызвать метод `Write` и сохранить его в новый файл:

```go
f, err := os.Create("cpu2.out")
if err != nil {
  return err
}
defer f.Close()
if err := p.Write(f); err != nil {
  return err
}
```

## Работаем с новым, дополненным профилем

Попробуем открыть новый профиль в pprof:

```bash
$ go tool pprof cpu2.out
```

![](https://habrastorage.org/webt/u9/0d/ev/u90devshjg_ntfiqhapux9mxsoy.jpeg)

Всё работает. GUI режим тоже работает, вы уже могли видеть скриншот выше (КДПВ).

На скриншоте можно так же видеть `runtime.nilcheck`. Добавить эту информацию в профиль не сложно, а алгоритм будет абсолютно идентичен. Отличается только mark фаза, где мы пытаемся матчить машинный код. Чаще всего nil check в Go выглядит так: `TESTB AX, (reg)`.

## Заключение

Добавление в профиль псевдо-функций `runtime.nilcheck` и `runtime.boundcheck` реализовано в [qpprof](https://github.com/quasilyte/qpprof). Полные версии исходников из примеров можно найти там же.

На данном этапе я не могу сказать, что мы получаем релевантные значения, но, по крайней мере, мы научились дополнять профили чем-то новым. Если использовать альтернативный профилировщик для Go, который имеет более высокий [hz для профилирования](https://github.com/golang/go/blob/master/src/runtime/pprof/pprof.go#L772), либо вообще является не семплирующим, то наши шансы получить более точные метрики возрастают. Если эти другие профилировщики умеют экспортировать данные в формате pprof ([profile.proto](https://github.com/google/pprof/blob/513e8ac6eea103037e9be150bd17ceccacbe7bf6/proto/profile.proto)), то все приёмчики из этой статьи будут актуальны. В противном случае мы можем попробовать конвертировать профиль в родной для Go формат.

У меня есть ещё одно секретное применения для таких расширенных профилей, но об этом мы поговорим в другой раз.

