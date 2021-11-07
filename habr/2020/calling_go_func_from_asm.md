# Что нужно знать, если вы хотите вызывать Go функции из ассемблера

> You've run into a really hairy area of asm code.
> My first suggestion is not try to call from assembler into Go. -- [Ian Lance Taylor](https://groups.google.com/d/msg/golang-nuts/a6NKBbL9fX0/SuMDpME-AgAJ)

До тех пор, пока ваш ассемблерный код делает что-то простое, всё выглядит неплохо.

Как только у вас возникает задача вызвать из ассемблерного кода Go функцию, один из первых советов, который вам дадут: не делайте так.

Но что если вам это очень-очень нужно? В таком случае, прошу под кат.

![](https://habrastorage.org/webt/ep/4u/el/ep4uelswcffwgq9yfqkyuvcy5bu.png)

<cut/>

## Calling convention

Всё начинается с того, что нужно понять, как [передавать функции аргументы и как принимать её результаты](https://en.wikipedia.org/wiki/Calling_convention).

Рекомендую ознакомиться с [Go  functions  in  assembly  language](https://github.com/golang/go/files/447163/GoFunctionsInAssembly.pdf), где наглядно описана большая часть необходимой нам информации.

Обычно, calling convention варьируется от платформы к платформе, поскольку может различаться набор доступных регистров. Мы будем рассматривать только `GOARCH=amd64`, но в случае Go конвенции отличаются не так значительно.

Вот некоторые особенности конвенции вызова функций в Go:
* Все аргументы передаются через стек, кроме "контекста" в замыканиях, он доступен через регистр `DX` (%rdx).
* Результаты возвращаются тоже через стек (в картинке ниже они включены в arguments).
* Аргументы вызываемой функции размещаются на фрейме вызывающей стороны.
* Выделение и уничтожение фрейма выполняется вызываемой функцией. Эти действия являются частью прологов и эпилогов, которые вставляются ассемблером автоматически.

При вызове функции может происходить ситуация, когда у стека горутины недостаточно места. В этой ситуации будет произведено расширение стека.

![](https://habrastorage.org/webt/kb/jk/hy/kbjkhyjugizyb1weh2haraas7qy.png)

Эта картина мира может поменяться, если произойдёт переход на [register-based calling convention](https://github.com/golang/go/issues/18597). Мы ещё вернёмся к теме эволюции конвенций вызова функций в Go.

Итак, чтобы вызвать функцию:
* Вам нужно, чтобы на фрейме текущей функции было место для аргументов вызываемой функции.
* Первый аргумент идёт в `0(SP)`, второй аргумент идёт в `8(SP)` (если размер первого аргумента равен 8 байтам), и так далее.
* Результат функции доставать из `n(SP)`, где `n` - это размер всех входных аргументов функции. Для функции с двумя аргументами `int64`, результат начинается с `16(SP)`.

Размер фрейма вы указываете при определении функции.

```go
package main

func asmfunc(x int32) (int32, int32)

func gofunc(a1 int64, a2, a3 int32) (int32, int32) {
	return int32(a1) + a2, int32(a1) + a3
}

func main() {
	v1, v2 := asmfunc(10)
	println(v1, v2) // => 3, 11
}
```

```asm
// func asmfunc(x int32) (int32, int32)
TEXT ·asmfunc(SB), 0, $24-12
  MOVL x+0(FP), AX
  MOVQ $1, 0(SP)  // Первый аргумент (a1 int64)
  MOVL $2, 8(SP)  // Второй аргумент (a2 int32)
  MOVL AX, 12(SP) // Третий аргумент (a3 int32)
  CALL ·gofunc(SB)
  MOVL 16(SP), AX // Забираем первый результат
  MOVL 20(SP), CX // Забираем второй результат
  MOVL AX, ret+8(FP)  // Возвращаем первый результат
  MOVL CX, ret+12(FP) // Возвращаем второй результат
  RET
```

```
$24-16 (locals=24 bytes, args=16 bytes)

          0     8     12    16     20     SP
locals=24 [a1:8][a2:4][a3:4][ret:4][ret:4]
(ret относится к фрейму asmfunc, хранит результаты gofunc)

        0    4          8      12     FP
args=16 [x:4][padding:4][ret:4][ret:4]
(ret относится к фрейму main, хранит результаты asmfunc)
```

Обратите внимание, между входными аргументами и результатами есть 4 байта для выравнивания. Это нужно для того, чтобы результаты функции начинались с адреса, который выравнен по ширине указателя (8 байт на `amd64`).

Каждый аргумент тоже выравнивается по тем же правилам, что и поля структур. Если у функции первый параметр - `int32`, а второй - `int64`, то offset второго параметра будет 8, а не 4, поскольку `reflect.TypeOf(int64(0)).Align()` равен 8.

Некоторые ошибки, связанные с размером фрейма и использованием регистра `FP` может найти `go vet`.

## Указатели и stackmap

Попробуем теперь вызвать функцию с аргументом-указателем.

```go
package foo

import (
	"fmt"
	"testing"
)

func foo(ptr *object)

type object struct {
	x, y, z int64
}

func printPtr(ptr *object) {
	fmt.Println(*ptr)
}

func TestFoo(t *testing.T) {
	foo(&object{x: 11, y: 22, z: 33})
}
```

```asm
TEXT ·foo(SB), 0, $8-8
        MOVQ ptr+0(FP), AX
        MOVQ AX, 0(SP)
        CALL ·printPtr(SB)
        RET
```

Если мы запустим тест, то получим [панику из-за stackmap](https://groups.google.com/d/msg/golang-nuts/a6NKBbL9fX0/MzUGGqQ9AgAJ):
```
=== RUN   TestFoo
runtime: frame <censored> untyped locals 0xc00008ff38+0x8
fatal error: missing stackmap
```

Для того, чтобы успешно найти указатели на стеке, GC нуждается в так называемых stackmaps. Для обычных Go функций их генерирует компилятор, но для ассемблерных функций этой информации у рантайма нет.

Для аргументов функции информацию можно передать через определение stub функции с корректными типами (функция без тела в Go файле). В [документации](https://golang.org/doc/asm) указаны случаи, когда stackmap не обязателен, но наш случай не один из них.

Здесь либо делать так, чтобы функция не требовала stackmap (сложно), либо использовать `NO_LOCAL_POINTERS` и не подорваться на нюансах (ещё сложнее).

## Макрос NO_LOCAL_POINTERS

С таким ассемблерным кодом `TestFoo` будет проходить:

```asm
#include "funcdata.h"

TEXT ·foo(SB), 0, $8-8
        NO_LOCAL_POINTERS
        MOVQ ptr+0(FP), AX
        MOVQ AX, 0(SP)
        CALL ·printPtr(SB)
        RET
```

Однако нужно понимать, чем достигнут этот прогресс.

Попробуем поразмышлять, зачем вообще сборщику мусора нужно знать про указатели на нашем стеке? Допустим, эти указатели пришли к нам извне, они "достижимы" из кода, который вызывал ассемблерную функцию, поэтому нам не страшно, если локальные указатели на нашем фрейме не будут считаться живыми, так?

Если мы вспомним, что указатели могут указывать не только на объекты в heap, то поймём, что это не всегда так. Если при вызове функции произойдёт увеличение стека, адреса стека изменятся, что инвалидирует все указатели объекты внутри него. В обычном режиме GC "чинит" все эти указатели и мы ничего не замечаем, но если у него нет информации об указателях на стеке, он этого сделать не сможет.

Здесь нам помогает то, что все указатели, передаваемые в ассемблерную функцию "утекают" (escapes to heap) в терминах escape analysis, так что для того, чтобы иметь внутри ассемблерного кода на стеке указатель на стековую память нужно постараться.

Сформулируем правило безопасного использования `NO_LOCAL_POINTERS`: данные, адресуемые указателями, лежащими внутри локальных слотов функции, должны удерживаться видимыми GC указателями. Запрещены указатели на стек.

В связи с появлением в Go [non-cooperative preemption](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md), важным дополнением будет то, что [ассемблерные функции не прерываются](https://go-review.googlesource.com/c/go/+/202337/).

Второй кейс безопасного использования можно найти [внутри рантайма Go](https://github.com/golang/go/blob/d67d044310bc5cc1c26b60caf23a58602e9a1946/src/runtime/vlop_arm.s#L147-L158). Функции, отмеченные `go:nosplit`, не могут расширить стек, так что часть проблем, связанная с `NO_LOCAL_POINTERS` уходит сама по себе.

## Макрос GO_ARGS

Для ассемблерных функций, которые имеют Go prototype, автоматически вставляется `GO_ARGS`.

`GO_ARGS` - это макрос из того же [funcdata.h](https://golang.org/src/runtime/funcdata.h), что и `NO_LOCAL_POINTERS`. Он указывает, что для получения информации о stackmap аргументов нужно использовать Go декларацию.

Раньше это не работало в ситуации, когда [stackmap для ассемблерной функции определялся в другом пакете](https://github.com/golang/go/issues/24419). Сейчас проставлять `args_stackmap` вручную для экспортируемых символов не обязательно. Но как пример этот патч всё равно интересен: он показывает, как можно ручками добавить метаданных в stackmap.

## Макрос GO_RESULTS_INITIALIZED

Если ассемблерная функция возвращает указатель и вызывает Go функции, то требуется начать тело этой функции с зануления стековых слотов под результат (так как там может находиться мусор) и вызвать макрос `GO_RESULTS_INITIALIZED` сразу после этого.

Например:

```asm
// func getg() interface{}
TEXT ·getg(SB), NOSPLIT, $32-16
  // Интерфейс состоит из двух указателей.
  // Оба из них нужно заполнить нулями.
  MOVQ $0, ret_type+0(FP)
  MOVQ $0, ret_data+8(FP)
  GO_RESULTS_INITIALIZED
  // Дальше код самой функции...
  RET
```

В целом, лучше избегать ассемблерных функций, которые возвращают типы-указатели.

Больше примеров использования можно найти на [GitHub](https://github.com/search?l=Unix+Assembly&q=GO_RESULTS_INITIALIZED&type=Code).

## Вызов Go функций из JIT-кода

Отдельное внимание уделим вызовам Go функций из динамически сгенерированного машинного кода.

Garbage collector Go ожидает, что весь код, вызывающий функции, доступен во время компиляции, поэтому можно сказать, что [Go не очень дружит с JIT'ом](https://github.com/golang/go/issues/20123).

Для начала, воспроизведём фатальную ошибку. Листинг спрятан под спойлером, чтобы не загромождать тело статьи для тех, кто захочет пропустить этот раздел.

<spoiler title="calljit-v1"><hr>

```go
// file jit.go
package main

import (
	"log"
	"reflect"
	"syscall"
	"unsafe"
)

func main() {
	a := funcAddr(goFunc)

	code := []byte{
		// MOVQ addr(goFunc), AX
		0xb8, byte(a), byte(a >> 8), byte(a >> 16), byte(a >> 24),
		// CALL AX
		0xff, 0xd0,
		// RET
		0xc3,
	}

	executable, err := mmapExecutable(len(code))
	if err != nil {
		log.Panicf("mmap: %v", err)
	}
	copy(executable, code)
	calljit(&executable[0])
}

func calljit(code *byte)

func goFunc() {
	println("called from JIT")
}


func mmapExecutable(length int) ([]byte, error) {
	const prot = syscall.PROT_READ | syscall.PROT_WRITE | syscall.PROT_EXEC
	const flags = syscall.MAP_PRIVATE | syscall.MAP_ANON
	return mmapLinux(0, uintptr(length), prot, flags, 0, 0)
}

func mmapLinux(addr, length, prot, flags, fd, off uintptr) ([]byte, error) {
	ptr, _, err := syscall.Syscall6(
		syscall.SYS_MMAP,
		addr, length, prot, flags, fd, offset)
	if err != 0 {
		return nil, err
	}
	slice := *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: ptr,
		Len:  int(length),
		Cap:  int(length),
	}))
	return slice, nil
}

func funcAddr(fn interface{}) uintptr {
	type emptyInterface struct {
		typ   uintptr
		value *uintptr
	}
	e := (*emptyInterface)(unsafe.Pointer(&fn))
	return *e.value
}
```

```asm
// file jit_amd64.s
TEXT ·calljit(SB), 0, $0-8
        MOVQ code+0(FP), AX
        JMP AX
```
<hr></spoiler>

Если мы соберём и запустим этот код, то всё будет выглядеть хорошо (хотя вам может не повезти):

```bash
$ go build -o jit . && ./jit
called from JIT
```

А теперь внесём небольшие изменения в функцию `goFunc`, которая вызывается из JIT-кода:

```diff
 func goFunc() {
 	println("called from JIT")
+ 	runtime.GC()
 }
```

Повторный запуск уже более надёжно падает с паникой:

```bash
$ go build -o jit . && ./jit
called from JIT
runtime: unexpected return pc for main.goFunc called from 0x7f9465f7c007
stack: frame={sp:0xc00008ced0, fp:0xc00008cef0} stack=[0xc00008c000,0xc00008d000)
000000c00008cdd0:  0000000000000000  00007f94681f7558 
000000c00008cde0:  000000c000029270  000000000000000b 
... (+ more)
```

Читаем: `unexpected return pc for main.goFunc called from 0x7f9465f7c007`, где `0x7f9465f7c007` - это адрес внутри нашего JIT-кода. Паника сообщает, что подобный вызов очень расстраивает runtime.

Гипотетически, эту проблему можно поправить, поправив значение `FP` и релевантных за хранение `BP` и return address локаций, но есть и другой способ.

Если Go runtime не хочет, чтобы мы вызывали функции из JIT-кода, мы будем вызывать их из других, разрешённых мест.

Перепишем `calljit` так, чтобы во второй его половине выполнялся код, ответственный за вызов Go функций. Каждый раз, когда нам нужно будет вызвать Go функцию, мы будем прыгать в этот участок кода, который находится в рамках известной Go функции (`calljit`).

```asm
#include "funcdata.h"

TEXT ·calljit(SB), 0, $8-8
        NO_LOCAL_POINTERS
        MOVQ code+0(FP), AX
        JMP AX
callgo:
        CALL CX
        JMP (SP)

```

Изменения:
* Нам понадобится хотя бы 8 байт фрейма, чтобы записать туда return address для JIT кода.
* `NO_LOCAL_POINTERS` нужен из-за того, что мы имеем ненулевой фрейм и делаем `CALL`.

Обычный путь `calljit` не изменился, происходит безусловный прыжок на машинный код, но добавилась вторая точка входа, специально для вызова Go функций. Мы ожидаем, что вызывающая сторона положит в регистр `CX` адрес вызываемой функции, а в `[rsp]` - адрес, куда надо вернуться после вызова.

Единственная сложность этого подхода в том, что нам нужно знать точный адрес метки `callgo`. Я подсмотрел через дизассемблер смещение этой метки, а потом захардкодил его в коде. Вот наш новый `main()`:

```go
a := funcAddr(goFunc)
j := funcAddr(calljit) + 36

code := []byte{
	// MOVQ funcAddr(goFunc), CX
	0x48, 0xc7, 0xc1, byte(a), byte(a >> 8), byte(a >> 16), byte(a >> 24),
	// MOVQ funcAddr(calljit), DI
	0x48, 0xc7, 0xc7, byte(j), byte(j >> 8), byte(j >> 16), byte(j >> 24),
	// LEAQ 6(PC), SI
	0x48, 0x8d, 0x35, (4 + 2), 0, 0, 0,
	// MOVQ SI, (SP)
	0x48, 0x89, 0x34, 0x24,
	// JMP DI
	0xff, 0xe7,

	// ADDQ $framesize, SP
	0x48, 0x83, 0xc4, (8 + 8),
	// RET
	0xc3,
}
```

Адрес вызываемой функции кладётся в `CX`, адрес `callgo` в `DI`, а в `SI` мы перемещаем позицию для возврата, затем записываем её в `[rsp]`. `4+2` - это сложение длин следующих за `LEAQ` двух инструкций.

Поскольку функция теперь имеет фрейм, нам нужно его чистить перед возвратом, поэтому выполняется `ADDQ`. 8 байтов под наш локальный слот плюс ещё 8 байтов на хранение регистра `BP`.

Вот так выглядит иллюстрация фрейма `calljit`:

```
здесь мы храним return address для callgo
|
|       Go сохраняет значение предыдущего BP по этому адресу
|       |
0(SP)   8(SP)    16(SP)    24(SP)
[empty] [prevBP] [retaddr] [arg1:code]
|             /  |         |
|            /   |         аргумент функции calljit (caller frame)
|           /    |      
|          /     пушится инструкцией CALL при вызове calljit
|         /
calljit frame, 16 bytes		
```

Теперь нам не страшен `runtime.GC()`:

```bash
$ go build -o jit . && ./jit
called from JIT
```

Это решение можно адаптировать для вызова Go функций с ненулевым количеством аргументов. Всё, что вам потребуется - это побольше места на фрейме, чтобы разместить там ещё и аргументы вызываемой функции вместе с её результатом.

Предложенное решение не обязательно самое оптимальное, но оно демонстрирует, что преодолеть описанную проблему возможно. По крайней мере в текущих версиях Go.

## Go Internal ABI

[Go Internal ABI](https://github.com/golang/proposal/blob/master/design/27539-internal-abi.md) - горячая тема в очень узких кругах.

Команда Go хочет иметь возможность менять такие детали, как конвенции вызова и правила взаимодействия с рантаймом, но эти изменения ломают существующий ассемблерный код. Предлагается ввести множественные ABI, часть из которых может использоваться публично, а как минимум одна будет приватной для компилятора и она же будет изменяться со временем.

Два ключевых ограничения:
1. Существующий ассемблерный код будет продолжать работать.
2. Эта поддержка обратной совместимости не будет прекращена в будущем.

Предыдущий calling convention теперь относится к `ABI0`, а экспериментальный новый к `ABIInternal`.

Если мы запустим компиляцию Go с флагом `-S`, то увидим, что `ABIInternal` уже существует, просто он не отличается на данный момент от `ABI0`:

![](https://habrastorage.org/webt/yf/cz/ox/yfczox8dzfemakdghfoit5m14iu.png)

Когда `ABIInternal` будет достаточно хорош, его переименуют в `ABI1`, сделав стабильным. `ABIInternal` же продолжит свой путь к идеальному calling convention и другим низкоуровневым радостям.

Хорошей новостью для нас является то, что в обозримом будущем существующий ассемблерный код продолжит работать корректно.

На этой оптимистической ноте, я хотел бы закончить эту небольшую заметку о вызове Go функций из ассемблерного кода. Если у вас есть дополнения, буду рад расширить материал.

## Полезные материалы

![](https://habrastorage.org/webt/q2/is/vu/q2isvuiycngnmodmqvefkv7bmfw.png)

* [Go functions in assembly language](https://github.com/golang/go/files/447163/GoFunctionsInAssembly.pdf)
* [Go internal ABI](https://github.com/golang/proposal/blob/master/design/27539-internal-abi.md)
* [Stack frame layout on x86-64](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)
* [NO_LOCAL_POINTERS и адреса на стек](https://groups.google.com/d/msg/golang-nuts/SxWxUG0uezY/YWXLSuesBQAJ)
* [Go assembly language complementary reference](https://quasilyte.dev/blog/post/go-asm-complementary-reference/)
* [ELFийские трюки в Go](https://habr.com/ru/post/482392/)
* [Библиотека amd64, которая умеет генерировать машинный код](https://github.com/modern-go/amd64)

## Hub-опрос

Мне всегда любопытно, не промахнулся ли я с хабами для публикации. Если вам не сложно, ознакомьтесь, пожалуйста, с опросом. Это может помочь найти потенциальные ошибки в подборе целевой аудитории для текущей статьи.
