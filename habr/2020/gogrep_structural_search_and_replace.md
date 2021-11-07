# gogrep: структурный поиск и замена Go кода

[gogrep](https://github.com/mvdan/gogrep) - это одна из моих любимых утилит для работы с Go. Она позволяет находить код по синтаксическим шаблонам, фильтровать результаты по типам выражений, а также выполнять замену (тоже по шаблону).

В этой заметке я расскажу как использовать `gogrep`, а также о [VS Code расширении](https://marketplace.visualstudio.com/items?itemName=quasilyte.gogrep) для более удобной работы с `gogrep` прямо из редактора.

![](https://habrastorage.org/webt/l7/s7/gc/l7s7gccru4deae3xyln5scwfof8.jpeg)

<cut/>

## Зачем нужен gogrep

Если в тезисах, то `gogrep` может быть полезен при:

* Рефакторинге
* Изучении кодовой базы
* Поиске подозрительного кода (пример: [ruleguard](https://habr.com/ru/post/481696/))

Рассмотрим пример, который демонстрирует изящность и эффективность структурного поиска.

Функции `a()` и `b()` выполняют одинаковые операции:

```go
func a(xs []int) []int {
  xs = append(xs, 1)
  xs = append(xs, 2)
  return xs
}

func b(xs []int) []int {
  xs = append(xs, 1, 2)
  return xs
}
```

Допустим, мы хотим переписать все места, где вызовы `append` можно схлопнуть.

Попробуем `gogrep`:
* Находим все подходящие пары с помощью `-x` шаблона `$x=append($x,$a); $x=append($x,$b)`
* Через `-s` шаблон `$x=append($x,$a,$b)` получаем искомую замену
* Передавая аргумент `-w` все затронутые файлы будут обновлены.

```bash
gogrep -w -x '$x=append($x,$a);$x=append($x,$b)' -s '$x=append($x,$a,$b)' ./...
```

Если поставить [расширение для VS Code](https://marketplace.visualstudio.com/items?itemName=quasilyte.gogrep), то становится ещё проще.

Вот пример замены `+=1` на `++`:

![](https://habrastorage.org/webt/bc/pq/1x/bcpq1xmkv2fq3annn9oo0lu9t6i.gif)

Пример из реальной жизни: как-то захотел выполнить замену `slice[:] -> slice`. Даже заводил [issue в staticcheck](https://github.com/dominikh/go-tools/issues/282). Специфика в том, что нельзя просто искать `[:]`, потому что брать такой слайс от массива имеет смысл, а вот от строки или слайса - нет.

Вот пример того, как можно найти лишние слайсы от `[]byte` в stdlib:

```bash
# Только поиск.
gogrep -x '$s[:]' -a 'type([]byte)' std

# Поиск+замена.
gogrep -x '$s[:]' -a 'type([]byte)' -s '$s' -w std
```

<spoiler title="Если интересно, что найдёт этот запуск"><hr>

Показываю только первые 30 результатов (всего их 300+):

```
$GOROOT/src/archive/tar/format.go:163:59: b[:]
$GOROOT/src/archive/tar/reader.go:345:33: tr.blk[:]
$GOROOT/src/archive/tar/reader.go:348:17: tr.blk[:]
$GOROOT/src/archive/tar/reader.go:348:28: zeroBlock[:]
$GOROOT/src/archive/tar/reader.go:349:34: tr.blk[:]
$GOROOT/src/archive/tar/reader.go:352:18: tr.blk[:]
$GOROOT/src/archive/tar/reader.go:352:29: zeroBlock[:]
$GOROOT/src/archive/tar/reader.go:396:23: tr.blk[:]
$GOROOT/src/archive/tar/reader.go:497:36: blk[:]
$GOROOT/src/archive/tar/reader.go:528:33: blk[:]
$GOROOT/src/archive/tar/reader.go:531:14: blk[:]
$GOROOT/src/archive/tar/writer.go:392:26: blk[:]
$GOROOT/src/archive/tar/writer.go:477:23: zeroBlock[:]
$GOROOT/src/archive/zip/reader.go:233:29: buf[:]
$GOROOT/src/archive/zip/reader.go:236:15: buf[:]
$GOROOT/src/archive/zip/reader.go:251:30: buf[:]
$GOROOT/src/archive/zip/reader.go:254:15: buf[:]
$GOROOT/src/archive/zip/writer.go:92:17: buf[:]
$GOROOT/src/archive/zip/writer.go:110:19: buf[:]
$GOROOT/src/archive/zip/writer.go:116:30: buf[:]
$GOROOT/src/archive/zip/writer.go:132:27: buf[:]
$GOROOT/src/archive/zip/writer.go:157:17: buf[:]
$GOROOT/src/archive/zip/writer.go:177:27: buf[:]
$GOROOT/src/archive/zip/writer.go:190:16: buf[:]
$GOROOT/src/archive/zip/writer.go:198:26: buf[:]
$GOROOT/src/archive/zip/writer.go:314:18: mbuf[:]
$GOROOT/src/archive/zip/writer.go:319:31: mbuf[:]
$GOROOT/src/archive/zip/writer.go:386:16: buf[:]
$GOROOT/src/archive/zip/writer.go:398:23: buf[:]
$GOROOT/src/bytes/bytes.go:172:24: b[:]
```

<hr></spoiler>

## Поисковые шаблоны

Поисковой шаблон - это небольшой фрагмент Go кода, который может включать в себя $-выражения (мы будем называть их "переменными шаблона"). Шаблон может быть выражением, statement (или их списком) или декларацией.

Переменные шаблона - это Go переменные с префиксом `$`. Переменные шаблона с одинаковым именем всегда захватывают идентичные элементы AST. Исключением является переменная с именем `$_`, их можно использовать для обозначения "что угодно".

Перед именем переменной шаблона можно поставить `*`, тогда переменная будет захватывать произвольное количество элементов.

| Поисковой шаблон | Интерпретация |
|---|---|
| `$_` | Что угодно. |
| `$x` | Идентично первому примеру, "что угодно". |
| `$x = $x` | Самоприсваивание. |
| `(($_))` | Любое выражение в двойных скобках. |
| `if $init; $cond {$x} else {$x}`  | `if` с дублирующимися then/else блоками. |
| `fmt.Fprintf(os.Stdout, $*_)` |  Вызов `Fprintf` с аргументом `os.Stdout`. |

Как уже демонстрировалось в примере с `append()`, шаблон может содержать несколько statement'ов. Нотация "`$x; $y`" означает "найди $x, за которым следует $y".

`gogrep` выполняет честный backtracking для шаблонов с `*`. К примеру, шаблоном можно найти все `map` литералы, где есть хотя бы один дублирующийся ключ:

```
map[$_]$_{$*_, $key: $val1, $*_, $key: $val2, $*_}
```

## Конвейеры и команды gogrep

Ранее мы использовали параметры `-x` и `-s`, не разбирая что они из себя представляют.

`gogrep` оперирует командами, которые составляют конвейер (pipeline). Порядок команд имеет значение. Полный синопсис выглядит следующим образом:

```
gogrep commands... [targets...]
```

`target` может быть файлом, директорией или пакетом. Всё эквивалентно тому, как обрабатывает аргументы команда `go build`.

| Команда | Описание |
|---|---|
| `-x pattern` | Найти все элементы AST, которые подходят под `pattern`. |
| `-g pattern` | Отбросить результаты, которые не подходят под `pattern`. |
| `-v pattern` | Отбросить результаты, которые подходят под `pattern`. |
| `-a attr` | Отбросить результаты, которые не имеют атрибута `attr`. |
| `-s pattern` | Переписать результат, используя `pattern`. |
| `-p n` | Для каждого результата, подняться на `n` уровней по AST. |

Как можно догадаться, `-x` чаще всего является первой командой в конвейере. Затем могут следовать фильтрующие команды или модифицирующие команды.

Рассмотрим это всё на примерах.

```go
// file foo.go
package foo

func bar() {
	println(1)
	println(2)
	println(3)
}
```
```bash
# Находим все вызовы println()
$ gogrep -x 'println($*_)' foo.go
foo.go:4:2: println(1)
foo.go:5:2: println(2)
foo.go:6:2: println(3)

# Добавляем команды -v для отбрасывания всех результатов,
# где есть литерал 1, а затем литерал 2.
$ gogrep -x 'println($*_)' -v 1 -v 2 foo.go
foo.go:6:2: println(3)

# Дополнительно поднимаемся на 2 уровня выше
# и доходим до содержащего *ast.BlockStmt.
$ gogrep -x 'println($*_)' -v 1 -v 2 -p 2 foo.go
foo.go:3:12: { println(1); println(2); println(3); }
```

Атрибутов довольно много, большая часть из них очень ситуативная, а документации на них нет совсем. Остаётся смотреть в [исходниках](https://github.com/mvdan/gogrep/blob/24e8804e5b3cbe82de972195f127eb3c3592d94b/parse.go#L362).

Одним из наиболее полезных атрибутов является `type`:

```bash
# Матчит и сложение, и конкатенацию.
gogrep -x '$lhs + $rhs'

# Матчит только конкатенацию.
gogrep -x '$lhs + $rhs' -a 'type(string)'
```

По умолчанию `gogrep` не выполняет поиск в тестовых файлах. Чтобы это исправить, стоит передавать аргумент `-tests`.

## Обзор возможностей VS Code расширения

Все предоставляемые функции сводятся к нескольким командам (`Ctrl+Shift+P` или `Cmd+Shift+P`):

![](https://habrastorage.org/webt/t6/to/yt/t6toytgl4xxl07-twzwidbguzlu.jpeg)

Каждая команда запрашивает поисковой шаблон:

![](https://habrastorage.org/webt/y8/4q/f9/y84qf9p7mxjajc0oligrbt_8tho.jpeg)

Результаты печатаются в канал (output channel) `gogrep`:

![](https://habrastorage.org/webt/pj/c6/h-/pjc6h-nobw_bpbly_51t-cgi-sk.jpeg)

Для search and replace нужно разделять части "Find" и "Replace" токеном `->`:

![](https://habrastorage.org/webt/72/7c/yy/727cyy4mbar0zq937ya_qfzrovo.jpeg)

Если убрать из шаблона `!`, то вместо изменений файлов inplace в канал будут распечатаны кандидаты для замены.

Пример поиска тех самых комбинируемых `append` (но без replace):

![](https://habrastorage.org/webt/hn/nc/zc/hnnczclqvnxb1maimtze2qlkrqo.gif)

По умолчанию за командами расширения не назначено никаких горячих клавиш. Если вам нужен более быстрый доступ к поиску, вы можете назначить их самостоятельно, следуя личным предпочтениям эргономики.

Пока что автоматическая установка бинарника `gogrep` предусмотрена только для `GOARCH=amd64` и `GOOS=linux|windows|darwin`.

Расширение не предоставляет возможностей использовать атрибуты или произвольные конвейеры. Интегрированы только `-x` и `-s`.

Если вам не хватает какого-то функционала или вы нашли баг, не стесняйтесь и не ленитесь [открывать issue на GitHub](https://github.com/quasilyte/vscode-gogrep/issues/new).

## Заключение

Надеюсь, эта заметка поможет этому замечательному инструменту стать хотя бы немного популярнее.

Если вы используете продукты JetBrains, то вам может быть знаком механизм [structural search and replace](https://www.jetbrains.com/help/idea/structural-search-and-replace.html) (SSR). Он решают ту же задачу, но, в отличие от SSR, `gogrep` удобнее запускать в произвольном окружении, так как это обычная утилита командной строки.

Для автоматического рефакторинга, например, при сохранении файла, можно использовать [ruleguard](https://github.com/quasilyte/go-ruleguard) с опцией `-fix`:

```go
m.Match(`fmt.Fprint(os.Stdout, $*args)`).Suggest(`fmt.Print($args)`)
m.Match(`fmt.Fprintln(os.Stdout, $*args)`).Suggest(`fmt.Println($args)`)
m.Match(`fmt.Fprintf(os.Stdout, $*args)`).Suggest(`fmt.Printf($args)`)
```

Эти три правила будут находить вызовы `Fprint*` с аргументов `Stdout` и заменять их на `Print*` эквиваленты. Шаблоны в `Match()` используют `gogrep` синтаксис.

Дополнительные материалы:
* [Daniel Martí рассказывает о gogrep](https://talks.godoc.org/github.com/mvdan/talks/2018/gogrep.slide)
* [Множество примеров gogrep шаблонов](https://github.com/quasilyte/go-ruleguard/blob/master/rules.go)
* Аналогичный инструмент для PHP - [phpgrep](https://habr.com/ru/post/464893/)
* [VS Code расширение для phpgrep](https://marketplace.visualstudio.com/items?itemName=quasilyte.phpgrep)
* [golang.org/x/tools/cmd/eg](https://godoc.org/golang.org/x/tools/cmd/eg)
