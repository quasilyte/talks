# Ищем баги в PHP коде без статических анализаторов

Моя самая любимая часть в статическом анализе кода - это выдвижение гипотез о потенциальных ошибках в коде с последующей их проверкой.

Пример гипотезы:
```
Функции strpos легко передать аргументы в неправильном порядке. 
```

Но есть вероятность, что даже на нескольких миллионах строк кода подобная диагностика не "выстрелит", поэтому на неудачные гипотезы тратить много времени не хочется.

Сегодня я покажу как выполнять простейший статический анализ с помощью утилиты [phpgrep](https://github.com/quasilyte/phpgrep) без написания кода.

<table>
<tr>
<td><img src="https://habrastorage.org/webt/ct/6n/nt/ct6nntwmvdgniqe2e625qogapj0.png"/></td>
<td>
<hr><p>Под катом:</p>

<ul>
<li>Поиск и разбор багов в open source проектах.</li>
<li>Quick start по phpgrep.</li>
<li>Принцип работы синтаксического поиска.</li>
</ul><br><br><hr></td>
</tr>
</table>

<cut/>

# Предпосылки

Вот уже несколько месяцев я занимаюсь поддержкой PHP линтера [NoVerify](https://github.com/VKCOM/noverify) (почитать о котором можно в статье [NoVerify: линтер для PHP от Команды ВКонтакте](https://habr.com/ru/company/vk/blog/442284/)).

Время от времени в команде появляются идеи для новых диагностик. Идей может быть много, а проверить хочется все, особенно если предложенная проверка нацелена на выявление критических дефектов.

Ранее я активно разрабатывал [go-critic](https://github.com/go-critic/go-critic) и ситуация была схожей, с той лишь разницей, что анализировались исходники на Go, а не на PHP. Когда я узнал об утилите [gogrep](https://github.com/mvdan/gogrep), мой мир перевернулся. Как видно из названия, эта утилита имеет что-то общее с grep'ом, только поиск производится не по регулярным выражениям, а по синтаксическим шаблонам (позже объясню, что это значит).

Я уже не хотел жить без умного grep, поэтому в один вечер решил сесть и написать `phpgrep`.

# Анализируемый корпус

Чтобы было увлекательно, мы сразу погрузимся в применение. Будем анализировать небольшой набор достаточно известных и крупных PHP проектов, доступных на GitHub.

В наш набор попали следующие проекты:
<table>
<tr>
<td><a href="https://github.com/WordPress/WordPress">WordPress</a></td>
<td><a href="https://github.com/Wikia/app">Wikia</a></td>
<td><a href="https://github.com/drupal/drupal">Drupal</a></td>
</tr>

<tr>
<td><a href="https://github.com/yiisoft/yii2">yii2</a></td>
<td><a href="https://github.com/bcit-ci/CodeIgniter">CodeIgniter</a></td>
<td><a href="https://github.com/composer/composer">composer</a></td>
</tr>

<tr>
<td><a href="https://github.com/phalcon/cphalcon">Phalcon</a></td>
<td><a href="https://github.com/joomla/joomla-cms">Joomla!</a></td>
<td><a href="https://github.com/laravel/framework">Laravel</a></td>
</tr>

<tr>
<td><a href="https://github.com/moodle/moodle">moodle</a></td>
<td><a href="https://github.com/owncloud/core">ownCloud core</a></td>
<td><a href="https://github.com/phacility/phabricator">Phabricator</a></td>
</tr>

<tr>
<td><a href="https://github.com/sebastianbergmann/phpunit">PHPUnit</a></td>
<td><a href="https://github.com/symfony/symfony">symfony</a></td>
<td><a href="https://github.com/woocommerce/woocommerce">woocommerce</a></td>
</tr>
</table>

Для людей, которые замышляют то, что мы с вами замышляем, это очень аппетитный набор.

Итак, поехали!

# Использование присваивания в качестве выражения

Если присваивание используется как выражение, причём:
* контекст ожидает результат логической операции (логическое условие) и
* правая часть выражения не имеет побочных эффектов и является константной,

то, скорее всего, в коде ошибка.

Для начала, возьмём за "логический контекст" следующие конструкции:
1. Выражение внутри "`if ($cond)`".
2. Условие тернарного оператора: "`$cond ? $x : $y`".
3. Условия продолжений циклов "`while ($cond)`" и "`for ($init; $cond; $post)`".

В правой части присваивания мы ожидаем константы или литералы.

<spoiler title="Зачем на нужны такие ограничения?">
<table><tr><td>
Оба ограничения нужны для уменьшения количества ложных срабатываний.

Если мы ищем присваивания исключительно внутри условий, то вероятность, что имела место опечатка и вместо "=" подразумевался "==", выше.

Но даже в этом случае остаются спорные ситуации, где присваивание имеет смысл из-за побочных эффектов, которые могут возникать внутри функции. Чтобы не иметь с ними дело, мы будем выбирать только простейшие присваиваемые значения.
</td></td></table>
</spoiler>

Начнём с (1):

```bash
# утилита поиска по синтаксическим шаблонам
# |     производим поиск по текущей директории (и всем дочерним)
# |     |
# |     |
phpgrep . 'if ($_ = []) $_' # 1
#         |               
#         |               
#         строка шаблона поиска

# Дополнительные 3 шаблона через отдельные запуски.
phpgrep . 'if ($_ = ${"const"}) $_' # 2
phpgrep . 'if ($_ = ${"str"}) $_'   # 3
phpgrep . 'if ($_ = ${"num"}) $_'   # 4
```

Здесь мы видим 4 шаблона, единственным различием между которыми выступает присваиваемое выражение (RHS). Начнём с первого из них.

Шаблон "`if ($_ = []) $_`" захватывает `if`, у которого любому выражению присваивается пустой массив. `$_` сопоставляется с любым expression или statement.

```
         литерал пустого массива (RHS)
         |
if ($_ = []) $_
    |        |
    |        любое тело if'а, причём не важно, с {} или без
    любой LHS у присваивания
```

В следующих примерах используются более сложные группы **const**, **str** и **num**. В отличие от `$_` они описывают ограничения на совместимые операции.

* `const` - именованная константа или константа класса.
* `str` - строковой литерал любого типа.
* `num` - числовой литерал любого типа.

Этих шаблонов достаточно, чтобы добиться нескольких срабатываний на корпусе.

⎆ [moodle/blocks/rss_client/viewfeed.php#L37](https://github.com/moodle/moodle/blob/0dca957716952323bd56089929ac6412136a29f1/blocks/rss_client/viewfeed.php#L37-L39):

```php
if ($courseid = SITEID) {
    $courseid = 0;
}
```

Вторым срабатыванием в moodle стала зависимость [ADOdb](https://github.com/ADOdb/ADOdb). В upstream библиотеки проблема всё ещё присутствует.

⎆ [ADOdb/drivers/adodb-odbtp.inc.php#L741](https://github.com/ADOdb/ADOdb/blob/a3410de7c9397046564936e7dc81b7a6143a9e7f/drivers/adodb-odbtp.inc.php#L741):

![](https://habrastorage.org/webt/uo/mv/wj/uomvwjrpztuhmrkm4lolol9sbvg.png)

В этом фрагменте прекрасно многое, но для нас релевантна только первая строка. Вместо сравнения поля `databaseType` мы выполняем присваивание и всегда входим внутрь условия.

Ещё одно интересное место, где мы хотим выполнять действия только для "корректных" записей, но, вместо этого, выполняем их всегда и, более того, отмечаем любую запись как корректную!

⎆ [moodle/question/format/blackboard_six/formatqti.php#L598](https://github.com/moodle/moodle/blob/0dca957716952323bd56089929ac6412136a29f1/question/format/blackboard_six/formatqti.php#L598-L600):

```php
// For BB Fill in the Blank, only interested in correct answers.
if ($response->feedback = 'correct') {
    // ...
}
```

<spoiler title="Расширенный список шаблонов для этой проверки">
<table><tr><td><pre>phpgrep . 'for ($_; $_ = []; $_) $_'
phpgrep . 'for ($_; $_ = ${"const"}; $_) $_'
phpgrep . 'for ($_; $_ = ${"num"}; $_) $_'
phpgrep . 'for ($_; $_ = ${"str"}; $_) $_'
phpgrep . 'while ($_ = []) $_'
phpgrep . 'while ($_ = ${"const"}) $_'
phpgrep . 'while ($_ = ${"num"}) $_'
phpgrep . 'while ($_ = ${"str"}) $_'
phpgrep . 'if ($_ = []) $_'
phpgrep . 'if ($_ = ${"const"}) $_'
phpgrep . 'if ($_ = ${"str"}) $_'
phpgrep . 'if ($_ = ${"num"}) $_'
phpgrep . '$_ = [] ? $_ : $_'
phpgrep . '$_ = ${"const"} ? $_ : $_'
phpgrep . '$_ = ${"str"} ? $_ : $_'
phpgrep . '$_ = ${"num"} ? $_ : $_'
phpgrep . '($_ = []) && $_'
phpgrep . '($_ = ${"const"}) && $_'
phpgrep . '($_ = ${"str"}) && $_'
phpgrep . '($_ = ${"num"}) && $_'
phpgrep . '$_ && $_ = []'
phpgrep . '$_ && $_ = ${"const"}'
phpgrep . '$_ && $_ = ${"str"}'
phpgrep . '$_ && $_ = ${"num"}'
phpgrep . '($_ = []) || $_'
phpgrep . '($_ = ${"const"}) || $_'
phpgrep . '($_ = ${"str"}) || $_'
phpgrep . '($_ = ${"num"}) || $_'
phpgrep . '$_ || $_ = []'
phpgrep . '$_ || $_ = ${"const"}'
phpgrep . '$_ || $_ = ${"str"}'
phpgrep . '$_ || $_ = ${"num"}'</pre></td></td></table>
</spoiler>

Повторим то, что мы изучили:
* Шаблоны выглядят как PHP-код, который они находят.
* `$_` обозначает "что угодно". Можно сравнить с `.` в регулярных выражениях.
* `${"<class>"}` работает как `$_` с ограничением на тип элементов AST.

Стоит ещё подчеркнуть, что всё, кроме переменных, сопоставляется дословно (literally). Это значит, что шаблону "`array(1, 2 + 3)`" будет удовлетворять лишь идентичный по синтаксической структуре код (пробелы не влияют). С другой стороны, шаблону "`array($_, $_)`" удовлетворяет любой литерал массива из двух элементов.

# Сравнение выражения с самим собой

Потребность сравнить что-либо с самим собой возникает очень редко. Это может быть проверка на `NaN`, но как минимум в половине случаев это ошибка copy/paste.

⎆ [Wikia/app/extensions/SemanticDrilldown/includes/SD_FilterValue.php#L103](https://github.com/Wikia/app/blob/26290575a662d621269a272339ec6154602736cd/extensions/SemanticDrilldown/includes/SD_FilterValue.php#L103):

```php
if ( $fv1->month == $fv1->month ) return 0;
```

Справа должно быть "`$fv2->month`".

Для выражения повторяющихся частей в шаблоне мы используем переменные с именами, отличными от "`_`". Механизм повторений в шаблоне похож на [обратные ссылки](https://www.regular-expressions.info/backref.html) в регулярных выражениях.

Шаблон "`$x == $x`" будет как раз тем, что найдёт пример выше. Вместо "`x`" может использоваться любое имя. Здесь важно лишь то, чтобы имена были идентичны. Переменные шаблона, которые имеют различающиеся имена, не обязаны иметь совпадающее содержимое при захвате.

Следующий пример найден с помощью "`$x <= $x`".

⎆ [Drupal/core/modules/views/tests/src/Unit/ViewsDataTest.php#L166](https://github.com/drupal/core/blob/a0093891fec7368cc6ec8276d44e28cac103465b/modules/views/tests/src/Unit/ViewsDataTest.php#L166):

```php
$prev = $base_tables[$base_tables_keys[$i - 1]];
$current = $base_tables[$base_tables_keys[$i]];
$this->assertTrue(
  $prev['weight'] <= $current['weight'] &&
  $prev['title'] <= $prev['title'], // <-------------- ошибка
  'The tables are sorted as expected.');
```

# Дублирующиеся подвыражения

Теперь, когда мы знаем про возможности повторяемых подвыражений, мы можем составить множество любопытных шаблонов.

Один из моих любимцев - "`$_ ? $x : $x`".<br>
Это тернарный оператор, у которого true/false ветки идентичны.

⎆ [joomla-cms/libraries/src/User/UserHelper.php#L522](https://github.com/joomla/joomla-cms/blob/81d7fcf47b821ec8f8a99fe5d87b2411579410f1/libraries/src/User/UserHelper.php#L522):

```php
return ($show_encrypt) ? '{SHA256}' . $encrypted : '{SHA256}' . $encrypted;
```

Обе ветви дублируются, что подсказывает о потенциальной проблеме в коде. Если мы посмотрим на код вокруг, то сможем понять, что там должно было быть вместо этого. В угоду читабельности я вырезал часть кода и сократил название переменной `$encrypted` до `$enc`.

```php
case 'crypt-blowfish':
	return ($show_encrypt ? '{crypt}' : '') . crypt($plaintext, $salt);
case 'md5-base64':
	return ($show_encrypt) ? '{MD5}' . $enc : $enc;
case 'ssha':
	return ($show_encrypt) ? '{SSHA}' . $enc : $enc;
case 'smd5':
	return ($show_encrypt) ? '{SMD5}' . $enc : $enc;
case 'sha256':
	return ($show_encrypt) ? '{SHA256}' . $enc : '{SHA256}' . $enc;
default:
	return ($show_encrypt) ? '{MD5}' . $enc : $enc;
```

Я бы поставил на то, что коду необходим следующий патч:

```diff
- ($show_encrypt) ? '{SHA256}' . $encrypted : '{SHA256}' . $encrypted;
+ ($show_encrypt) ? '{SHA256}' . $encrypted : $encrypted;
```

# Опасные приоритеты операций в PHP

Хорошей мерой предосторожности в PHP является использование группирующих скобочек везде, где важно иметь правильный порядок вычислений.

Во многих языках программирования выражение "`x & mask != 0`" имеет интуитивный смысл. Если `mask` описывает какой-то бит, то данный код проверяет, что в `x` этот бит не равен нулю. К сожалению, для PHP это выражение будет вычисляться так: "`x & (mask != 0)`", что почти всегда не то что вам нужно.

WordPress, Joomla и moodle используют [SimplePie](https://github.com/simplepie/simplepie).

⎆ [SimplePie/library/SimplePie/Locator.php#L254](https://github.com/simplepie/simplepie/blob/38b504969ed08903cb12718e8270263a8c93080e/library/SimplePie/Locator.php#L254)
⎆ [SimplePie/library/SimplePie/Locator.php#L384](https://github.com/simplepie/simplepie/blob/38b504969ed08903cb12718e8270263a8c93080e/library/SimplePie/Locator.php#L384)
⎆ [SimplePie/library/SimplePie/Locator.php#L412](https://github.com/simplepie/simplepie/blob/38b504969ed08903cb12718e8270263a8c93080e/library/SimplePie/Locator.php#L412)
⎆ [SimplePie/library/SimplePie/Sanitize.php#L349](https://github.com/simplepie/simplepie/blob/38b504969ed08903cb12718e8270263a8c93080e/library/SimplePie/Sanitize.php#L349)
⎆ [SimplePie/library/SimplePie.php#L1634](https://github.com/simplepie/simplepie/blob/38b504969ed08903cb12718e8270263a8c93080e/library/SimplePie.php#L1634)

```php
$feed->method & SIMPLEPIE_FILE_SOURCE_REMOTE === 0
```

`SIMPLEPIE_FILE_SOURCE_REMOTE` определён как `1`, поэтому выражение будет эквивалентно:

```
$feed->method & (1 === 0)
// =>
$feed->method & false
```

<spoiler title="Примеры шаблонов для поиска">
<table><tr><td><pre>phpgrep . '$x & $mask == $y'
phpgrep . '$x & $mask === $y'
phpgrep . '$x & $mask !== $y'
phpgrep . '$x & $mask != $y'
phpgrep . '$x | $mask == $y'
phpgrep . '$x | $mask === $y'
phpgrep . '$x | $mask !== $y'
phpgrep . '$x | $mask != $y'</pre></td></td></table>
</spoiler>

Продолжая тему неожиданных приоритетов операций, вы можете почитать про [тернарный оператор в PHP](https://stackoverflow.com/questions/6224330/understanding-nested-php-ternary-operator). На хабре этому даже посвятили статью: [Порядок выполнения тернарного оператора](https://habr.com/ru/post/114899/).

Можно ли такие места найти с помощью `phpgrep`? Ответ: **да**!

```bash
phpgrep . '$_ == $_ ? $_ : $_ ? $_ : $_'
phpgrep . '$_ != $_ ? $_ : $_ ? $_ : $_'
```

# Прелести проверки URL с помощью регулярных выражений

⎆ [Wikia/app/maintenance/wikia/updateCentralInterwiki.inc#L95](https://github.com/Wikia/app/blob/dev/maintenance/wikia/updateCentralInterwiki.inc#L95):

```php
if ( preg_match( '/(wowwiki.com|wikia.com|falloutvault.com)/', $url ) ) {
	$local = 1;
} else {
	$local = 0;
}
```

По задумке автора кода, мы проверяем URL на совпадение с одним из 3-х вариантов. К сожалению, символ `.` не экранирован, что приведёт к тому, что вместо `falloutvault.com` мы можем завести себе `falloutvaultxcom` на любом домене и пройти проверку.

![](https://habrastorage.org/webt/n7/sj/pc/n7sjpcnsabhg5ggan7tiscxy0q4.png)

Это не PHP-специфичная ошибка. В любом приложении, где валидация выполняется через регулярные выражения и частью проверяемой строки является мета-символ, есть риск забыть экранирование там, где оно нужно, и получить уязвимость.

Найти такие места можно с помощью запуска `phpgrep`:
```bash
phpgrep . 'preg_match(${"pat:str"}, ${"*"})' 'pat~[^\\]\.(com|ru|net|org)\b'
```

Мы вводим именованный подшаблон `pat`, который захватывает любой строковой литерал, а затем применяем к нему фильтр из регулярного выражения.

Фильтры можно применять к любой переменной шаблона. Кроме регулярных выражений есть также структурные операторы `=` и `!=`. Полный список можно найти в [документации](https://github.com/quasilyte/phpgrep/blob/master/pattern_language.md).

`${"*"}` захватывает произвольное количество любых аргументов, поэтому нам можно не волноваться за опциональные параметры функции `preg_match`.

# Дубликаты ключей в литерале массива

В PHP вы не получите никакого предупреждения, если выполните этот код:

```php
<?php
var_dump(['a' => 1, 'a' => 2]);
// Результат: array(1) {["a"]=> int(2)}
```

Мы можем найти такие массивы с помощью `phpgrep`:

```
[${"*"}, $k => $_, ${"*"}, $k => $_, ${"*"}]
```

Этот шаблон можно расшифровать так: "литерал массива, в котором есть хотя бы два идентичных ключа в произвольной позиции". Выражения `${"*"}` помогают нам описать "произвольную позицию", допуская 0-N элементов до, между и после интересующих нас ключей.

⎆ [Wikia/app/extensions/wikia/WikiaMiniUpload/WikiaMiniUpload_body.php#L23](https://github.com/Wikia/app/blob/26290575a662d621269a272339ec6154602736cd/extensions/wikia/WikiaMiniUpload/WikiaMiniUpload_body.php#L23-L24):

```php
$script_a = [
	'wmu_back' => wfMessage( 'wmu_back' )->escaped(),
	'wmu_back' => wfMessage( 'wmu_back' )->escaped(),
	// ...
];
```

В данном случае это не является грубой ошибкой, но мне известны случаи, когда дублирование ключей в крупных (100+ элементов) массивах несло как минимум неожиданное поведение, в котором один из ключей перекрывал значение другого.

<hr>

На этом наш краткий экскурс на примерах окончен. Если вам хочется ещё, в конце статьи описано, как получить все результаты.

# Что же такое phpgrep?

Большая часть редакторов и IDE используют для поиска кода (если это не поиск по специальному символу типа класса или переменной) обычный текстовой поиск — проще говоря, что-то вроде grep'а.

Вы вводите "`$x`", находите "`$x`". Вам могут быть доступны регулярные выражения, тогда вы можете пытаться, по сути, парсить PHP-код регулярками. Иногда это даже работает, если ищется что-то вполне определённое и простое — например, «любая переменная с некоторым суффиксом». Но если эта переменная с суффиксом должна быть частью другого составного выражения, возникают трудности.

[phpgrep](https://github.com/quasilyte/phpgrep) — это инструмент для удобного поиска PHP-кода, который позволяет искать не с помощью text-oriented регулярок, а с помощью syntax-aware шаблонов.

Syntax-aware означает, что язык шаблонов отражает целевой язык, а не оперирует отдельными символами, как это делают регулярные выражения. Нам также нет никакой разницы до форматирования кода, важна лишь его структура.

<spoiler title="Опциональный контент: Quick Start">

# Quick start

### Установка

Для amd64 есть готовые [релизные сборки под Linux и Windows](https://github.com/quasilyte/phpgrep/releases), но если у вас установлен Go, то достаточно одной команды, чтобы получить свежий бинарник под вашу платформу:

```bash
go get -v github.com/quasilyte/phpgrep/cmd/phpgrep
```

Если `$GOPATH/bin` находится в системном `$PATH`, то команда `phpgrep` станет сразу же доступной. Чтобы это проверить, попробуйте запустить команду с параметром `-help`:

```bash
phpgrep -help
```

Если же ничего не происходит, найдите, куда Go установил бинарник и добавьте его в переменную окружения `$PATH`.

Старый и надёжный способ посмотреть `$GOPATH`, даже если он не выставлен явно: 

```bash
go env GOPATH
```

### Использование

Создайте тестовый файл `hello.php`:

```php
<?php
function f(...$xs) {}
f(10);
f(20);
f(30);
f($x);
f();
```

Запустите на нём `phpgrep`:

```bash
# phpgrep hello.php 'f(${"x:int"})' 'x!=20'
hello.php:3: f(10)
hello.php:5: f(30)
```

Мы нашли все вызовы функции `f` с одним аргументом-числом, значение которого не равно 20.
</spoiler>

# Как работает phpgrep

Для разбора PHP используется библиотека [github.com/z7zmey/php-parser](https://github.com/z7zmey/php-parser). Она достаточно хороша, но некоторые ограничения `phpgrep` следуют из особенностей используемого парсера. Особенно много трудностей возникает при попытках нормально работать со скобочками.

Принцип работы `phpgrep` прост:
* строится AST из входного шаблона, разбираются фильтры;
* для каждого входного файла строится полное AST-дерево;
* обходим AST каждого файла, пытаясь найти такие поддеревья, которые совпадают с шаблоном;
* для каждого результата применяется список фильтров;
* все результаты, которые прошли фильтры, печатаются на экран.

Самое интересное — это как именно сопоставляются на равенство два AST-узла. Иногда тривиально: один-к-одному, а мета-узлы могут захватывать более одного элемента. Примерами мета-узлов является `${"*"}` и `${"str"}`.

# Заключение

Было бы нечестно говорить о `phpgrep`, не упомянув [structural search and replace](https://www.jetbrains.com/help/phpstorm/structural-search-and-replace.html) (SSR) из PhpStorm. Они решают похожие задачи, причём у SSR есть свои преимущества, например, интеграция в IDE, а `phpgrep` может похвастаться тем, что является standalone программой, которую гораздо проще поставить, например, на CI.

Помимо прочего, `phpgrep` - это ещё и библиотека, которую можно использовать в своих программах для матчинга PHP кода. Особенно это полезно для линтеров и кодогенерации.

Буду рад, если этот инструмент будет вам полезен. Если же эта статья просто мотивирует вас посмотреть в сторону вышеупомянутого SSR, тоже хорошо.

<img src="https://habrastorage.org/webt/o_/9a/cz/o_9aczasnsf3wssmuunol5lyjfe.png">
<br>

# Дополнительные материалы

Полный список шаблонов, который был использован для анализа, можно найти в файле [patterns.txt](https://github.com/quasilyte/phpgrep-contrib/blob/master/scripts/patterns.txt). Рядом с этим файлом можно найти скрипт `phpgrep-lint.sh`, упрощающий запуск `phpgrep` со списком шаблонов.

В статье не дан полный список срабатываний, но вы можете воспроизвести эксперимент, произведя клонирование всех названых репозиториев и запустив `phpgrep-lint.sh` на них.

Черпать вдохновение на шаблоны проверок можно, например, из статей [PVS studio](https://habr.com/ru/company/pvs-studio/). Мне очень понравилась [Logical Expressions: Mistakes Made by Professionals](https://www.viva64.com/en/b/0390/), которая трансформируется во что-то такое:

```
# Для "x != y || x != z":
phpgrep . '$x != $a || $x != $b'
phpgrep . '$x !== $a || $x != $b'
phpgrep . '$x != $a || $x !== $b'
phpgrep . '$x !== $a || $x !== $b'
```

Вам также может быть интересна презентация [phpgrep: syntax-aware code search](https://speakerdeck.com/quasilyte/phpgrep-syntax-aware-code-search).

> В статье используются изображения гоферов, которые были созданы через [gopherkon](https://quasilyte.dev/gopherkon/?state=0a03080f020801000002000e00000009).
