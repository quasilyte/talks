# gocorpus: открытый корпус Go кода, поддерживающий запросы

На днях я запустил [wasm-приложение](quasilyte.dev/gocorpus/), которое позволяет запускать [gogrep шаблоны](https://habr.com/ru/post/505652/) на относительно крупном корпусе Go кода (~11 миллионов строк кода).

В этой заметке я напишу как этим пользоваться и зачем оно вообще может быть нужно.

<s>Звёздочки нести сюда</s> Исходный код можно найти здесь: [github.com/quasilyte/gocorpus](https://github.com/quasilyte/gocorpus).

![](https://habrastorage.org/webt/h4/6j/ld/h46jldankein58g_ayg_7iwf1si.png)

<cut/>

## Зачем?

Допустим, вы хотите проверить утверждение, что в среднестатистическом Go коде так никто не пишет (или наоборот, что все так пишут). Для этого вам придётся выполнить эти шаги:

1. Собрать коллекцию Go кода. Несколько репозиториев, желательно разнообразных.
2. Придумать, как исполнять поиск. Регулярные выражения могут быть не самым подходящим инструментом.
3. Сделать результаты воспроизводимыми для других людей, чтобы им не пришлось верить вам на слово.

Проект [gocorpus](https://quasilyte.dev/gocorpus/) решает все эти шаги за вас.

> Разве что на момент написания статьи пункт `(3)` не решён полностью. Другой человек может зайти на страницу и повторить запрос, но "share" не реализован. По задумке, share будет выдавать URL с опциями для запуска: шаблон поиска, фильтры, выбранные репозитории. 

Поскольку gocorpus лично мне нужен был для проверки статистики, а не именно для поиска по проектам, результаты сейчас выдаются без информации о локации в исходном коде. Это не принципиальное ограничение: я планирую добавить настройку формата результатов в будущем.

## О существующих решениях

Я несколько раз слышал о [bigquery корпусе](https://cloud.google.com/bigquery/public-data/), но, насколько мне известно, это не совсем бесплатная штука. К тому же, я считаю, что этим вариантом не очень удобно пользоваться.

[codesearch](https://github.com/google/codesearch) работает на регулярных выражениях. Этого не всегда достаточно, как я уже упоминал выше.

Был ещё старенький [корпус](https://github.com/rsc/corpus) от Russ Cox, но он уже в архиве и его содержимое никогда не актуализировалось. Тем более это просто коллекция кода, а не готовое решение всё-в-одном.

[Daniel Marti](https://github.com/mvdan/) (оригинальный автор [gogrep](https://github.com/mvdan/gogrep)) собрал что-то вроде индекса популярного кода: [github.com/mvdan/corpus](https://github.com/mvdan/corpus). В теории, этот индекс можно использовать для формирования набора репозиториев, доступных в моём приложении.

И некоторые другие:

* https://cs.bazel.build/
* https://grep.app/
* https://sourcegraph.com/search/

Моё решение выгодно отличается как минимум тем, что оно заточено под Go: правильно распознаются autogenerated файлы, файлы с тестами, а к выражениям можно применять фильтры. Например, можно требовать от них того, чтобы они не имели побочных эффектов (`$x.IsPure`).

## Корпус кода

Для выбора репозиториев я делал поиск по GitHub с фильтрами и сортировкой по количеству звёздочек. Не самый научный подход, но он достаточно хорош для первой итерации.

После этого выбранные репозитории клонируются, анализируются, [минифицируются](https://github.com/quasilyte/minformat/tree/master/go/minformat) и кладутся в tar архивы со сжатием.

По мере анализа файлов, мы записываем в метаданные некоторые метрики и факты о файле. Например, импортирует ли файл "unsafe" или "C" (cgo). Эту информацию затем можно использовать в фильтрах.

На клиенте мы качаем `tar.gz` файлы и на месте их разжимаем.

Подробнее о том, как собирается корпус, можно посмотреть в [makecorpus](https://github.com/quasilyte/gocorpus/tree/master/makecorpus).

![](https://habrastorage.org/webt/n0/ta/yp/n0taypnndadre3rp6cjgfq0fm80.gif)

## Фильтры

На текущий момент реализованы следующие фильтры:

* `$var.IsConst()` выражение, захваченное `$var` является константой
* `$var.IsPure()` выражение, захваченное `$var` не имеет побочных эффектов
* `$var.Is<Kind>Lit()` выражение, захваченное `$var` является литералом данного типа*
* `file.IsTest()` true для файлов с суффиксом `_test.go` в названии или `_test` в имени пакета
* `file.IsMain()` true для файлов с именем пакета `main`
* `file.IsAutogen()` true для файлов, которые размечены как автоматически сгенерированные
* `file.MaxDepth()` int значение, которое можно сравнивать с константой**

> `(*)` Kind может быть `String`, `Int`, `Float`, `Rune` (`token.CHAR`), `Complex`.

> `(**)` Некоторые файлы имеют аномальную максимальную глубину AST, из-за чего в некоторых браузерах могут происходить stack overflow внутри wasm кода. С помощью подбора правильного `MaxFileDepth` можно обойти эту проблему, игнорируя эти страшные файлы.

Фильтры - это обычные Go выражения. Их можно сочетать через `&&` и `||`. Также можно использовать `(` и `)` для группирования, а `!` для инвертирования эффекта.

`file` - это предопределённая переменная, которая привязана к текущему обрабатываемому файлу. `$<var>` - это переменная из шаблона поиска.

* `$x.IsConst() && !file.IsTest()` - не искать в тестах, `$x` должен быть константным.
* `$x.IsPure() && !$y.IsPure()` - `$x` должен быть чистым выражением, а `$y` - нет.
* `!file.IsAutogen() && !file.IsTest()` - не искать в тестах и автоматически сгенерированных файлах.
* `file.MaxDepth() <= 100` - пропускать файлы, в которых максимальная глубина AST выше 100

Если вы пользовались [ruleguard](https://github.com/quasilyte/go-ruleguard), то вам это может напомнить фильтры в `Where()`.

## Примеры запросов

Приведу несколько примеров поисковых шаблонов.

| Шаблон | Описание |
|---|---|
| `copy($_, []byte($_))` | Находим избыточные конвертации к `[]byte`. |
| `reflect.DeepEqual($x, $x)` | Сомнительное использование DeepEqual. |
| `if err != nil { return nil }` | Потенциальные опечатки в обработке ошибок. |
| `ioutil.ReadAll($_)` | Находим использования deprecated функции. |
| `var()` | Пустой gen decl блок для переменных. |
| `len($_) >= 0` | Ошибочная проверка длины (always true). |

Одинаковые имена переменных будут требовать идентичных матчей. То есть `$x = $x` находит только самоприсваивания, а `$x = $y` может найти любые присваивания (в том числе и самоприсваивания). Исключением из правил является `$_` - пустая переменная не требует соответствий даже если она используется несколько раз.

Вот паттерн посложнее: `map[$_]$_{$*_, $k: $_, $*_, $k: $_, $*_}`. Он находит map-литералы, которые содержат ключи-дубликаты. Модификатор `*` работает прямо как в регулярных выражениях: будет 0 или более матчей. Чтобы найти любые вызовы `fmt.Printf`, мы можем сделать такой шаблон: `fmt.Printf($*_)`.

Переменная с модификатором необязательно должна быть пустой. Например, вот это тоже валидный шаблон: `fmt.Printf($format, $*args)`.

Модификатора `+` нет, но его часто можно эмулировать вот так: `f($_, $*_)` - вызовы `f` с одним или более аргументами.

Больше примеров шаблонов можно подсмотреть в [правилах go-critic](https://github.com/go-critic/go-critic/blob/master/checkers/rules/rules.go).

![](https://habrastorage.org/webt/w0/gc/iy/w0gciy8eenipshzritjzofbuswo.png)

## Frequency score

Если сделать несколько запросов, то можно сравнить количество матчей между ними. Если паттерн X даёт в 2 раза больше матчей, то считаем, что он встречается в коде в такое же количество раз чаще.

Но как анализировать частоту паттерна относительно среднестатистического кода? Мифические 100 матчей могут значить разное в зависимости от размера данных, на которых мы запускали запрос.

Чтобы посчитать некоторую частотность, я взял за основу частоту `err != nil`. На 11 миллионах строк кода нашлось ~150 тысяч этих проверок на ошибку. Будем считать, что коэффициент `1.0` - это 1 матч на 70 строк кода. Следовательно, если тестируемый шаблон встречается раз в 140 строк, то его frequency score будет равен `1.0`. Чтобы результаты было более легко интерпретировать, я домножаю значение на 100, чтобы получились более красивые глазу числа.

Значение frequency score выдаётся на уровне с другими результатами после полного исполнения запроса.

## Заключение

Если вы считаете, что нужно добавить какой-то репозиторий в корпус, [сообщите мне об этом](https://github.com/quasilyte/gocorpus/issues/new). Аналогично, если не хватает какой-то фичи или вы нашли баг, тоже открываем issue и я постараюсь это исправить.

Буду рад обратной связи.

P.S. - у меня мало опыта работы с фронтендом, поэтому код на TypeScript и вёртска оставляют желать лучшего. Если кто-то поможет с этой частью, я буду очень признателен.

