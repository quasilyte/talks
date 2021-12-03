# gocorpus: открытый корпус Go кода, поддерживающий запросы

На днях я запустил [wasm-приложение](quasilyte.dev/gocorpus/), которое позволяет запускать [gogrep](https://habr.com/ru/post/505652/) шаблоны на относительно крупном корпусе Go кода (~11 миллионов строк кода).

 В этой заметке я напишу как этим пользоваться и зачем оно вообще может быть нужно.

<s>Звёздочки нести сюда</s> Исходный код можно найти здесь: [github.com/quasilyte/gocorpus](https://github.com/quasilyte/gocorpus).

![](https://habrastorage.org/webt/h4/6j/ld/h46jldankein58g_ayg_7iwf1si.png)

<cut/>

## Зачем?

Допустим, вы хотите проверить утверждение, что в среднестатистическом Go кода так никто не пишет (или наоборот, что все так пишут). Для этого вам придётся выполнить эти шаги:

1. Собрать коллекцию Go кода. Несколько репозиториев, желательно разнообразных.
2. Придумать, как исполнять поиск. Регулярные выражения могут быть не самым подходящим инструментом.
3. Сделать результаты воспроизводимыми для других людей, чтобы им не пришлось верить вам на слово.

Проект [gocorpus](https://quasilyte.dev/gocorpus/) решает все эти шаги за вас.

> Разве что на момент написания статьи пункт `(3)` не решён полностью. Другой человек может зайти на страницу и повторить запрос, но "share" не реализован. По задумке, share будет выдавать URL с опциями для запуска: шаблон поиска, фильтры, выбранные репозитории. 

Поскольку gocorpus лично мне нужен был для проверки статистики, а не именно для поиска по проектам, результаты сейчас выдаются без информации о локации в исходном коде. Это не принципиальное ограничение: я планирую добавить настройку формата результатов в будущем. Но нужно учитывать, что мне интереснее добавлять агрегацию и инсайты по результатам, а не превращать это в песочницу для gogrep. Например, мне очень хотелось бы вычислять некоторый hit rate для шаблона, чтобы легче было понять частоту результата.

## О существующих решениях

Я несколько раз слышал о [bigquery корпусе](https://cloud.google.com/bigquery/public-data/), но, насколько мне известно, это не совсем бесплатная штука. К тому же, я считаю, что этим вариантом не очень удобно пользоваться.

[codesearch](https://github.com/google/codesearch) работает на регулярных выражениях. Этого не всегда достаточно, как я уже упоминал выше.

Был ещё старенький [корпус](https://github.com/rsc/corpus) от Russ Cox, но он уже в архиве и его содержимое никогда не актуализировалось. Тем более это просто коллекция кода, а не готовое решение всё-в-одном.

[Daniel Marti](https://github.com/mvdan/) (оригинальный автор [gogrep](https://github.com/mvdan/gogrep)) собрал что-то вроде индекса популярного кода: [github.com/mvdan/corpus](https://github.com/mvdan/corpus). В теории, этот индекс можно использовать для формирования набора репозиториев, доступных в моём приложении.

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
* `file.IsTest()` true для файлов с суффиксом `_test.go` в названии или `_test` в имени пакета
* `file.IsMain()` true для файлов с именем пакета `main`
* `file.IsAutogen()` true для файлов, которые размечены как автоматически сгенерированные

Фильтры - это обычные Go выражения. Их можно сочетать через `&&`. Также можно использовать `(` и `)` для группирования, а `!` для инвертирования эффекта.

`file` - это предопределённая переменная, которая привязана к текущему обрабатываемому файлу. `$<var>` - это переменная из шаблона поиска.

* `$x.IsConst() && !file.IsTest()` - не искать в тестах, `$x` должен быть константным.
* `$x.IsPure() && !$y.IsPure()` - `$x` должен быть чистым выражением, а `$y` - нет.
* `!file.IsAutogen() && !file.IsTest()` - не искать в тестах и автоматически сгенерированных файлах.

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

Если вы считаете, что нужно добавить какой-то репозиторий в корпус, [сообщите мне об этом](https://github.com/quasilyte/gocorpus/issues/new). Аналогично, если не хватает какой-то фичи или вы нашли баг, тоже открываем issue и я постараюсь это исправить.

Буду рад обратной связи.

P.S. - у меня мало опыта работы с фронтендом, поэтому код на TypeScript и вёртска оставляют желать лучшего. Если кто-то поможет с этой частью, я буду очень признателен.