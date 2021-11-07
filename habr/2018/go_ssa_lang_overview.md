# Компилятор Go: язык описания правил SSA оптимизаций

![](https://habrastorage.org/webt/0b/vq/ua/0bvquazri632jnibzwkywt7hfim.png)

В компиляторе `gc` для описания [Static Single Assignment](https://en.wikipedia.org/wiki/Static_single_assignment_form) (SSA) правил оптимизаций используется специальный Лисп-подобный предметно-ориентированный язык ([DSL](https://en.wikipedia.org/wiki/Domain-specific_language)).

Предлагаю разобрать основные элементы этого языка, его особенности и ограничения.
В качестве упражнения, добавим в Go компилятор генерацию инструкции, которую он раньше не генерировал, оптимизируя выражение `a*b+c`.

Это первая статья из серии про внутренности [Go SSA compiler backend](https://github.com/golang/go/tree/master/src/cmd/compile/internal/ssa), поэтому помимо обзора самого DSL описания правил мы рассмотрим связанные компоненты, чтобы создать необходимую базу для нашей следующей сессии.

<cut/>

# Введение

Frontend Go компилятора заканчивается на моменте генерации SSA представления из аннотированного AST. Функции, ответственные за конвертацию можно найти в [cmd/compile/internal/gc/ssa.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/gc/ssa.go). Точкой входа в SSA backend является функция `ssa.Compile`, определённая в файле [cmd/compile/internal/ssa/compile.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/compile.go).

<spoiler title="Терминология">

| EN | RU | Значение |
|------|------|----------|
| Compiler frontend | Фронтенд компилятора | Парсинг и лексический анализ, иногда разрешение типов, промежуточное представление близко к исходному коду, обычно какое-нибудь аннотированное AST. |
| Compiler backend | Бэкенд компилятора | Более низкоуровневые оптимизации и промежуточное представление, кодогенерация. |
| Form | Форма | Практически синоним слову "выражение" (expression). Обычно в Лиспах `form` - довольно распространённый способ именовать элемент программы, будь то список или атом. |
| Optimization pass | Фаза оптимизации | Выполнение определённого алгоритма над программой. Слово "проход" несколько неоднозначно, потому что одна фаза может выполнять несколько проходов, и/или использовать общий с другими фазами код. |

Если по мере прочтения статьи вы нашли совершенно непонятный для вас термин, стоит сообщить об этом, возможно он будет добавлен в эту таблицу.

</spoiler>

SSA оптимизатор Go состоит из нескольких фаз, каждая из которых выполняет проходы по компилируемой функции. Некоторые фазы используют так называемые "rewrite rules", правила преобразования одних SSA последовательностей в другие, потенциально более оптимальные.

Правила преобразования описываются с помощью [S-выражений](https://en.wikipedia.org/wiki/S-expression). Элементы этих выражений - [ssa.Value](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/value.go#L19). В простейшем случае эти правила позволяют заменить один `ssa.Value` на другой.

Например, код ниже сворачивает умножение 8-битных констант:
```lisp
(Mul8 (Const8 [c]) (Const8 [d])) -> (Const8 [int64(int8(c*d))])
```

Есть две основные категории SSA значений: высокоуровневые, почти не зависящие от целевой машины и те, что архитектурно-специфичны (обычно отображаются на машинные инструкции 1-в-1).

Оптимизации описываются в терминах этих двух категорий. Сначала высокоуровневые и общие для всех архитектур, затем платформенно-ориентированные.

Весь код, связанный с правилами, лежит в [cmd/compile/internal/ssa/gen](https://github.com/golang/go/tree/master/src/cmd/compile/internal/ssa/gen). Мы будем рассматривать два набора:
1. [genericOps.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/gen/genericOps.go) - машино-независимые операции.
2. [AMD64Ops.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/gen/AMD64Ops.go) - операции, специфичные для `GOARCH=AMD64` (64-bit x86).

После нескольких первых фаз, которые работают над абстрактной машиной, выполняется так называемый lowering, в результате которого происходит переход из `genericOps` в набор конкретной архитектуры. В нашем примере это будет `AMD64Ops`. После этого момента, все последующие фазы оперируют над представлением из второй категории.

После оптимизатора в игру вступает кодогенератор. Для AMD64 реализацию кодогенерации можно найти в пакете [cmd/compile/internal/amd64](https://github.com/golang/go/tree/master/src/cmd/compile/internal/amd64). Задача кодогенератора - заменить `ssa.Block` и `ssa.Value` в последовательность [obj.Prog](https://github.com/golang/go/blob/0dc814cd7f6a5c01213169be17e823b69e949ada/src/cmd/internal/obj/link.go#L271), передаваемые [ассемблеру](https://github.com/golang/go/tree/master/src/cmd/internal/obj/x86). Ассемблер соберёт машинный код, который будет готов к исполнению после [линковки](https://github.com/golang/go/tree/master/src/cmd/link). 

# Правила оптимизаций

Если файлы с определениями операций имеют название вида "`${ARCH}Ops.go`", то правила оптимизаций размещаются в "`${ARCH}.Rules`".

Высокоуровневые правила выполняют простейшие преобразования, большую часть [сворачивания константных выражений](https://en.wikipedia.org/wiki/Constant_folding), а также некоторые преобразования, которые упрощают последующую обработку.

Каждый файл с низкоуровневыми правилами состоит из двух частей:
1. Lowering - замена абстрактных операций на машинные эквиваленты.
2. Непосредственно сами оптимизации.

Пример сведения операции к машинной:
```lisp
(Const32 [val]) -> (MOVLconst [val]) // L - long, 32-бита
(Const64 [val]) -> (MOVQconst [val]) // Q - quad, 64-бита
 |                  |
 generic op         |
                   AMD64 op
```

Именно в низкоуровневых оптимизациях выполняется основное количество важных оптимизаций, таких как [снижение стоимости операций](https://en.wikipedia.org/wiki/Strength_reduction), частичное встраивание и утилизация возможностей доступных в процессоре [режимов адресации памяти](https://en.wikipedia.org/wiki/Addressing_mode).

Операции имеют мнемоническое имя, которое обычно называется опкодом (opcode). Опкоды архитектурно-зависимых операций, как правило, отражают названия реальных инструкций.

# Синтаксис языка описания правил

Базовая грамматика описана в [rulegen.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/gen/rulegen.go):

```go
// rule syntax:
//    sexpr [&& extra conditions] -> [@block] sexpr
//
// sexpr are s-expressions (lisp-like parenthesized groupings)
// sexpr ::= [variable:](opcode sexpr*)
//         | variable
//         | <type>
//         | [auxint]
//         | {aux}
//
// aux      ::= variable | {code}
// type     ::= variable | {code}
// variable ::= some token
// opcode   ::= one of the opcodes from the *Ops.go files
```

<spoiler title="Перевод сниппета выше">

```go
// синтаксис правил:
//    sexpr [&& дополнительные условия] -> [@block] sexpr
//
// sexpr - это S-выражений (группирование в стиле Лиспа)
// sexpr ::= [variable:](opcode sexpr*)
//         | variable
//         | <type>
//         | [auxint]
//         | {aux}
//
// aux      ::= variable | {code}
// type     ::= variable | {code}
// variable ::= Go переменная (любой токен)
// opcode   ::= опкод из *Ops.go файла
```

</spointer>

Стоит также упомянуть, что внутри `.Rules` файлов разрешены "`//`" комментарии.

Разберём простой пример, который содержит все эти элементы:

```lisp
   Opcode=ADDLconst - сложение аргумента с 32-битной константой
     :    AuxInt=c - константа, которая прибавляется к `x`
     :      :
(ADDLconst [c] x) && int32(c)==0 -> x
|              /  |           /     |
|             /   |          /      |
|            /    |         /       Форма для замены
|           /     Условие замены (через `&&` можно добавить ещё условий)
Форма, которую мы пытаемся заменить
```

> Все эти пояснительные подписи не являются частью валидной записи правил.

Данное правило преобразует `x+0` в `x`. Всё внутри секции условий - это обычный Go код,
разве что ограниченный до выражений, результатом которых должен быть `bool`.
Можно вызывать предикаты, определённые в [rewrite.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/rewrite.go).

Кроме обычных опкодов, могут использоваться комбинации, которые порождают несколько правил:

```lisp
(ADD(Q|L)const [off] x:(SP)) -> (LEA(Q|L) [off] x)
// Уберём Q|L alternation:
(ADDQconst [off] x:(SP)) -> (LEAQ [off] x)
(ADDLconst [off] x:(SP)) -> (LEAL [off] x)
// Уберём привязку к `x`:
(ADDQconst [off] (SP)) -> (LEAQ [off] (SP))
(ADDLconst [off] (SP)) -> (LEAL [off] (SP))
```

> `(SP)` - это одна из операций в genericOps.go и выражает загрузку указателя на аппаратный стек. Для архитектур, где аппаратного `SP` нет, он эмулируется.

Особенности переменных в шаблонах (S-выражения слева от `->`):
* Переменные, типа `x`, без выражения через `:`, захватывают что угодно
* аналогично обычной переменной, `_` захватывает любое значение, но результат можно игнорировать

```lisp
// Оба правила делают одно и то же: реализуют функцию идентичности ADDQconst,
// то есть они возвращают совпавшую с шаблоном форму без изменений:
(ADDQconst _) -> v
(ADDQconst x) -> (ADDQconst x)
```

Если `AuxInt` не указан (выражение в квадратных скобках), то правило будет срабатывать на любом значении `AuxInt`. Аналогично с `{}`-параметрами (о них ниже).

Имя `v` означает самую внешнюю захваченную форму.<br>
Например, для выражения `(ADDQconst (SUBQconst x))` внешней формой является `ADDQconst`.

Переменные можно использовать несколько раз, это позволяет требовать соответствия нескольких частей S-выражения между собой:
```lisp
(ADDQconst [v] (ADDQconst [v] x))
// Сработает, например, для "x+2+2" (x+v+v).
```

# Типы в правилах

В некоторых случаях требуется явно указывать тип генерируемой и/или сопостовляемой формы.
Указывается тип в "треугольных скобках", как аргумент-тип в шаблонах C++:

```lisp
// typ.UInt32 - тип операции BTSLconst.
// BSFL имеет фиксированный тип typ.UInt32, поэтому указывать для
// него тип не требуется.
(Ctz16 x) -> (BSFL (BTSLconst <typ.UInt32> [16] x))
```

Кроме типов, существуют "символы" (или, более универсально - `Aux` свойства).
```list
(StaticCall [argsWidth] {target} mem) -> (CALLstatic [argsWidth] {target} mem)
```

* `[argsWidth]` - `Value.AuxInt`. Для `StaticCall` - суммарный размер передаваемых аргументов
* `{target}` - `Value.Aux`. Для `StaticCall` - вызываемая функция
* `<typ.UInt32>` - `Value.Type`. Тип значения

Семантика `Aux` и `AuxInt` сильно варьируется от операции к операции. Лучшим источником документации в данном случае являются `*Ops.go` файлы. У каждого определения опкода `opData` есть поле `aux`, которое описывает как именно интерпретировать эти поля.

Для описания типов используется пакет [cmd/compile/internal/types](https://github.com/golang/go/tree/master/src/cmd/compile/internal/types). Некоторые типы специфичны для SSA бэкенда, например `types.TypeFlags`, остальные общие между `cmd/compile/internal/gc` и `cmd/compile/internal/ssa`.

# Особые типы

Существует особый тип `types.TypeMem`, который выполняет сразу несколько функций:
1. Позволяет сортировать и группировать `ssa.Value` по паттернам доступа к памяти. В частности, это гарантирует правильный порядок выполнения в рамках базовых блоков (о них ниже).
2. Определяет состояние потока памяти в программе. Если инструкция модифицирует память, новое SSA значение с типом `types.TypeMem` будет сгенерировано в результате этой операции.

Подобно особому значению `OpPhi`, память интерпретируется исключительным образом во многих фазах оптимизатора.

<spoiler title="Немного о Phi">
`Phi` имеет роль, которая немного меняется от фазы к фазе.

В самом начале работы SSA части компилятора, `Phi` служит своей классической цели и выражает выбор значения в зависимости от пути исполнения, который привёл нас к этому значению.

Например, если в блок есть два прыжка, и оба модифицируют память, то блок-назначение получит память равную `(Phi mem1 mem2)`. Циклы также протаскивают `Phi` значение.
</spoiler>

Другим особым типом является упомянутый выше `types.TypeFlags`. Этот тип описывает генерацию инструкцией [CPU флагов](https://en.wikipedia.org/wiki/FLAGS_register).

При этом инструкции, вроде `ADDQ`, хоть и генерируют флаги, не имеют тип `types.Flags`, но отмечены атрибутом `clobberFlags`.

`types.Flags` используется для выделения результата инструкций вроде `CMPQ`, которые не пишут результат ни в один из своих явных операндов, а только обновляют внутреннее состояние процессора, которое может использоваться следующей инструкцией.

Инструкции, вроде `SETL` позволяют "прочитать" флаги и вернуть их в виде `ssa.Value`, который может быть размещён в регистре.

```lisp
 L-less than               G-greater than
 |                         |
(SETL (InvertFlags x)) -> (SETG x)
                   |
                   Форма, генерирующая флаги
```

# Инспекция SSA программы

Допустим, у нас есть такая программа (`example.go`):
```go
package example

func fusedMulAdd(a, b, c float64) float64 {
	return a*c + b
}
```

Мы можем просмотреть SSA код, который сгенерирован для функции `fusedMulAdd`:
```bash
$ GOSSAFUNC=fusedMulAdd go tool compile example.go > ssa.txt
```

Теперь проверьте рабочую (текущую) директорию:
* `ssa.txt` содержит тектовой дамп.
* `ssa.html`, который генерируется автоматически, содержит ту же информацию, но в более интерактивном и удобночитабельном формате. Попробуйте открыть в браузере.

<spoiler title="Машинный код для fusedMulAdd">
Символ `~r3` переименован в `ret` для выразительности.

```asm
v7  (4) MOVSD a(SP), X0
v11 (4) MOVSD c+16(SP), X1
v12 (4) MULSD X1, X0
v6  (4) MOVSD b+8(SP), X1
v13 (4) ADDSD X1, X0
v15 (4) MOVSD X0, ret+24(SP)
b1  (4) RET
```
</spoiler>

Вот так выглядит SSA программа для `fusedMulAdd` после фазы `lower` (ssa.html):

![](https://habrastorage.org/webt/zs/yb/ax/zsybaxjcr1s0_u1csmhgkt3d1za.png)

<spoiler title="Текстовой формат SSA программы">
Если вам почему-то захотелось это скопировать:
```
lower [77667 ns]
b1:
    v1 (?) = InitMem <mem>
    v2 (?) = SP <uintptr>
    v7 (?) = LEAQ <*float64> {~r3} v2
    v8 (3) = Arg <float64> {a}
    v9 (3) = Arg <float64> {b}
    v10 (3) = Arg <float64> {c}
    v12 (+4) = MULSD <float64> v8 v10
    v13 (4) = ADDSD <float64> v12 v9
    v14 (4) = VarDef <mem> {~r3} v1
    v15 (4) = MOVSDstore <mem> {~r3} v2 v13 v14
Ret v15 (line +4)
```
</spoiler>

Переводя это в S-выражения:

```lisp
(MOVQstore {~r3} 
           (SP)
           (ADDSD (MULSD (Arg {a})
                         (Arg {c}))
                  (Arg {b})))
```

<spoiler title="SSA после фазы regalloc">
Так выглядит вывод `ssa.html` для фазы `regalloc`.

![](https://habrastorage.org/webt/ef/kx/u5/efkxu5pdwqvs14c9xwblajxoywo.png)

```
regalloc [87237 ns]
b1:
    v1 (?) = InitMem <mem>
    v14 (4) = VarDef <mem> {~r3} v1
    v2 (?) = SP <uintptr> : SP
    v8 (3) = Arg <float64> {a} : a[float64]
    v9 (3) = Arg <float64> {b} : b[float64]
    v10 (3) = Arg <float64> {c} : c[float64]
    v7 (4) = LoadReg <float64> v8 : X0
    v11 (4) = LoadReg <float64> v10 : X1
    v12 (+4) = MULSD <float64> v7 v11 : X0
    v6 (4) = LoadReg <float64> v9 : X1
    v13 (4) = ADDSD <float64> v12 v6 : X0
    v15 (4) = MOVSDstore <mem> {~r3} v2 v13 v14
Ret v15 (line +4)
```
</spoiler>

# Добавление новых правил оптимизаций

На процессорах с [FMA](https://en.wikipedia.org/wiki/FMA_instruction_set) мы можем вычислять `a*c + b` за одну инструкцию вместо двух.

Возьмём за основу [CL117295](https://go-review.googlesource.com/c/go/+/117295) за авторством [Ильи Токаря](https://github.com/TocarIP).

Для вашего удобства, я подготовил [минимальный `diff` патч](https://gist.github.com/Quasilyte/0d4dbb0f8311f38d00a7b2d25dcec704).<br>

### 1. Добавление новой операции - FMASD

В файле `compile/internal/ssa/gen/AMD64Ops.go` найдите переменную-слайс `AMD64ops` и добавьте туда новый элемент (в любое место):

```go
{ // fp64 fma
  name: "FMASD",      // Опкод для SSA
  argLength: 3,
  reg: fp31,          // Информация для regalloc, спецификатор регистров
  resultInArg0: true, // Первый аргумент является и source, и destination
  asm: "VFMADD231SD", // Ассемблерный опкод
},
```

Поскольку ранее `(fp,fp,fp -> fp)` операций не было, нужно определить новый спецификатор:

```diff
  fp01     = regInfo{inputs: nil, outputs: fponly}
  fp21     = regInfo{inputs: []regMask{fp, fp}, outputs: fponly}
+ fp31     = regInfo{inputs: []regMask{fp, fp, fp}, outputs: fponly}
```

### 2. Добавление правила оптимизации

```lisp
(ADDSD (MULSD x y) z) -> (FMASD z x y)
```

Более правильная реализация не была бы безусловной и проверяла бы доступность FMA. Мы будем считать, что на нашей целевой машине FMA точно есть.

В компиляторе для подобных проверок используется `config`:
```lisp
// Если config.useFMA=false, оптимизация выполняться не будет.
(ADDSD (MULSD x y) z) && config.useFMA-> (FMASD z x y)
```

<spoiler title="Как проверить поддержку FMA?">
Если в системе доступен `lscpu`, то, например так:
```bash
$ lscpu | grep fma
```
</spoiler>

### 3. Реализация кодогенерации

Теперь в функцию `ssaGenValue`, определённую в файле `compile/internal/amd64/ssa.go`, нужно добавить кодогенерацию для `FMASD`:

```go
func ssaGenValue(s *gc.SSAGenState, v *ssa.Value) {
  switch v.Op {
  case ssa.OpAMD64FMASD:
    p := s.Prog(v.Op.Asm()) // Создание нового obj.Prog в текущем блоке
    // From: первый source операнд.
    p.From = obj.Addr{Type: obj.TYPE_REG, Reg: v.Args[2].Reg()}
    // To: destination операнд.
    // v.Reg() возвращает регистр, который был выделен для результата FMASD.
    p.To = obj.Addr{Type: obj.TYPE_REG, Reg: v.Reg()}
    // From3: второй source операнд.
    // Название From3 историческое. На самом деле заполняется
    // поле RestArgs, который содержит все source операнды, кроме первого.
    p.SetFrom3(obj.Addr{
      Type: obj.TYPE_REG,
      Reg: v.Args[1].Reg(),
    })
    if v.Reg() != v.Args[0].Reg() { // Валидация условия resultInArg0
      s := v.LongString()
      v.Fatalf("input[0] and output not in same register %s", s)
    }
  // Остальной код остаётся неизменным, мы только добавляем новый case.
  }
}
```

Теперь всё готово, чтобы проверять работу нашей новой оптимизации. Добавлять новые инструкции приходится очень редко, обычно новые оптимизации делают на основе уже определённых SSA операций.

# Проверка результатов

Первым шагом является генерация Go кода из обновлённых `gen/AMD64Ops.go` и `gen/AMD64.Rules`.

```bash
# Если GOROOT не задан, перейдите в директорию, которую выводит `go env GOROOT`.
cd $GOROOT/src/cmd/compile/internal/ssa/gen && go run *.go
```

Далее нужно собрать наш новый компилятор:
```bash
go install cmd/compile
```

Теперь при компиляции того же примера, мы получим иной машинный код:

```diff
- v7  (4) MOVSD a(SP), X0
- v11 (4) MOVSD c+16(SP), X1
- v12 (4) MULSD X1, X0
- v6  (4) MOVSD b+8(SP), X1
- v13 (4) ADDSD X1, X0
- v15 (4) MOVSD X0, ret+24(SP)
- b1  (4) RET
+ v12 (4) MOVSD b+8(SP), X0
+ v7  (4) MOVSD a(SP), X1
+ v11 (4) MOVSD c+16(SP), X2
+ v13 (4) VFMADD231SD X2, X1, X0
+ v15 (4) MOVSD  X0, ret+24(SP)
+ b1  (4) RET
```

# Базовые блоки (basic blocks)

Теперь, когда самая сложная работа сделана, поговорим о [базовых блоках](https://en.wikipedia.org/wiki/Basic_block).

Значения, которые мы оптимизировали выше, находятся в блоках, а блоки находятся в функции.

Блоки, как и `ssa.Value`, бывают абстрактными и машино-зависимыми. Все блоки имеют ровно одну точку входа и от 0 до 2 блоков-назначений (зависит от типа блока).

Самыми простыми блоками являются `If`, `Exit` и `Plain`:
* `Exit` блок имеет 0 блоков-назначений. Это листовые блоки, которые совершают нелокальный прыжок, например, с помощью `panic`.
* `Plain` блок имеет 1 блоков-назначение. Можно рассматривать как безусловный переход после выполнения всех инструкций блока в другой блок.
* `If` блок имеет 2 блока-назначения. Переход осуществляется в зависимости от условия (`Block.Control`).

Вот простые примеры переписывания абстрактных блоков в блоки `AMD64`:
```lisp
                Тело "then" (является блоком)
                |   Тело "else" (является блоком)
                |   |
(If (SETL  cmp) yes no) -> (LT cmp yes no)
(If (SETLE cmp) yes no) -> (LE cmp yes no)
```

Тему блоков более подробно будем рассматривать в контексте других оптимизирующих фаз в SSA части компилятора.

# Ограничения правил оптимизаций

У SSA бэкенда есть свои преимущества. Некоторые оптимизации выполнимы за `O(1)`. Однако есть и недостатки, из-за которых одного лишь SSA оптимизатора будет маловато, по крайней мере пока не изменятся некоторые аспекты его реализации.

Предположим, вы хотите выполнять [слияние вызовов `append`](https://go-critic.github.io/overview#appendCombine-ref):
```go
xs = append(xs, 'a')
xs = append(xs, 'b')
// =>
xs = append(xs, 'a', 'b')
```

На момент, когда SSA сгенерирован, высокоуровневая структура кода утрачена и `append`, будучи не обычной функцией, уже встроен в тело содержащего блока. Вам придётся писать громоздкое правило, которое захватывает всю сгенерированную для `append` последовательность операций.

Если говорить конкретно о `.Rules`, то в этом DSL довольно слабая работа с блоками (`ssa.Block`). Любая нетривиальная оптимизация, связанная с блоками, невозможна для выражения на этом языке. Частичное обновление блока невозможно. Выбрасывать блоки тоже невозможно (но есть хак в виде `First` блока, который используется для удаления мёртвого кода).

> Даже если часть недостатков исправима, большинство компиляторов сходятся во мнении, что нет единой и лучшей промежуточной формы для представления кода.

# Go that goes faster

Если придумаете какие-то крутые правила оптимизации, смело высылайте на [go-review.googlesource.com](https://go-review.googlesource.com). Буду рад произвести ревью (добавляйте в CC `iskander.sharipov@intel.com`).

Счастливого хакинга компилятора!

![](https://habrastorage.org/webt/op/wr/50/opwr50a-n6imbex_ykcxpia5_ny.gif)

## Бонусный материал

Примеры хороших патчей в Go, которые добавляли или изменяли SSA rules:
* [CL99656: cmd/compile/internal/ssa: emit IMUL3{L/Q} for MUL{L/Q}const on x86](https://go-review.googlesource.com/c/go/+/99656)
* [CL102277: cmd/compile/internal/ssa: optimize away double NEG on amd64](https://go-review.googlesource.com/c/go/+/102277)
* [CL54410: cmd/compile/internal/ssa: use sse to zero on amd64](https://go-review.googlesource.com/c/go/+/54410)
* [CL58090: cmd/compile/internal/ssa: remove redundant zeroextensions on amd64](https://go-review.googlesource.com/c/go/+/58090)
* [CL95475: cmd/compile/internal/ssa: combine byte stores on amd64](https://go-review.googlesource.com/c/go/+/95475)
* [CL97175: cmd/compile/internal/ssa: combine consecutive LittleEndian stores on arm64](https://go-review.googlesource.com/c/go/+/97175)
* [CL115617: cmd/compile/internal/ssa: remove useless zero extension](https://go-review.googlesource.com/c/go/+/115617)
* [CL101275: cmd/compile: add amd64 LEAL{1,2,4,8} ops](https://go-review.googlesource.com/c/go/+/101275)

Не так давно появился [README](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/README.md) документ для описания SSA части компилятора.<br>
Рекомендуется к прочтению.
