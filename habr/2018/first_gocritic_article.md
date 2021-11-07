# go-critic: самый упрямый статический анализатор для Go

![](https://habrastorage.org/webt/qo/je/x2/qojex27wlzk161udn-fmhxwsnzc.png)

Анонсируем новый линтер (статический анализатор) для [Go](https://golang.org/), который одновременно является песочницей для прототипирования ваших задумок в мире статического анализа.

[go-critic](https://github.com/go-critic/go-critic) построен вокруг следующих наблюдений:
* Лучше иметь “good enough” реализацию проверки, чем не иметь её вовсе
* Если проверка спорная, это ещё не значит, что она не может быть полезна. Помечаем как “opinionated” и вливаем
* Писать линтер с нуля, как правило, сложнее, чем добавлять новую проверку в существующий каркас, если сам фреймворк прост для понимания

В этом посте мы рассмотрим использование и архитектуру go-critic, некоторые [реализованные в нём проверки](https://go-critic.github.io/overview.html), а также опишем основные шаги добавления своей функции-анализатора в него.

<cut/>

# Быстрый старт

```bash
$ cd $GOPATH
$ go get -u github.com/go-critic/go-critic/...
$ ./bin/gocritic check-package strings

$GOROOT/src/strings/replace.go:450:22: unslice: 
  could simplify s[:] to s
$GOROOT/src/strings/replace.go:148:2: elseif: 
  should rewrite if-else to switch statement
$GOROOT/src/strings/replace.go:156:3: elseif: 
  should rewrite if-else to switch statement
$GOROOT/src/strings/replace.go:219:3: elseif:
  should rewrite if-else to switch statement
$GOROOT/src/strings/replace.go:370:1: paramTypeCombine:
  func(pattern string, value string) *singleStringReplacer
  could be replaced with
  func(pattern, value string) *singleStringReplacer
$GOROOT/src/strings/replace.go:259:2: rangeExprCopy:
  copy of r.mapping (256 bytes) can be avoided with &r.mapping
$GOROOT/src/strings/replace.go:264:2: rangeExprCopy:
  copy of r.mapping (256 bytes) can be avoided with &r.mapping
$GOROOT/src/strings/strings.go:791:1: paramTypeCombine:
  func(s string, cutset string) string
  could be replaced with
  func(s, cutset string) string
$GOROOT/src/strings/strings.go:800:1: paramTypeCombine:
  func(s string, cutset string) string
  could be replaced with
  func(s, cutset string) string
$GOROOT/src/strings/strings.go:809:1: paramTypeCombine:
  func(s string, cutset string) string
  could be replaced with
  func(s, cutset string) string
$GOROOT/src/strings/strings.go:44:1: unnamedResult: 
  consider to give name to results
$GOROOT/src/strings/strings.go:61:1: unnamedResult:
  consider to give name to results
$GOROOT/src/strings/export_test.go:28:3: rangeExprCopy:
  copy of r.mapping (256 bytes) can be avoided with &r.mapping
$GOROOT/src/strings/export_test.go:42:1: unnamedResult:
  consider to give name to results
```
(Форматирование предупреждений отредактировано, оригиналы доступны в [gist'е](https://gist.github.com/Quasilyte/22db23699108a0a04ce7544a552c6caa).)

Утилита [gocritic](https://github.com/go-critic/go-critic/tree/master/cmd/gocritic) умеет проверять отдельные пакеты по их import пути (`check-package`), а так же рекурсивно обходить все директории (`check-project`). Например, вы можете проверить весь `$GOROOT` или `$GOPATH` с помощью одной команды:

```bash
$ gocritic check-project $GOROOT/src
$ gocritic check-project $GOPATH/src
```

Есть поддержка “белого списка” для проверок, чтобы явно перечислить какие проверки нужно выполнять (флаг `-enable`). По умолчанию запускаются все те проверки, которые не отмечены значком `Experimental` или `VeryOpinionated`.

Планируются интеграции в [golangci-lint](https://github.com/golangci/golangci-lint) и [gometalinter](https://github.com/alecthomas/gometalinter).

# Как всё начиналось

Проводя очередное код-ревью Go проекта, или при аудите некоторой 3-rd party библиотеки, вы можете раз за разом замечать одни и те же проблемы.

К вашему сожалению, найти линтер, который диагностировал бы этот класс проблем, не удалось.

Первым вашим действием может быть попытка категоризировать проблему и связаться с авторами существующих линтеров, предлагая им добавить новую проверку. Шансы на то, что ваше предложение примут, сильно зависит от проекта и могут быть довольно низкими. Далее, скорее всего, последуют месяцы ожидания.

А что, если проверка и вовсе неоднозначная и может быть кем-то воспринята как слишком субъективная или недостаточно точная?

Может, есть смысл попробовать написать эту проверку самому?

`go-critic` существует для того, чтобы стать домом для экспериментальных проверок, которые проще реализовать самим, чем пристроить их в существующие статические анализаторы. Само устройство `go-critic` минимизирует количество контекста и действий, которые необходимы для добавления новой проверки - можно сказать, что потребуется добавить только один файл (не считая тестов).

# Как устроен go-critic

Критик - это набор **правил** (rules), которые описывают свойства проверки и **микро-линтеры** (checkers), реализующие инспекцию кода на соответствие правилу.

Приложение, которое встраивает линтер (например, [cmd/gocritic](https://github.com/go-critic/go-critic/tree/master/cmd/gocritic) или [golangci-lint](https://github.com/golangci/golangci-lint)), получает список поддерживаемых правил, фильтрует их определённых образом, создаёт для каждого отобранного правила check функцию, и запускает каждую из них над исследуемым пакетом.

Работа по добавлению нового checker’а сводится к трём основным шагам:
1. Добавление тестов.
2. Реализация непосредственно самой проверки.
3. Добавление документации для линтера.

Пройдём все эти пункты на примере правила [captLocal](https://go-critic.github.io/overview.html#captLocal-ref), которое требует отсутствия локальных имён, начинающихся с заглавной буквы.

![](https://habrastorage.org/webt/t9/a_/0-/t9a_0-i2mt6zy1dlzixhb6dkcc8.gif)

# Добавление тестов

Для добавления тестовых данных для новой проверки, нужно создать новую директорию в [lint/testdata](https://github.com/go-critic/go-critic/tree/master/lint/testdata/captLocal).

Каждая такая директория должна иметь файл [positive_tests.go](https://github.com/go-critic/go-critic/blob/master/lint/testdata/captLocal/positive_tests.go), который описывает примеры кода, на которых проверка должна срабатывать. Чтобы тестировать отсутствие ложных срабатываний, тесты дополняются “корректным” кодом, в котором новая проверка не должна находить никаких проблем ([negative_tests.go](https://github.com/go-critic/go-critic/blob/master/lint/testdata/captLocal/negative_tests.go)).

**Примеры:**

```go
// lint/testdata/positive_tests.go

/// consider `in' name instead of `IN'
/// `X' should not be capitalized
/// `Y' should not be capitalized
/// `Z' should not be capitalized
func badFunc1(IN, X int) (Y, Z int) {
	/// `V' should not be capitalized
	V := 1
	return V, 0
}
```

```go
// lint/testdata/negative_tests.go

func goodFunc1(in, x int) (x, y int) {
	v := 1
	return v, 0
}
```

Запустить тесты можно после добавления нового линтера.

# Реализация проверки

Создадим файл с названием чекера: `lint/captLocal_checker.go`.
По конвенции, все файлы с микро-линтерами имеют суффикс `_checker`.

```go
package lint

// Суффикс “Checker” в имени типа обязателен.
type captLocalChecker struct {
	checkerBase
	upcaseNames map[string]bool
}
```

[checkerBase](https://github.com/go-critic/go-critic/blob/master/lint/checker_base.go) - это тип, который должен быть встроен (embedded) в каждый checker.
Он предоставляет реализации по умолчанию, что позволяет писать меньше кода в каждом линтере.
Помимо прочего, checkerBase включает указатель на `lint.context`, который содержит информацию о типах и другие метаданные о проверяемом файле.

Поле `upcaseNames` будет содержать таблицу известных имён, которые мы будем предлагать заменять на `strings.ToLower(name)` версию. Для тех имён, которые не содержатся в мапе, будет предлагаться не использовать заглавную букву, но корректной замены предоставляться не будет.

Внутреннее состояние инициализируется единожды для каждого экземпляра.
Метод `Init()` требуется определять только для тех линтеров, которым нужно провести предварительную инициализацию.

```go
func (c *captLocalChecker) Init() {
	c.upcaseNames = map[string]bool{
		"IN":    true,
		"OUT":   true,
		"INOUT": true,
	}
}
```

Теперь требуется определить саму проверяющую функцию.
В случае `captLocal`, нам нужно проверять все локальные `ast.Ident`, которые вводят новые переменные.

Для того, чтобы проверять все локальные определения имён, следует в своём чекере реализовать метод со следующей сигнатурой:

```go
VisitLocalDef(name astwalk.Name, initializer ast.Expr)
```

Список доступных visitor интерфейсов может быть найден в файле [lint/internal/visitor.go](https://github.com/go-critic/go-critic/blob/master/lint/internal/astwalk/visitor.go).
`captLocal` реализует `LocalDefVisitor`.

```go
// Игнорируем ast.Expr, потому что нам не интересно какое значение присваивается
// локальному имени. Нас интересуют только сами названия переменных.
func (c *captLocalChecker) VisitLocalDef(name astwalk.Name, _ ast.Expr) {
	switch {
	case c.upcaseNames[name.ID.String()]:
		c.warnUpcase(name.ID)
	case ast.IsExported(name.ID.String()):
		c.warnCapitalized(name.ID)
	}
}

func (c *captLocalChecker) warnUpcase(id *ast.Ident) {
	c.ctx.Warn(id, "consider `%s' name instead of `%s'", strings.ToLower(id.Name), id)
}

func (c *captLocalChecker) warnCapitalized(id ast.Node) {
	c.ctx.Warn(id, "`%s' should not be capitalized", id)
}
```

По конвенции, методы, которые генерируют предупреждения, обычно выносятся в отдельные методы. Есть редкие исключения, но следование этому правилу считается хорошей практикой.

# Добавление документации

Ещё одним необходимым к реализации методом является `InitDocumentation`:

```go
func (c *captLocalChecker) InitDocumentation(d *Documentation) {
	d.Summary = "Detects capitalized names for local variables"
	d.Before = `func f(IN int, OUT *int) (ERR error) {}`
	d.After = `func f(in int, out *int) (err error) {}`
}
```

Обычно, достаточно заполнить 3 поля:

* `Summary` - описание действия проверки в одно предложение.
* `Before` - код до исправления.
* `After` - код после исправления (не должен вызывать предупреждения).

<spoiler title="Генерация документации">
Повторно генерировать документацию не является обязательным требованием для нового линтера, возможно в ближайшем будущем этот шаг будет полностью автоматизирован. Но если вы всё же хотите проверить, как будет выглядеть выходной markdown файл, воспользуйтесь командой `make docs`. Файл `docs/overview.md` будет обновлён.
</spoiler>

# Регистрация нового линтера и запуск тестов

Последним штрихом является регистрация нового линтера:

```go
// Локальная для captLocal_checker.go init функция.
func init() {
	addChecker(&captLocalChecker{}, attrExperimental, attrSyntaxOnly)
}
```

`addChecker` ожидает указатель на zero-value нового линтера. Далее идёт вариадичный аргумент, позволяющий передать ноль или несколько аттрибутов, описывающих свойства реализации правила.

`attrSyntaxOnly` - опциональный маркер для линтеров, которые не пользуются информацией о типах в своей реализации, что позволяет запускать их без выполнения проверки на типы. `golangci-lint` отмечает такие линтеры флагом “fast” (потому что они выполняются гораздо быстрее).

`attrExperimental` - атрибут, присваиваемый всем новым реализациям. Удаление этого атрибута возможно только после стабилизации реализуемой проверки.

Теперь, когда новый линтер зарегистрирован через addChecker, можно запустить тесты:

```bash
# Из GOPATH:
$ go test -v github.com/go-critic/go-critic/lint
# Из GOPATH/src/github.com/go-critic/go-critic:
$ go test -v ./lint
# Оттуда же, можно запустить тесты с помощью make:
$ make test
```

# Optimistic merging (почти)

При рассматривании pull requests мы стараемся придерживаться стратегии [optimistic merging](https://rfc.zeromq.org/spec:42/C4/). В основном это выражается в принятии тех PR, к которым у проводящего ревью могут оставаться некоторые, в частности чисто субъективные, претензии. Сразу после вливания такого патча может последовать PR от ревьювера, который исправляет эти недочёты, в CC (copy) добавляется автор исходного патча.

У нас также есть два маркера линтеров, которые могут быть использованы во избежание красных флагов в случае отсутствия полного консенсуса:

1. `Experimental`: реализация может иметь большое количество false positive, быть неэффективной (при этом источник проблемы идентифицирован) или “падать” в некоторых ситуациях. Такую реализацию влить можно, если пометить её атрибутом `attrExperimental`. Иногда с помощью experimental обозначаются те проверки, которым не получилось подобрать хорошее название с первого коммита.
2. `VeryOpinionated`: если проверка может иметь как защитников, так и врагов, стоит пометить её атрибутом `attrVeryOpinionated`. Таким образом мы можем избегать отклонения тех идей о стиле кода, которые могут не совпадать со вкусом некоторых гоферов.

`Experimental` - потенциально временное и исправимое свойство реализации. `VeryOpinionated` - это более фундаментальное свойство правила, которое не зависит от реализации.

Рекомендуется создавать `[checker-request]` тикет на [github’е](https://github.com/go-critic/go-critic/issues) перед тем, как отправлять реализацию, но если вы уже отправили pull request, открыть соответствующий issue могут за вас.

Подробнее о деталях процесса разработки смотри в [CONTRIBUTING.md](https://github.com/go-critic/go-critic/blob/master/CONTRIBUTING.md).
Основные правила перечислены в секции [main rules](https://github.com/go-critic/go-critic/blob/master/CONTRIBUTING.md#code-review-main-rules).

# Напутствия

Вы можете участвовать в проекте не только добавлением новых линтеров.  
Есть множество других путей:

* Пробовать его на своих проектах или крупных/известных open-source проектах и докладывать false positives, false negatives и другие недоработки. Будем признательны, если вы также добавите заметку о найденной/исправленной проблеме на страницу [trophies](https://go-critic.github.io/trophies.html).
* Предлагать идеи для новых проверок. Достаточно создать [issue](https://github.com/go-critic/go-critic/issues/new) на нашем трекере.
* Добавлять тесты для существующих линтеров.

`go-critic` критикует ваш Go код голосами всех программистов, участвующих в его разработке. Критиковать может каждый, поэтому - присоединяйтесь!

![](https://habrastorage.org/webt/kd/oc/qy/kdocqyc1jjv48xggojdca62tmqg.png)
