# Go 1.11: AVX-512 со вкусом Go

<img src="https://habrastorage.org/webt/cd/io/8q/cdio8qky0mrwhnuvsvrx_vkm0t4.png" />

<br>

В Go 1.11 значительно обновлён ассемблер под платформу x86.

У программистов появится возможность использовать [AVX-512](https://en.wikipedia.org/wiki/AVX-512) - новейшие инструкции, доступные в процессорах Intel.

Под катом:

- Самые значительные обновления в `cmd/asm` (`go tool asm`)
- Как был внедрён новый набор инструкций в Go ассемблер
- Использование новых инструкций и специальных возможностей EVEX префикса
- Уровень интеграции в тулчейн (рецепты обхождения текущих ограничений)

<cut/>

# Что нового?

Из видимого для программиста:

- AVX-512

-- Новые векторные регистры: `X16-X31`, `Y16-Y31` и `Z0-Z31`
-- Добавлены регистры масок: `K0-K7`
-- Особые возможности EVEX префикса (см. ниже: rounding, zeroing, ...).
-- Сотни новых инструкций (379 новых опкодов + AVX{1,2} инструкции с EVEX префиксом).
- Добавлено 110 недостающих legacy инструкций ([CL97235](https://golang.org/cl/97235)).
- Почти на 25% более быстрое ассемблирование ([CL108895](https://golang.org/cl/108895)). Ускоряет сборку примерно на 1.5%.

Была также проведена предварительная работа по улучшению сообщений об ошибках ([CL108515](https://golang.org/cl/108515)), но в релиз go1.11 это не попадёт.

Кроме самого факта добавления новых расширений, важно то, что в новом ассемблере все VEX и EVEX таблицы сгенерированы автоматически.

Теперь в Go такой x86 ассемблер, в который не нужно добавлять новые инструкции вручную.

# Encoder в Go ассемблере

Часть ассемблера, ответственная за генерацию машинного кода, находится в стандартном пакете [cmd/internal/obj/x86](https://github.com/golang/go/tree/master/src/cmd/internal/obj/x86).

Большая часть кода в нём - это транслированные из C исходники [x86 ассемблера из plan9](https://bitbucket.org/inferno-os/inferno-os/src/default/utils/6l).

Таблицы ассемблера концептуально состоят из 3-х измерений: X, Y и Z.
Конкретная инструкция генерируется как `encode(X, Y, Z)`.
Альтернативной ментальной моделью может являться `table[X][Y][Z]`, но она менее близка к деталям реализации.

Из пространства опкодов (измерение X) выбирается соответствующий ассемблируемой инструкции объект `optab`. Затем перебирается список доступных комбинаций операндов (измерение Y) и выбирается соответствующий аргументам инструкции объект `ytab`. Последним шагом является выбор схемы кодогенерации: Z-case.

В коде легко найти константы, которые имеют [Y](https://github.com/golang/go/blob/f2239d39571a176ebeb726e2d6cccbc3aa1ba4a9/src/cmd/internal/obj/x86/asm6.go#L97-L173) и [Z](https://github.com/golang/go/blob/f2239d39571a176ebeb726e2d6cccbc3aa1ba4a9/src/cmd/internal/obj/x86/asm6.go#L175-L238) префиксы, но ничего с префиксом X там нет.

<spoiler title="Забавное примечание">
Есть гипотеза, что изначально это были A, B и C префиксы, затем B и C переименовали в Y и Z, а опкоды так и остались с префиксом A.

Что также забавно, тип A-констант - `obj.As`, что может быть сокращением от `asm` (assembler opcode), либо просто означать множественное число `A`.
</spoiler>

Ранее инструкции в Go x86 ассемблер добавлялись вручную, по следующей схеме:

1. Добавление новой константы в [aenum.go](https://github.com/golang/go/blob/master/src/cmd/internal/obj/x86/aenum.go).
2. Добавление `optab` в [глобальную таблицу](https://github.com/golang/go/blob/f2239d39571a176ebeb726e2d6cccbc3aa1ba4a9/src/cmd/internal/obj/x86/asm6.go#L1134) ассемблера x86.
3. Подбор или добавление нужного `ytab` списка.
4. Добавление [end2end](https://github.com/golang/go/tree/master/src/cmd/asm/internal/asm/testdata) тестов для новой инструкции.

Если у нас уже есть все нужные A, Y и Z константы, остаётся генерировать сами таблицы encoder'а и тесты.

Этот процесс неплохо автоматизируется, если у нас есть источник, из которого можно читать информацию об инструкциях: их encoding, типы разрешённых операндов и прочее. 
К счастью, у нас есть такой источник.

# x86avxgen и Intel XED

Для генерации всех инструкций, которые используют [VEX](https://en.wikipedia.org/wiki/VEX_prefix) и [EVEX](https://en.wikipedia.org/wiki/EVEX_prefix) префиксы, была написана утилита [x86avxgen](https://github.com/golang/arch/tree/master/x86/x86avxgen). Данная программа генерирует те самые `optab` и `ytab` объекты для ассемблера.

Входными данными для программы являются [XED datafiles](https://github.com/intelxed/xed/tree/master/datafiles), работать с которыми из Go можно с помощью пакета [xeddata](https://github.com/golang/arch/tree/master/x86/xeddata).

Преимущество кодогенерации в том, что для реализации новых инструкций из серии [AVX](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions) будет достаточно перезапустить `x86avxgen` и добавить тестов.
Генерация тестов также автоматизирована с помощью [Intel XED](https://github.com/intelxed/xed) encoder'а (XED - это в первую очередь библиотека).

EVEX имеет большое свободное пространство для опкодов и потенциал для расширений, так что новые инструкции точно появятся.
В ближайшее будущее можно подглядывать с помощью документа [ISA-extensions](https://software.intel.com/sites/default/files/managed/c5/15/architecture-instruction-set-extensions-programming-reference.pdf).

# Синтаксис

Кроме самих таблиц кодогенератора был обновлён парсер.
Теперь для `x86` можно использовать [списки регистров](https://github.com/golang/go/blob/f2239d39571a176ebeb726e2d6cccbc3aa1ba4a9/src/cmd/internal/obj/link.go#L140-L147) и суффиксы опкодов.

Списки регистров используются для multi-source инструкций, таких как `VP4DPWSSD`.
В [мануале](https://software.intel.com/en-us/articles/intel-sdm) используется нотация `+n`:

```go
VP4DPWSSD zmm1{k1}{z}, zmm2+3, m128
```

В данном случае `+3` означает, что второй `zmm` операнд описывает диапазон регистров из 4 элементов (в мануале эти диапазоны именуются "register block").

Диапазон для `Z0+3` в Go ассемблере будет выглядеть следующим образом:

```go
VP4DPWSSD Z25, [Z0-Z3], (AX)
```

Использование диапазонов типа `[Z0-Z1]`, `[Z3-Z0]`, `[AX-DX]` является ошибкой
этапа ассемблирования.

Суффиксы используются для активации особых возможностей AVX-512.
Например, возьмём одну из новых форм инструкции `VADDPD`:

```go
VADDPD zmm1 {k1}{z}, zmm2, zmm3/m512/m64bcst{er}
```

Сейчас мы разберём, что означает вся эта магия из `{k1}`, `{z}`, `m64bcst` и `{er}`.

> Обратите внимание: порядок операндов полностью обратен Intel синтаксису.
> Точно так же, как и в GNU ассемблере (AT&T синтаксис).

```go
// Без суффиксов, "обычный" VADDPD.
VADDPD (AX), Z30, Z10

// {k1} - merging с использованием маски из K аргумента.
VADDPD (AX), Z30, K5, Z10

// Для всех инструкций ниже можно использовать формы с явным K регистром,
// что будет активировать селективный merging или zeroing.

// {z} - возможен zeroing-mask (без этого суффикса будет merging-mask).
VADDPD.Z (AX), Z30, Z10

// m64bcst - возможность использовать embedded broadcasting.
// Обозначение "bcst" позаимствовано у Microsoft ассемблера (MASM).
VADDPD.BCST (AX), Z30, Z10

// {er} - доступен embedded rounding. Не совместим с memory операндами.
// Округление всегда включает SAE (см. ниже), поэтому суффикс выглядит как составной.
VADDPD.RU_SAE Z0, Z30, Z10 // Округление к +Inf
VADDPD.RD_SAE Z0, Z30, Z10 // Округление к -Inf
VADDPD.RZ_SAE Z0, Z30, Z10 // Округление к нулю
VADDPD.RN_SAE Z0, Z30, Z10 // Округление в "ближайшую сторону"
```

Что более интересно, суффикс `Z`, если инструкция его поддерживает, может использоваться совместно с другими суффиксами:

```go
// SAE - доступен "surpress all exceptions".
// В мануале обозначается меткой {sae}.
VMAXPD.SAE.Z Z3, Z2, Z1
```

На вопросы вроде "А почему именно так?" может ответить [go#22779: AVX512 design](https://github.com/golang/go/issues/22779).
Рекомендуется также пройтись по приводимой там ссылке на [golang-dev](https://groups.google.com/forum/#!topic/golang-dev/DdQX1mN1Rd4).

# Сравнение с GNU ассемблером

Порядок операндов идентичен GNU ассемблеру.

Тех, кто застал "странный" порядок операндов в `CMP` инструкциях, ждёт новость:
для AVX инструкций эти особые правила не действуют (хорошо это или плохо - решайте сами).

| Feature | GNU ассемблер | Go ассемблер |
| ------- | ------------- | ------------ |
| Masking | `VPORD %ZMM0, %ZMM1, %ZMM2{%K2}`<br>`{k}` всегда у dst операнда | `VPODR Z0, Z1, K2, Z2`<br>`{k}` всегда перед dst операндом |
| Broadcasting | `VPORD (%RDX){1to16}, %ZMM1, %ZMM2`<br>`1toN` у аргумента-памяти | `VPORD.BCST (DX), Z1, Z2`<br>`BCST` суффикс |
| Zeroing | `VPORD %ZMM0, %ZMM1, %ZMM2{z}`<br>`{z}` аргумент у dst операнда | `VPORD.Z Z0, Z1, Z2`<br>`Z` суффикс |
| Rounding | `VSQRTPD {ru-sae}, %ZMM0, %ZMM1`<br>Особый первый аргумент | `VSQRTPD.RU_SAE Z0, Z1`<br>Суффикс |
| SAE | `VUCOMISD {sae}, %XMM0, %XMM1`<br>Аналогично rounding | `VUCOMISD.SAE X0, X1`<br>Аналогично rounding |
| Multi-source | `V4FMADDPS (%RCX), %ZMM4, %ZMM1`<br>Указывается первый регистр | `V4FMADDPS (CX), [Z4-Z7], Z1`<br>Явное указание диапазона |

Оба ассемблера при сборке инструкций, где возможно применить и VEX, и EVEX схемы, используют VEX. Иными словами, `VADDPD X1, X2, X3` будет иметь VEX префикс.

В случаях, когда есть неоднозначность размерности операнда, в Go ассемблере опкоды получают дополнительные size-суффиксы:

```go
VCVTSS2USIL (AX), DX // VCVTSS2USI (%RAX), %EDX
VCVTSS2USIQ (AX), DX // VCVTSS2USI (%RAX), %RDX
```

Там, где в Intel синтаксисе можно указать ширину memory операнда, в GNU и Go ассемблерах используются `X` и `Y` size-суффиксы:

```go
VCVTTPD2DQX (AX), X0 // VCVTTPD2DQ XMM0, XMMWORD PTR [RAX]
VCVTTPD2DQY (AX), X0 // VCVTTPD2DQ XMM0, YMMWORD PTR [RAX]
```

Полный список инструкций с size-суффиксами можно найти в [документации](https://github.com/golang/go/wiki/AVX-512-support-in-Go-assembler#instructions-with-size-suffix).

# Дизассемблирование AVX-512

[CL113315](https://golang.org/cl/113315) добавляет поддержку AVX-512 в `go tool asm`, в основном затрагивая парсер и кодогенератор `obj/x86`, но что произойдёт, если вы соберёте `.s` файл и попробуете исследовать его с помощью `go tool objdump`?

```go
// Файл avx.s
TEXT avxCheck(SB), 0, $0
        VPOR X0, X1, X2             // AVX1
        VPOR Y0, Y1, Y2             // AVX2
        VPORD.BCST (DX), Z1, K2, Z2 // AVX-512
        RET
```

Вы увидите не то, что ожидаете:

```bash
$ go tool asm avx.s
$ go tool objdump avx.o

TEXT avxCheck(SB) gofile..$GOROOT/avx.s
  avx.s:2   0xb7   c5f1ebd0   JMP 0x8b
  avx.s:3   0xbb   c5f5ebd0   JMP 0x8f
  avx.s:4   0xbf   62         ?
  avx.s:4   0xc0   f1         ICEBP
  avx.s:4   0xc1   755a       JNE 0x11d
  avx.s:4   0xc3   eb12       JMP 0xd7
  avx.s:5   0xc5   c3         RET
```

Использовать `objdump` на объектных файлах Go не получится:

```bash
$ objdump -D avx.o

objdump: avx.o: File format not recognized
```

Но его можно использовать на исполняемых файлах.
Если ассемблерный код включён в main пакет, системный `objdump` справится с задачей.

Более простым способом получить машинный код является передача `-S` аргумента:

```bash
$ go tool asm -S avx.s

avxCheck STEXT nosplit size=15 args=0xffffffff80000000 locals=0x0
	0x0000 00000 (avx.s:1)	TEXT	avxCheck(SB), NOSPLIT, $0
	0x0000 00000 (avx.s:2)	VPOR	X0, X1, X2
	0x0004 00004 (avx.s:3)	VPOR	Y0, Y1, Y2
	0x0008 00008 (avx.s:4)	VPORD.BCST	(DX), Z1, K2, Z2
	0x000e 00014 (avx.s:5)	RET
	0x0000 c5 f1 eb d0 c5 f5 eb d0 62 f1 75 5a eb 12 c3     ........b.uZ...
go.info.avxCheck SDWARFINFO size=34
	0x0000 02 61 76 78 43 68 65 63 6b 00 00 00 00 00 00 00  .avxCheck.......
	0x0010 00 00 00 00 00 00 00 00 00 00 01 9c 00 00 00 00  ................
	0x0020 01 00
```

Интересующие нас октеты: `c5 f1 eb d0 c5 f5 eb d0 62 f1 75 5a eb 12 c3`.
Скопируем их и будем делать реверс через системный `objdump`:

```bash
# 1. Перегнать текстовое представление в бинарное с помощью xxd.
# 2. Запустить objdump в binary режиме.
# Примечание: для Intel синтаксиса можно вместо "i386" использовать "i386:intel".
$ echo 'c5 f1 eb d0 c5 f5 eb d0 62 f1 75 5a eb 12 c3' |
    xxd -r -p > shellcode.bin &&
    objdump -b binary -m i386 -D shellcode.bin

Disassembly of section .data:

00000000 <.data>:
   0:   c5 f1 eb d0             vpor   %xmm0,%xmm1,%xmm2
   4:   c5 f5 eb d0             vpor   %ymm0,%ymm1,%ymm2
   8:   62 f1 75 5a eb 12       vpord  (%edx){1to16},%zmm1,%zmm2{%k2}
   e:   c3                      ret
```

<spoiler title="Дизассемблирование с помощью XED">
XED также предоставляет несколько полезных утилит, одна из которых позволяет
использовать encoder/decoder через командную строку.

```bash
$ echo 'c5 f1 eb d0 c5 f5 eb d0 62 f1 75 5a eb 12 c3' > data.txt &&
    xed -64 -A -ih data.txt && rm data.txt

00 LOGICAL AVX        C5F1EBD0     vpor %xmm0, %xmm1, %xmm2
04 LOGICAL AVX2       C5F5EBD0     vpor %ymm0, %ymm1, %ymm2
08 LOGICAL AVX512EVEX 62F1755AEB12 vpordl  (%rdx){1to16}, %zmm1, %zmm2{%k2}
0e RET     BASE       C3           retq
```

Флаг `-A` выбирает AT&T синтаксис, `-64` выбирает 64-битный режим.

Утилита `xed-ex4` показывает детальную информацию об инструкции:

```bash
$ xed-ex4 -64 C5 F1 EB D0

PARSING BYTES: c5 f1 eb d0
  VPOR VPOR_XMMdq_XMMdq_XMMdq
  EASZ:3,
  EOSZ:2,
  HAS_MODRM:1,
  LZCNT,
  MAP:1,
  MAX_BYTES:4,
  MOD:3,
  MODE:2,
  MODRM_BYTE:208,
  NOMINAL_OPCODE:235,
  OUTREG:XMM0,
  P4,
  POS_MODRM:3,
  POS_NOMINAL_OPCODE:2,
  REG:2,
  REG0:XMM2,
  REG1:XMM1,
  REG2:XMM0,
  SMODE:2,
  TZCNT,
  VEXDEST210:6,
  VEXDEST3,
  VEXVALID:1,
  VEX_PREFIX:1
0		REG0/W/DQ/EXPLICIT/NT_LOOKUP_FN/XMM_R
1		REG1/R/DQ/EXPLICIT/NT_LOOKUP_FN/XMM_N
2		REG2/R/DQ/EXPLICIT/NT_LOOKUP_FN/XMM_B
YDIS: vpor xmm2, xmm1, xmm0
ATT syntax: vpor %xmm0, %xmm1, %xmm2
INTEL syntax: vpor xmm2, xmm1, xmm0
```

</spoiler>

`go tool objdump` основан на [x86.csv](https://github.com/golang/arch/blob/master/x86/x86.csv), который не содержит многих новых инструкций и имеет неточности.

Сам csv файл создан утилитой [x86spec](https://godoc.org/golang.org/x/arch/x86/x86spec) на основе конвертации из Intel мануала (PDF).
Следующим шагом может быть создание `x86.csv` из таблиц XED'а, что позволит повторно сгенерировать таблицы для decoder'а.

# Применение AVX-512

Одним из крупных пользователей AVX-512 в мире Go является [minio](https://github.com/minio).
До 1.11 им приходилось использовать утилиту [asm2plan9s](https://github.com/minio/asm2plan9s).

Вот, например, их [результаты для sha256](https://github.com/minio/sha256-simd#performance):

```bash
Processor                          SIMD    Speed (MB/s)
3.0 GHz Intel Xeon Platinum 8124M  AVX512  3498
1.2 GHz ARM Cortex-A53             ARM64   638
3.0 GHz Intel Xeon Platinum 8124M  AVX2    449
3.1 GHz Intel Core i7              AVX     362
3.1 GHz Intel Core i7              SSE     299
```

Для того, чтобы начать знакомиться с новым расширением, вы можете попробовать использовать уже знакомые вам по AVX1 и AVX2 инструкции (без `Z` регистров). Таким образом вы сможете эксперементировать с новыми возможностями, типа merging/zeroing масок, без риска попасть в полностью новое пространство "особенностей".

Самое главное - измеряйте, перед тем, как делать окончательные выводы. При этом проверяйте как производительность самой функции, так и приложения в целом.

Рекомендую также ознакомиться с [golang.org/wiki/AVX-512-support-in-Go-assembler](https://github.com/golang/go/wiki/AVX512).

Более детально тема эффективного использования AVX-512 будет затронута в отдельной статье.
