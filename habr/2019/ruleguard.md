# ruleguard: динамические проверки для Go

![](https://habrastorage.org/webt/b5/p-/sq/b5p-sqgr-9b1e5mimtxaftmryau.png)


В этой статье я расскажу о новой библиотеке (и утилите) статического анализа [`go-ruleguard`](https://github.com/quasilyte/go-ruleguard), которая адаптирует [`gogrep`](https://github.com/mvdan/gogrep) для использования внутри линтеров.

Отличительная особенность: правила статического анализа вы описываете на особом Go-подобном DSL, который на старте `ruleguard` превращается в набор диагностик. Возможно, это один из самых легко конфигурируемых инструментов для реализации кастомных инспекций для Go.

В качестве бонуса, мы поговорим об [`go/analysis`](https://godoc.org/golang.org/x/tools/go/analysis) и его [предшественниках](https://github.com/go-lintpack/lintpack).

<cut/>

# Расширяемость статического анализа

Для Go существует [множество](https://github.com/golangci/awesome-go-linters) линтеров, некоторые из которых можно расширять. Обычно для расширения линтера вам требуется написать Go код, использующий специальное API линтера.

Есть два основных пути: [Go plugins](https://golang.org/pkg/plugin/) и монолит. Монолит подразумевает, что все проверки (в том числе ваши личные) доступны на этапе компиляции.

[`revive`](https://github.com/mgechev/revive) требует включения новых проверок в своё ядро для расширения. [`go-critic`](https://github.com/go-critic/go-critic) вдобавок к этому умеет в плагины, что позволяет собирать расширения независимо от основного кода. Оба эти подхода подразумевают, что вы реализуете манипуляции над [`go/ast`](https://golang.org/pkg/go/ast/) и [`go/types`](https://golang.org/pkg/go/types/) на Go, используя API линтера. Даже простые проверки требуют [много кода](https://github.com/mgechev/revive/blob/master/rule/call-to-gc.go).

[`go/analysis`](https://godoc.org/golang.org/x/tools/go/analysis) призван упростить картину тем, что "фреймворк" у линтеров становится практически идентичным, но он не решает проблему сложности технической реализации самих диагностик.

<spoiler title="Отступление о `loader` и `go/packages`"><hr>
Когда вы пишите анализатор для Go, вашей конечной целью является взаимодействие с AST и типами, но перед тем, как вы это сможете сделать, исходные коды нужно правильным образом "загрузить". Если упрощённо, в понятие загрузки входит [парсинг](https://golang.org/pkg/go/parser/), проверка типов и [импортирование зависимостей](https://golang.org/pkg/go/importer/).

Первым шагом в упрощении этого пайплайна стал пакет [`go/loader`](https://godoc.org/golang.org/x/tools/go/loader), который позволят "загрузить" всё нужное через пару вызовов. Всё было почти хорошо, а потом он стал deprecated в пользу [`go/packages`](https://godoc.org/golang.org/x/tools/go/packages). `go/packages` имеет немного улучшенное API и, в теории, хорошо работает с модулями.

Теперь для написания анализаторов лучше всего не использовать ничего из выше перечисленного напрямую, потому что [`go/analysis`](https://godoc.org/golang.org/x/tools/go/analysis) дал `go/packages` то, чего не было ни у одного предыдущего решения - структуру для вашей программы. Теперь мы можем использовать диктуемую `go/analysis` парадигму и переиспользовать работу анализаторов более эффективно. У этой парадигмы есть спорные моменты, например, `go/analysis` хорошо подходит для анализа на уровне одного пакета и его зависимостей, но сделать на нём глобальный анализ без хитрых инженерных ухищрений будет не просто.

`go/analysis` также упрощает [тестирование анализаторов](https://godoc.org/golang.org/x/tools/go/analysis/analysistest).
<hr></spoiler>

# Что же такое ruleguard?

![](https://habrastorage.org/webt/zp/ym/rj/zpymrjjb8zkqa_c069ccd-yf3xg.png)

[`go-ruleguard`](https://github.com/quasilyte/go-ruleguard) - это утилита статического анализа, которая по умолчанию не включает в себя ни единой проверки.

Правила `ruleguard` подгружаются на старте, из специального файла, декларативно описывающего паттерны кода, на которые стоит выдавать предупреждения. Этот файл может свободно редактироваться пользователями `ruleguard`.

Перекомпилировать управляющую программу для подключения новых проверок не нужно, поэтому правила из ruleguard файлов можно называть [динамическими](https://habr.com/ru/company/vk/blog/473718/).

Управляющая программа `ruleguard` выглядит так:

```go
package main

import (
	"github.com/quasilyte/go-ruleguard/analyzer"
	"golang.org/x/tools/go/analysis/singlechecker"
)

func main() {
	singlechecker.Main(analyzer.Analyzer)
}
```

При этом `analyzer` реализован через пакет [`ruleguard`](https://godoc.org/github.com/quasilyte/go-ruleguard/ruleguard), который и нужно использовать в случае, если вы хотите использовать его как библиотеку.

# ruleguard VS revive

Возьмём простой, но реальный пример: предположим, мы хотите избегать вызовов [`runtime.GC()`](https://golang.org/pkg/runtime/#GC) в наших программах. В revive для этого уже есть отдельная диагностика, она называется `"call-to-gc"`.

<spoiler title="Реализация call-to-gc (70 строк на Эльфийском)"><hr>
```go
package rule

import (
	"go/ast"

	"github.com/mgechev/revive/lint"
)

// CallToGCRule lints calls to the garbage collector.
type CallToGCRule struct{}

// Apply applies the rule to given file.
func (r *CallToGCRule) Apply(file *lint.File, _ lint.Arguments) []lint.Failure {
	var failures []lint.Failure
	onFailure := func(failure lint.Failure) {
		failures = append(failures, failure)
	}

	var gcTriggeringFunctions = map[string]map[string]bool{
		"runtime": map[string]bool{"GC": true},
	}

	w := lintCallToGC{onFailure, gcTriggeringFunctions}
	ast.Walk(w, file.AST)

	return failures
}

// Name returns the rule name.
func (r *CallToGCRule) Name() string {
	return "call-to-gc"
}

type lintCallToGC struct {
	onFailure             func(lint.Failure)
	gcTriggeringFunctions map[string]map[string]bool
}

func (w lintCallToGC) Visit(node ast.Node) ast.Visitor {
	ce, ok := node.(*ast.CallExpr)
	if !ok {
		return w // nothing to do, the node is not a call
	}

	fc, ok := ce.Fun.(*ast.SelectorExpr)
	if !ok {
		return nil // nothing to do, the call is not of the form pkg.func(...)
	}

	id, ok := fc.X.(*ast.Ident)

	if !ok {
		return nil // in case X is not an id (it should be!)
	}

	fn := fc.Sel.Name
	pkg := id.Name
	if !w.gcTriggeringFunctions[pkg][fn] {
		return nil // it isn't a call to a GC triggering function
	}

	w.onFailure(lint.Failure{
		Confidence: 1,
		Node:       node,
		Category:   "bad practice",
		Failure:    "explicit call to the garbage collector",
	})

	return w
}
```
<hr></spoiler>

А теперь сравните с тем, как это делается в [`go-ruleguard`](https://github.com/quasilyte/go-ruleguard):

```go
package gorules

import "github.com/quasilyte/go-ruleguard/dsl"

func callToGC(m dsl.Matcher) {
	m.Match(`runtime.GC()`).Report(`explicit call to the garbage collector`)
}
```

Ничего лишнего, только то, что действительно важно - `runtime.GC` и сообщение, которое нужно выдавать в случае срабатывания правила.

Вы можете спросить: и это всё? Я специально начал с такого простого примера, чтобы показать, как много кода может потребоваться для очень тривиальной диагностики в случае традиционного подхода. Обещаю, дальше будут более захватывающие примеры.

# Quick start

В `go-critic` есть диагностика [`rangeExprCopy`](https://go-critic.github.io/overview#rangeExprCopy-ref), которая находит в коде потенциально неожиданные копирования массивов.

Вот этот код итерируется по **копии** массива:

```go
var xs [2048]byte
for _, x := range xs { // Copies 2048 bytes
	// Loop body.
}
```

Исправление этой проблемы заключается в добавлении одного символа:

```diff
  var xs [2048]byte
- for _, x := range xs {  // Copies 2048 bytes
+ for _, x := range &xs { // No copy
  	// Loop body.
  }
```

Скорее всего, вам это копирование не нужно, а производительность исправленного варианта всегда лучше. Можно ждать, пока Go компилятор станет лучше, а можно детектировать такие места в коде и поправить их уже сегодня с помощью того же `go-critic`.

Эту диагностику можно реализовать на нашем DSL (файл `rules.go`):

```go
package gorules

import "github.com/quasilyte/go-ruleguard/dsl"

func rangeExprCopy(m dsl.Matcher) {
    m.Match(`for $_, $_ := range $x { $*_ }`,
            `for $_, $_ = range $x { $*_ }`).
            Where(m["x"].Addressable && m["x"].Type.Size >= 128).
            Report(`$x copy can be avoided with &$x`).
            At(m["x"]).
            Suggest(`&$x`)
}
```

Правило находит все циклы `for-range`, где используются обе итерируемые переменные (именно этот случай ведёт к копированию). Итерируемое выражение `$x` должно быть [`addressable`](https://golang.org/ref/spec#Address_operators) и его размер должен быть выше выбранного порога в байтах.

[`Report()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Report) определяет сообщение, которое нужно выдавать пользователю, а [`Suggest()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Suggest) описывает `quickfix` шаблон, который может использоваться в вашем редакторе через [gopls](https://github.com/golang/tools/tree/master/gopls) (LSP), а также интерактивно, если `ruleguard` вызван с аргументом `-fix` (мы ещё к этому вернёмся). [`At()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.At) привязывает предупреждение **и** `quickfix` к конкретной части шаблона. Нам это необходимо, чтобы заменить `$x` на `&$x`, а не переписать весь цикл.

И `Report()`, и `Suggest()`, принимают строку, в которую можно интерполировать захваченные шаблоном из `Match()` выражения. Предопределённая переменная `$$` означает "весь захваченный фрагмент" (как `$0` в регулярных выражениях).

Создадим файл `rangecopy.go`:

```go
package example

// sizeof(builtins[...]) = 240 on x86-64
var builtins = [...]string{
	"append", "cap", "close", "complex", "copy",
	"delete", "imag", "len", "make", "new", "panic",
	"print", "println", "real", "recover",
}

func builtinID(name string) int {
	for i, s := range builtins {
		if s == name {
			return i
		}
	}
	return -1
}
```

Теперь мы можем запустить `ruleguard`:

```bash
$ ruleguard -rules rules.go -fix rangecopy.go
rangecopy.go:12:20: builtins copy can be avoided with &builtins
```

Если после этого мы посмотрим в `rangecopy.go`, то увидим исправленный вариант, потому что `ruleguard` был вызван с параметром `-fix`.

Простейшие правила можно отлаживать и без создания `rules.go` файла:

```
$ ruleguard -c 1 -e 'm.Match(`return -1`)' rangecopy.go
rangecopy.go:17:2: return -1
16		}
17		return -1
18	}
```

Благодаря использованию [`go/analysis/singlechecker`](https://godoc.org/golang.org/x/tools/go/analysis/singlechecker) у нас есть опция `-c`, которая позволяет выводить указанное строк контекста вместе с самим предупреждением. Управление этим параметром немного контринтуитивно: значение по умолчанию равно `-c=-1`, что означает "без контекста", а `-c=0` будет выводить одну строку контекста (ту, на которую указывает диагностика).

Вот ещё несколько интересных возможностей DSL:
* [Шаблоны типов](https://github.com/quasilyte/go-ruleguard/blob/master/_docs/dsl.md#type-pattern-matching), которые позволяют задавать ожидаемые типы. Например, выражение `map[$t]$t` описывает все мапы, у которых тип значения совпадает с типом ключа, а `*[$len]$elem` захватываем все указатели на массивы.
* Внутри одной функции может быть несколько правил,
   а сами функции стоит называть [группами правил](https://github.com/quasilyte/go-ruleguard/blob/master/_docs/dsl.md#rule-group-statements).
* Правила в группе применяются одно за другим, в порядке их определения. Первое сработавшее правило отменяет сопоставление с оставшимися правилами. Это важно не столько для оптимизации, сколько для специализации правил для конкретных случаев. Примером, где это полезно, является правило переписывания `$x=$x+$y` в `$x+=$y`, для случая с `$y=1` вы хотите предлагать `$x++`, а не `$x+=1`.

Больше информации об используемом DSL можно найти в [`docs/dsl.md`](https://github.com/quasilyte/go-ruleguard/blob/master/_docs/dsl.md).

# Ещё больше примеров

```go
package gorules

import "github.com/quasilyte/go-ruleguard/dsl"

func exampleGroup(m dsl.Matcher) {
        // Находим потенциально некорректные использования json.Decoder.
        // См. http://golang.org/issue/36225
        m.Match(`json.NewDecoder($_).Decode($_)`).
                Report(`this json.Decoder usage is erroneous`)

        // Делаем умный unconvert, предлагая убирать лишние преобразования.
        m.Match(`time.Duration($x) * time.Second`).
                Where(m["x"].Const).
                Suggest(`$x * time.Second`)

        // Предлагаем заменить fmt.Sprint() на вызов метода String(),
        // если у $x таковой имеется.
        m.Match(`fmt.Sprint($x)`).
                Where(m["x"].Type.Implements(`fmt.Stringer`)).
                Suggest(`$x.String()`)

        // Упрощаем логические выражения.
        m.Match(`!($x != $y)`).Suggest(`$x == $y`)
        m.Match(`!($x == $y)`).Suggest(`$x != $y`)
}
```

Если для правила нет вызова [`Report()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Report), то будет использоваться сообщение, выводимое из [`Suggest()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Suggest). Это позволяет в некоторых случаях избежать дублирования.

Фильтры типов и подвыражений могут проверять различные свойства. Например, полезными являются свойства `Pure` и `Const`:
* [`Var.Pure`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Var) означает, что выражение не имеет побочных эффектов.
* [`Var.Const`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Var) означает, что выражение может быть использовано в константном контексте (например, размерность массива).

Для `package-qualified` имён в [`Where()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Where) условиях нужно использовать метод [`Import()`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl#Matcher.Import). Для удобства, все стандартные пакеты импортированы за вас, поэтому в примере выше нам не нужно делать дополнительных импортов.

#  `go/analysis` quickfix actions

Поддержку `quickfix` за нас реализует `go/analysis`.

В модели `go/analysis`, анализатор генерирует [диагностики](https://godoc.org/golang.org/x/tools/go/analysis#Diagnostic) и [факты](https://godoc.org/golang.org/x/tools/go/analysis#Fact). Диагностики отправляются пользователям, а факты предназначены для использования другими анализаторами.

Диагностика может иметь набор [предлагаемых исправлений](https://godoc.org/golang.org/x/tools/go/analysis#SuggestedFix), каждое из которых описывает как нужно изменить исходные коды в указанном диапазоне, чтобы устранить проблему, найденную диагностикой.

Официальное описание доступно в документе [`go/analysis/doc/suggested_fixes.md`](https://github.com/golang/tools/blob/master/go/analysis/doc/suggested_fixes.md).

# Заключение

![](https://habrastorage.org/webt/m3/bs/zw/m3bszwzp2nwkxrnxdnenypatyjk.png)

Попробуйте `ruleguard` на своих проектах, а в случае, если вы нашли баг или хотите попросить новую фичу, [откройте issue](https://github.com/quasilyte/go-ruleguard/issues/new).

Если вам всё ещё сложно придумать применение `ruleguard`, вот примеры:
* Реализация своих собственных диагностик для Go.
* Автоматическая модернизация или рефакторинг кода с помощью `-fix`.
* Сбор статистики по коду с обработкой [`-json`](https://github.com/golang/tools/blob/master/go/analysis/internal/analysisflags/flags.go#L76) результата анализатора.

Планы по развитию `ruleguard` на ближайшее будущее:

* Внедрить `ruleguard` в [`go-critic`](https://github.com/go-critic/go-critic) как один из способов его расширения.
* Испробовать идеи из [Applied Go code similarity analysis](https://github.com/quasilyte/talks/tree/master/2019-7-Oct-moscow) ([нормализация кода](https://github.com/quasilyte/astnorm)).
* Добавлять новые возможности в DSL. [sub-matches](https://github.com/quasilyte/go-ruleguard/issues/28) может быть полезным дополнением.

# Полезные ссылки и ресурсы

* Примеры диагностик можно найти в [go-ruleguard/rules](https://github.com/quasilyte/go-ruleguard/blob/master/rules)
* [Документация `dsl`](https://godoc.org/github.com/quasilyte/go-ruleguard/dsl)
* [Документация `ruleguard`](https://godoc.org/github.com/quasilyte/go-ruleguard/ruleguard)
* Используемый движок для матчинга AST: [`mvdan/gogrep`](https://github.com/mvdan/gogrep)
* [Динамические проверки в NoVerify](https://habr.com/ru/company/vk/blog/473718/)
