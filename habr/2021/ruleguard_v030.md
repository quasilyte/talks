# Релиз ruleguard v0.3.0

А что, если я скажу вам, что линтеры для Go можно создавать вот таким декларативным способом?

```go
func alwaysTrue(m dsl.Matcher) {
    m.Match(`strings.Count($_, $_) >= 0`).Report(`always evaluates to true`)
    m.Match(`bytes.Count($_, $_) >= 0`).Report(`always evaluates to true`)
}

func replaceAll() {
    m.Match(`strings.Replace($s, $d, $w, $n)`).
        Where(m["n"].Value.Int() <= 0).
        Suggest(`strings.ReplaceAll($s, $d, $w)`)
}
```

Год назад я уже рассказывал об утилите [ruleguard](https://github.com/quasilyte/go-ruleguard). Сегодня хотелось бы поделиться тем, что нового появилось за это время.

Основные нововведения:

* Поддержка установки наборов правил через [Go модули](https://github.com/golang/go/wiki/Modules) (bundles)
* Программируемые фильтры (компилируются в байт-код)
* Добавлен режим отладки фильтров
* Появился хороший обучающий материал: [ruleguard by example](https://go-ruleguard.github.io/by-example/)
* У проекта появились [реальные пользователи](https://github.com/grafana/grafana/pull/28419) и внешние [наборы правил](https://github.com/dgryski/semgrep-go)
* [Онлайн песочница](https://go-ruleguard.github.io/play/), позволяющая попробовать ruleguard прямо в браузере

<img title="Автор иллюстрации: Татьяна Уфимцева @leased_line" src="https://habrastorage.org/webt/jb/iy/a0/jbiya0ab6njechtwp9dwtxhmw24.jpeg">

<cut/>

## Небольшое введение

ruleguard — это платформа для запуска динамических диагностик. Что-то вроде интерпретатора для скриптов, специализирующихся на статическом анализе.

Вы описываете на DSL свой набор правил (или используете уже готовые наборы) и запускаете их через утилиту ruleguard.

<img src="https://habrastorage.org/webt/yy/fj/wk/yyfjwkkjfg6tm_8hekskud3c63o.jpeg">
<img src="https://habrastorage.org/webt/4c/r3/7x/4cr37xaaochgqrqogyvne2vud4s.png">

<br>

Эти правила **интерпретируются** во время работы, поэтому нет необходимости повторно собирать анализатор каждый раз, когда вы добавляете новые диагностики. Это особенно важно, если мы рассматриваем интеграцию с [golangci-lint](https://github.com/golangci/golangci-lint). Было бы очень неудобно перекомпилировать golangci-lint при желании использовать свой набор правил.

Если называть наиболее близкие к этой концепции проекты, то в голову приходят CodeQL и Semgrep. Некоторое время назад я проводил [сравнение](https://speakerdeck.com/quasilyte/ruleguard-vs-semgrep-vs-codeql), хотя часть информации из того доклада уже устарела (все проекты получают новые фичи).

Работаю над проектом я в свободное время, когда появляется настроение, поэтому результаты за год могут показаться не такими впечатляющими. Тем не менее проект развивается.

Большая часть нововведений адресует какую-то конкретную проблему, отсюда и формат заголовков.

<spoiler title="Терминология, используемая в статье"><hr>

Поскольку я иногда использую специфичную для проекта терминологию, приведу здесь несколько расшифровок.

| En | Ru | Значение |
|---|---|---|
| Rule | Правило | AST-шаблон, совмещённый с фильтрами и ассоциированными действиями (чаще всего — создание предупреждения). |
| Rules group | Группа правил | Именованный набор правил. Мы могли бы называть группы "диагностиками", как это делается в других линтерах, но группа не обязана выполнять единственную проверку. |
| Rule set | Набор правил | Совокупность групп правил. |
| Rule bundle | Бандл (извините) | Набор правил, оформленный как Go модуль, доступный для импортирования в другие наборы правил. |
| Module | Модуль | Модули Go; каждый бандл — модуль, но сами модули к бандлам не имеют никакого отношения. |

Если по мере прочтения статьи вы нашли совершенно непонятный для вас термин, стоит сообщить об этом, возможно он будет добавлен в эту таблицу.

<hr></spoiler>

## Проблема: переиспользование наборов правил

Раньше всё было относительно просто: есть файл с правилами, утилита принимает его на вход и применяет его к проверяемой кодовой базе.

Далее мы понимаем, что хранить всё в одном файле не очень удобно, и я добавляю поддержку множественных файлов правил.

Затем появился хороший набор правил, написанный [Damian Gryski](https://twitter.com/dgryski). Единственный способ его использовать на своих проектах — это копировать в свой репозиторий.

У этого подхода с полным копированием есть преимущество: всё лишнее можно удалить, а свои правила добавлять в этот же файл. Но это не самый частый сценарий использования. Как оказалось, чаще люди хотят взять уже готовый набор правил и запускать его с минимальными усилиями.

Новый механизм бандлов для правил позволит решить сразу несколько проблем:

* Установка бандлов через go get
* Версионирование с помощью Go модулей: удобно делать релизы и закреплять версию
* Культура оформления правил в модули упрощает тестирование

Всё это возможно благодаря тому, что ruleguard файлы, в которых пишутся правила — это обычный Go код (по этой же причине мы имеем нормальный autocomplete и поддержку редакторов).

Вот так выглядит простейший файл правил, который использует упомянутые выше правила, а также определяет парочку своих:

```go
package gorules

import (
    "github.com/quasilyte/go-ruleguard/dsl"
    damianrules "github.com/dgryski/semgrep-go"
)

func init() {
    // Импорт всех правил, без префикса.
    dsl.ImportRules("", damianrules.Bundle)
}

func emptyStringTest(m dsl.Matcher) {
    m.Match(`len($s) == 0`).
        Where(m["s"].Type.Is("string")).
        Report(`maybe use $s == "" instead?`)

    m.Match(`len($s) != 0`).
        Where(m["s"].Type.Is("string")).
        Report(`maybe use $s != "" instead?`)
}
```

Если требуется выключить некоторые импортируемые правила, делается это через командную строку параметром `--disable`.

## Проблема: недостаточная выразительность DSL

[dsl.Matcher](https://pkg.go.dev/github.com/quasilyte/go-ruleguard/dsl#Matcher) предоставляет несколько фильтров, которые часто нужны в типичных для ruleguard правилах.

Но бывают моменты, когда требуется создать довольно сложное условие или фильтр, имеющий промежуточные результаты. В этой ситуации можно использовать новый метод [Filter()](https://pkg.go.dev/github.com/quasilyte/go-ruleguard/dsl#Var.Filter), который принимает Go функцию-предикат в качестве аргумента. Эта функция будет вызываться во время применения фильтра.

```go
package gorules

import (
    "github.com/quasilyte/go-ruleguard/dsl"
    "github.com/quasilyte/go-ruleguard/dsl/types"
)

// implementsStringer является пользовательским фильтром.
// Этот фильтр проверяет, реализуют ли T или *T интерфейс `fmt.Stringer`.
func implementsStringer(ctx *dsl.VarFilterContext) bool {
    stringer := ctx.GetInterface(`fmt.Stringer`)
    return types.Implements(ctx.Type, stringer) ||
        types.Implements(types.NewPointer(ctx.Type), stringer)
}

func sprintStringer(m dsl.Matcher) {
    // Если бы мы использовали m["x"].Type.Implements(`fmt.Stringer`), тогда
    // мы бы не получили все желаемые результаты: если тип $x реализует
    // fmt.Stringer как *T, то значения типа T не будут считаться реализациями.
    // Наш кастомный фильтр примеряет обе версии: с указателем и без укатателя.
    m.Match(`fmt.Sprint($x)`).
        Where(m["x"].Filter(implementsStringer) && m["x"].Addressable).
        Report(`can use $x.String() directly`)
}
```

Запускать эти правила будем на следующем файле:

```go
package main

import "fmt"

func main() {
    fooPtr := &Foo{}
    foo := Foo{}

    println(fmt.Sprint(foo))
    println(fmt.Sprint(fooPtr))

    println(fmt.Sprint(0))    // Не fmt.Stringer
    println(fmt.Sprint(&foo)) // Отбрасывается условием addressable
}

type Foo struct{}

func (*Foo) String() string { return "Foo" }
```

Результат запуска:

```
$ ruleguard -rules=rules.go main.go
main.go:9:10: can use foo.String() directly
main.go:10:10: can use fooPtr.String() directly
```

Флаг `--debug-filter позволяет посмотреть, во что скомпилировался выбранный фильтр:

<img src="https://habrastorage.org/webt/t2/td/c3/t2tdc3gmpwnumqbairmh1y91y6s.png">

На данный момент байт-код компилятор не выполняет никаких оптимизаций генерируемого кода, но даже в текущем виде производительность в несколько раз выше, чем при использовании [yaegi](https://github.com/traefik/yaegi).

## Проблема: правила сложно отлаживать

Поскольку в [Where()](https://pkg.go.dev/github.com/quasilyte/go-ruleguard/dsl#Matcher.Where) может использоваться довольно сложное выражение, не всегда понятно, почему правило не срабатывает на анализируемых фрагментах кода.

На помощь приходит новый флаг `--debug-group`, включающий детальную информацию о неуспешно выполнившихся фильтрах для выбранной группы правил.

Допустим, вы описали следующее правило:

```go
func offBy1(m dsl.Matcher) {
    m.Match(`$s[len($s)]`).
        Where(m["s"].Type.Is(`[]$elem`) && m["s"].Pure).
        Report(`index expr always panics; maybe you wanted $s[len($s)-1]?`)
}
```

И запустили его на следующем файле:

```go
func lastByte(s string) byte {
    return s[len(s)]
}

func f() byte {
    return randString()[len(randString())]
}
```

И не получили ни одного предупреждения… Давайте попробуем включить отладочную печать.

```
$ ruleguard -rules=rules.go -debug-group offBy1 test.go
test.go:6: [rules.go:6] rejected by m["s"].Type.Is(`[]$elem`)
  $s string: s
test.go:10: [rules.go:6] rejected by m["s"].Pure
  $s []byte: randBytes()
```

Мы видим конкретное выражение из `Where()`, которое не дало сработать правилу. Мы также видим все захваченные Go выражения в именованных частях AST шаблона (в данном случае это $s), а также их тип.

В первом случае условие типа `[]$elem` требует произвольного слайса, а в коде — строка. Во втором случае правило не срабатывает из-за вызова функции (нарушается условие pure).

Скорее всего, мы не хотим убирать условие на чистоту выражений, а вот добавить тип string в диагностику можно:

```diff
- Where(m["s"].Type.Is(`[]$elem`) && m["s"].Pure).
+ Where((m["s"].Type.Is(`[]$elem`) || m["s"].Type.Is(`string`)) && m["s"].Pure).
```

Повторный запуск с обновлённой версией найдёт ошибку в индексировании строки:

```
test.go:6:9: offBy1: index expr always panics; maybe you wanted s[len(s)-1]?
```

## Проблема: трудности изучения DSL

Когда у вас на руках только документация, которая зачастую направляет вас читать исходные коды, то освоение технологии будет требовать многих усилий.

Мне нравится подход [Go by Example](https://gobyexample.com/). В нём введение производится через набор примеров с пояснениями, от простого к более продвинутому. Это полезно как начинающим, так и продолжающим.

[ruleguard by example](https://go-ruleguard.github.io/by-example/) написан в таком же стиле. Он позволяет достаточно быстро получить все необходимые знания в наглядной форме.

<img src="https://habrastorage.org/webt/uw/mt/en/uwmtenp4kuw9-mdit3imh4nht8e.png">

## Как начать использовать ruleguard?

> Внимание! Лучше всего ruleguard работает с проектами, которые используют Go модули.

Лучше всего дождаться момента, когда в golangci-lint появится новая версия.

Однако, если вы не используете golangci-lint или хотите попробовать уже сегодня, то можно скачать бинарник ruleguard со [страницы релиза](https://github.com/quasilyte/go-ruleguard/releases).

Вам также понадобится набор правил. Здесь есть как минимум два варианта: использовать минималистичный набор [github.com/quasilyte/go-ruleguard/rules](https://github.com/quasilyte/go-ruleguard/tree/master/rules) или более обширный [github.com/dgryski/semgrep-go](https://github.com/dgryski/semgrep-go). Вы также можете импортировать оба этих бандла или не импортировать ничего и использовать лишь свои наработки.

Допустим, вы выбрали `github.com/quasilyte/go-ruleguard/rules`, тогда:

1. Скачиваем ruleguard для своей платформы (или собираем из исходников)
2. Выполняем `go get -v github.com/quasilyte/go-ruleguard/dsl` внутри модуля вашего проекта
3. Выполняем `go get -v github.com/quasilyte/go-ruleguard/rules` внутри модуля вашего проекта
4. Создаём свой файл правил `rules.go`, импортируем там установленный бандл
5. Запускаем ruleguard с параметром `-rules=rules.go` на вашем проекте

```bash
$ ruleguard -rules=rules.go ./...
```

Если у вас возникают проблемы с запуском или установкой ruleguard, [сообщите об этом](https://github.com/quasilyte/go-ruleguard/issues/new).

## Создаём свой бандл

Есть только два требования:

1. Бандл должен быть отдельным Go модулем
2. Пакет должен определять экспортируемую переменную Bundle

Временным ограничением является то, что бандл не может импортировать другой бандл.

В бандле может быть несколько Go файлов, каждый из которых будет содержать правила. При импортировании бандла будут подключаться все файлы, как и в случае обычных Go пакетов.

```go
package gorules

import "github.com/quasilyte/go-ruleguard/dsl"

// Bundle содержит метаданные о наборе правил.
var Bundle = dsl.Bundle{}

func boolComparison(m dsl.Matcher) {
    m.Match(`$x == true`,
        `$x != true`,
        `$x == false`,
        `$x != false`).
        Report(`omit bool literal in expression`)
}
```

В качестве примера, можно посмотреть на репозиторий [ruleguard-rules-test](https://github.com/quasilyte/ruleguard-rules-test).

## Тестируем свой бандл

Тестирование основано на фреймворке [go/analysis](https://godoc.org/golang.org/x/tools/go/analysis) и вспомогательном пакете [analysistest](https://godoc.org/golang.org/x/tools/go/analysis/analysistest).

Рядом с модулем создаётся директория `testdata`, куда складываются Go файлы, на которых будут запускаться ваши диагностики.

Для запуска тестов нужно написать некоторый шаблонный код:

```go
// file rules_test.go

package gorules_test

import (
    "testing"

    "github.com/quasilyte/go-ruleguard/analyzer"
    "golang.org/x/tools/go/analysis/analysistest"
)

func TestRules(t *testing.T) {
    // Если у вас несколько файлов с правилами, то вместо "rules.go"
    // нужно указать имена всех файлов через запятую, например: "style.go,perf.go".
    if err := analyzer.Analyzer.Flags.Set("rules", "rules.go"); err != nil {
        t.Fatalf("set rules flag: %v", err)
    }
    analysistest.Run(t, analysistest.TestData(), analyzer.Analyzer, "./...")
}
```

Структура бандла будет выглядеть примерно так:

```
mybundle/
  go.mod        -- файл, создаваемый "go mod init"
  rules.go      -- здесь ваши правила (можно назвать файл иначе)
  rules_test.go -- запускатель тестов
  testdata/     -- файлы, на которых будем запускать анализ
    target1.go
    target2.go
    ...
```

Тестовые файлы будут содержать магические комментарии:

```go
// file testdata/target1.go

package test

func f(cond bool) {
    if cond == true { // want `omit bool literal in expression`
    }
}
```

После want идёт регулярное выражение, которое должно матчить выдаваемое предупреждение. Могу рекомендовать использовать `\Q` в начале, чтобы не приходилось ничего экранировать.

Тест запускается обычным `go test` из директории бандла.

## Ссылки и дополнительные материалы

<img src="https://habrastorage.org/webt/ni/hn/rh/nihnrhigipwqd1z4ulqf032owo4.png">

* [Список похожих проектов](https://github.com/quasilyte/go-ruleguard/issues/36)
* [Сайт проекта ruleguard](https://go-ruleguard.github.io)
* [Телеграм чатик, где обсуждается go-critic и ruleguard](https://t.me/go_critic_ru)
* [Использование ruleguard из golangci-lint](https://quasilyte.dev/blog/post/ruleguard/#using-from-the-golangci-lint)
* [DSL мануал](https://github.com/quasilyte/go-ruleguard/blob/master/_docs/dsl.md)
* [go-critic](https://github.com/go-critic/go-critic) — линтер, в который встроен ruleguard
* [Введение в бандлы](https://quasilyte.dev/blog/post/ruleguard-modules)
