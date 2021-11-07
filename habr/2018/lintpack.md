# Go lintpack: менеджер компонуемых линтеров

![](https://habrastorage.org/webt/65/mx/ms/65mxmsgairo1a_s7civi_cgklqq.png)

[lintpack](https://github.com/go-lintpack/lintpack) - это утилита для сборки линтеров (статических анализаторов), которые написаны с использованием предоставляемого API. На основе него сейчас переписывается знакомый некоторым статический анализатор [go-critic](https://github.com/go-critic/go-critic).

Сегодня мы подробнее разберём что такое `lintpack` с точки зрения пользователя.

<cut/>

# В начале был go-critic...

[go-critic](https://github.com/go-critic/go-critic) начинался как экспериментальный проект, который являлся песочницей для прототипирования практически любых идей в области статического анализа для Go.

Приятным удивлением было то, что некоторые люди действительно отправляли реализации детекторов тех или иных проблем в коде. Всё было под контролем, пока не начал накапливаться технический долг, устранять который было практически некому. Люди приходили, добавляли проверку, а затем исчезали. Кто после этого должен исправлять ошибки и дорабатывать реализацию?

Знаменательным событием было предложение добавить проверки, требующие дополнительной конфигурации, то есть такие, которые зависят от локальных для проекта договорённостей. Примером является выявление наличия copyright заголовка в файле (license header) по особому шаблону или запрет импортирования некоторых пакетов с предложением заданной альтернативы.

Другой трудностью была расширяемость. Отправлять свой код в чужой репозиторий удобно не каждому. Некоторым хотелось динамического подключения своих проверок, чтобы не нужно было модифицировать исходные коды `go-critic`.

Резюмируя, вот проблемы, которые стояли на пути развития `go-critic`:
* Груз сложности. Слишком много поддерживать, наличие бесхозного кода.
* Низкий средний уровень качества. `experimental` означал как "почти готово к использованию", так и "лучше не запускать вообще".
* Иногда трудно принимать решение включения проверки в `go-critic`, а отклонять их противоречит исходной философии проекта.
* Разные люди видели `go-critic` по-разному. Большинству хотелось иметь его в виде CI линтера, который идёт в поставке с `gometalinter`.

Чтобы хоть как-то ограничить количество разночтений и несовпадающих интерпретаций проекта, был написан [манифест](https://github.com/go-critic/go-critic/blob/master/docs/manifest.md).

> Если вам хочется дополнительного исторического контекста и ещё больше размышлений на тему категоризации статических анализаторов, можете послушать запись [GoCritic — новый статический анализатор для Go](https://www.youtube.com/watch?v=6SDk8ibowW4). В тот момент lintpack ещё не существовал, но часть идей  родилась именно в тот день, после доклада.

А что если бы нам не нужно было хранить все проверки в одном репозитории?

# Встречайте - [lintpack](https://github.com/go-lintpack/lintpack)

<p>
    <img title="Первоначальный вариант логотипа проекта" src="https://habrastorage.org/webt/_h/s3/wr/_hs3wriufjyktqsybtqsnkhka6m.png"/>
</p>

`go-critic` состоит из двух основных компонентов:
1. Реализация самих проверок.
2. Программа, которая загружает проверяемые Go пакеты и запускает на них проверки.

Наша цель: иметь возможность хранить проверки для линтера в разных репозиториях и собирать их воедино, когда это необходимо.

[lintpack](https://github.com/go-lintpack/lintpack) делает именно это. Он определяет функции, позволяющие описывать свои проверки таким образом, что их затем можно запускать через генерируемый линтер.

> Пакеты, которые реализованы с использованием `lintpack` как фреймворка, будем называть `lintpack`-совместимыми или `lintpack`-compatible пакетами.

Если бы сам `go-critic` был реализован на основе `lintpack`, все проверки можно было бы разделить на несколько репозиториев. Одним из вариантов разделения может быть следующий:
1. Основной набор, куда попадают все стабильные и поддерживаемые проверки.
2. contrib репозиторий, где лежит код, который либо слишком экспериментальный, либо не имеет меинтейнера.
3. Настраиваемые под конкретный проект проверки.

Первый пункт имеет особо важное значение в связи с [интеграцией go-critic в golangci-lint](https://github.com/golangci/golangci-lint/releases/tag/v1.12).

Если оставаться на уровне `go-critic`, то для пользователей практически ничего не изменилось. `lintpack` создаёт почти идентичный прежнему линтер, а `golangci-lint` инкапсулирует все различающиеся детали реализации.

Но кое-что всё же изменилось. Если на основе `lintpack` будут создаваться новые линтеры, у вас появится более богатый выбор готовых диагностик для генерации линтера. На минуту представим, что это так, и в мире существует более 10 разных наборов проверок.

# Quick start

![](https://habrastorage.org/webt/jk/kn/2f/jkkn2fql7m45ueypxb7dytrvol8.jpeg)

Для начала, нужно установить сам `lintpack`:

```bash
# lintpack будет установлен в `$(go env GOPATH)/bin`.
go get -v github.com/go-lintpack/lintpack/...
```

Создадим линтер, используя тестовый пакет из `lintpack`:

```bash
lintpack build -o mylinter github.com/go-lintpack/lintpack/checkers
```

В набор входит `panicNil`, который находит в коде `panic(nil)` и просить выполнить замену на что-то различимое, поскольку в противном случае `recover()` не сможет подсказать, был ли вызван `panic` с `nil` аргументом, или паники не было вовсе.

<spoiler title="Пример с panic(nil)"><hr>

Код ниже пытается описать значение, полученное из `recover()`:

```go
r := recover()
fmt.Printf("%T, %v\n", r, r)
```

Результат будет идентичен для `panic(nil)` и для программы, которая не паникует.

[Запускаемый пример описываемого поведения](https://play.golang.org/p/04CfT3Sy2Rc).

<hr></spoiler>

Запускать линтер можно на отдельных файлах, аргументами типа `./...` или пакетах (по их import пути).

```bash
./mylinter check bytes
$GOROOT/src/bytes/buffer_test.go:276:3: panicNil: panic(nil) calls are discouraged
```

```bash
# Далее делается предположение, что go-lintpack есть под вашим $GOPATH.
mylinter=$(pwd)/mylinter

cd $(go env GOPATH)/src/github.com/go-lintpack/lintpack/checkers/testdata

$mylinter check ./panicNil/
./panicNil/positive_tests.go:5:3: panicNil: panic(nil) calls are discouraged
./panicNil/positive_tests.go:9:3: panicNil: panic(interface{}(nil)) calls are discouraged
```

По умолчанию данная проверка также реагирует на `panic(interface{}(nil))`. Чтобы переопределить это поведение, нужно установить значение `skipNilEfaceLit` в `true`. Сделать это можно через командную строку:

```bash
$mylinter check -@panicNil.skipNilEfaceLit=true ./panicNil/
./panicNil/positive_tests.go:5:3: panicNil: panic(nil) calls are discouraged
```

<spoiler title="usage для cmd/lintpack и генерируемого линтера"><hr>

И `lintpack`, и генерируемый линтер, используют первый аргумент для выбора подкоманды. Список доступных подкоманд и примеров их запуска можно получить вызвав утилиту без аргументов.

```bash
lintpack
not enough arguments, expected sub-command name

Supported sub-commands:
	build - build linter from made of lintpack-compatible packages
		$ lintpack build -help
		$ lintpack build -o gocritic github.com/go-critic/checkers
		$ lintpack build -linter.version=v1.0.0 .
	version - print lintpack version
		$ lintpack version
```

Предположим, мы назвали созданный линтер именем `gocritic`:

```bash
./gocritic
not enough arguments, expected sub-command name

Supported sub-commands:
	check - run linter over specified targets
		$ linter check -help
		$ linter check -disableTags=none strings bytes
		$ linter check -enableTags=diagnostic ./...
	version - print linter version
		$ linter version
	doc - get installed checkers documentation
		$ linter doc -help
		$ linter doc
		$ linter doc checkerName
```

Для некоторых подкоманд доступен флаг `-help`, который предоставляет дополнительную информацию (я вырезал некоторые слишком широкие строки):

```bash
./gocritic check -help
# Информация о всех доступных флагах.
```

<hr></spoiler>

# Документация установленных проверок

Ответ на вопрос "как узнать о том самом параметре skipNilEfaceLit?" - read the fancy manual (RTFM)!

Вся документация об установленных проверках находится внутри `mylinter`. Доступна эта документация через подкоманду `doc`:

```bash
# Выводит список всех установленных проверок:
$mylinter doc
panicNil [diagnostic]

# Выводит детальную документацию по запрашиваемой проверке:
$mylinter doc panicNil
panicNil checker documentation
URL: github.com/go-lintpack/lintpack
Tags: [diagnostic]

Detects panic(nil) calls.

Such panic calls are hard to handle during recover.

Non-compliant code:
panic(nil)

Compliant code:
panic("something meaningful")

Checker parameters:
  -@panicNil.skipNilEfaceLit bool
    	whether to ignore interface{}(nil) arguments (default false)
```

Подобно поддержке шаблонов в `go list -f`, вы можете передать строку шаблона, которая отвечает за формат вывода документации, что может быть полезным при составлении markdown документов.

# Где искать проверки для установки?

Для упрощения поиска полезных наборов проверок есть централизованный список `lintpack`-совместимых пакетов: https://go-lintpack.github.io/.

Вот некоторые из списка:
* [https://github.com/go-critic/go-critic/checkers](https://github.com/go-critic/go-critic/tree/master/checkers)
* https://github.com/go-critic/checkers-contrib

Этот список периодически обновляется и он открыт для заявок на добавление. Любой из этих пакетов может использоваться для создания линтера.

Команда ниже создаёт линтер, который содержит все проверки из списка выше:
```bash
# Сначала нужно убедиться, что исходные коды всех проверок
# доступны для Go компилятора.
go get -v github.com/go-critic/go-critic/checkers
go get -v github.com/go-critic/checkers-contrib

# build принимает список пакетов.
lintpack build \
  github.com/go-critic/go-critic/checkers \
  github.com/go-critic/checkers-contrib
```

`lintpack build` включает все проверки на этапе компиляции, получаемый линтер может быть размещён в окружении, где отсутствуют исходные коды реализации установленных диагностик, всё как обычно при статической линковке.

# Динамическое подключение пакетов

В дополнение к статической сборке есть возможность загружать плагины, предоставляющие дополнительные проверки.

Особенностью является то, что реализация чекера не знает, будут ли её использовать при статической компиляции или будут подгружать в виде плагина. Никаких изменений в коде не требуется.

Допустим, мы хотим добавить `panicNil` в линтер, но мы не имеем возможности пересобрать его из всех исходников, которые использовались при первой компиляции.

1. Создаём `linterPlugin.go`:

```go
package main

// Если требуется включить в плагин более одного набора проверок,
// просто добавьте требуемые import'ы.
import (
    _ "github.com/go-lintpack/lintpack/checkers"
)
```

2. Собираем динамическую библиотеку:

```bash
go build -buildmode=plugin -o linterPlugin.so linterPlugin.go
```

3. Запускаем линтер с параметром `-pluginPath`:

```bash
./linter check -pluginPath=linterPlugin.so bytes
```

> **Предупреждение:** Поддержка динамических модулей реализована через пакет [plugin](https://golang.org/pkg/plugin/), который не работает на Windows.

Флаг `-verbose` может помочь разобраться какая проверка включена или выключена, а, самое главное, там будет отображено какой из фильтров отключил проверку.

<spoiler title="Пример с -verbose"><hr>

Обратите внимание, что `panicNil` отображается в списке включенных проверок. Если мы уберём аргумент `-pluginPath`, это перестанет быть истиной.

```bash
./linter check -verbose -pluginPath=./linterPlugin.so bytes
	debug: appendCombine: disabled by tags (-disableTags)
	debug: boolExprSimplify: disabled by tags (-disableTags)
	debug: builtinShadow: disabled by tags (-disableTags)
	debug: commentedOutCode: disabled by tags (-disableTags)
	debug: deprecatedComment: disabled by tags (-disableTags)
	debug: docStub: disabled by tags (-disableTags)
	debug: emptyFallthrough: disabled by tags (-disableTags)
	debug: hugeParam: disabled by tags (-disableTags)
	debug: importShadow: disabled by tags (-disableTags)
	debug: indexAlloc: disabled by tags (-disableTags)
	debug: methodExprCall: disabled by tags (-disableTags)
	debug: nilValReturn: disabled by tags (-disableTags)
	debug: paramTypeCombine: disabled by tags (-disableTags)
	debug: rangeExprCopy: disabled by tags (-disableTags)
	debug: rangeValCopy: disabled by tags (-disableTags)
	debug: sloppyReassign: disabled by tags (-disableTags)
	debug: typeUnparen: disabled by tags (-disableTags)
	debug: unlabelStmt: disabled by tags (-disableTags)
	debug: wrapperFunc: disabled by tags (-disableTags)
	debug: appendAssign is enabled
	debug: assignOp is enabled
	debug: captLocal is enabled
	debug: caseOrder is enabled
	debug: defaultCaseOrder is enabled
	debug: dupArg is enabled
	debug: dupBranchBody is enabled
	debug: dupCase is enabled
	debug: dupSubExpr is enabled
	debug: elseif is enabled
	debug: flagDeref is enabled
	debug: ifElseChain is enabled
	debug: panicNil is enabled
	debug: regexpMust is enabled
	debug: singleCaseSwitch is enabled
	debug: sloppyLen is enabled
	debug: switchTrue is enabled
	debug: typeSwitchVar is enabled
	debug: underef is enabled
	debug: unlambda is enabled
	debug: unslice is enabled
# ... результат работы линтера.
```

<hr></spoiler>

# Сравнение с [gometalinter](https://github.com/alecthomas/gometalinter) и [golangci-lint](https://github.com/golangci/golangci-lint)

Во избежание путаницы, стоит описать основные различия между проектами.

[gometalinter](https://github.com/alecthomas/gometalinter) и [golangci-lint](https://github.com/golangci/golangci-lint) в первую очередь интегрируют другие, зачастую очень по-разному реализованные, линтеры, предоставляют к ним удобный доступ. Они нацелены на конечных пользователей, которые будут использовать статические анализаторы.

[lintpack](https://github.com/go-lintpack/lintpack) упрощает создание новых линтеров, предоставляет фреймворк, делающий разные пакеты, реализованные на его основе, совместимыми в пределах одного исполняемого файла. Эти проверки (для golangci-lint) или исполняемый файл (для gometalinter) далее могут быть встроены в вышеупомянутые мета-линтеры.

Допустим, какая-то из `lintpack`-совместимых проверок является частью `golangci-lint`. Если существует какая-то проблема, связанная с удобством её использования - это может быть зоной ответственности `golangci-lint`, но если речь идёт об ошибке в реализации самой проверки, то это проблема авторов проверки, lintpack экосистемы.

Иными словами, эти проекты решают разные проблемы.

# А что там с go-critic?

Процесс портирования `go-critic` на `lintpack` уже завершён, чекеры можно найти в репозитории [go-critic/checkers](https://github.com/go-critic/go-critic/tree/master/checkers). 

```bash
# Установка go-critic до:
go get -v github.com/go-critic/go-critic/...

# Установка go-critic после:
lintpack -o gocritic github.com/go-critic/go-critic/checkers
```

Большого смысла использовать `go-critic` вне `golangci-lint` нет, а вот `lintpack` может позволить установить те проверки, которые не входят в набор `go-critic`. Например, это могут быть диагностики, написанные вами.

# Продолжение следует

Как создавать свои `lintpack`-совместимые проверки вы узнаете в следующей статье.

Там же мы разберём какие преимущества вы получаете при реализации своего линтера на основе `lintpack` по сравнению с реализацией с чистого листа.

Надеюсь, у вас появился аппетит к новым проверкам для Go. Дайте знать, как статического анализа станет слишком много, будем оперативно решать эту проблему вместе.
