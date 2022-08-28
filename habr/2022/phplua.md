[Foreign Function Interface](https://www.php.net/manual/ru/class.ffi.php) - это перспективная альтернатива для традиционных PHP-расширений.

Сегодня мы будем разбирать FFI-библиотеку для работы с [liblua5](https://www.lua.org/manual/5.3/) из PHP, которая позволит исполнять скрипты на Lua из нашего приложения.

![](https://habrastorage.org/webt/xv/em/nl/xvemnlq_kqrasv7-eqlz0uy4hdc.png)

<cut/>

## Мотивация

Для PHP уже есть расширение Lua с [PECL](https://pecl.php.net/package/lua). Тем не менее смысл в нашей задумке есть:

1. Через FFI есть доступ к полному Lua API, поэтому у нас больше свободы.
2. FFI-библиотеки не используют Zend API, им проще пережить мажорный релиз PHP.
3. Читать PHP код нам обычно проще, чем C в связке с внутренностями PHP.
4. Легче распространять результат как composer пакет, ведь это обычный PHP-код.

Ещё одним бонусом FFI-библиотек является то, что они будут успешно запускаться и на [KPHP](github.com/VKCOM/kphp), если мы правильно расставим типы через phpdoc.

Сейчас один из главных недостатков этого подхода - не очень высокая производительность. И хотя в KPHP использование FFI несёт минимальные накладные расходы, в PHP всё гораздо сложнее. Автор JIT-компиляции в ядре PHP, [Дмитрий Стогов](https://github.com/dstogov), планирует в отдалённом будущем «подружить» JIT и FFI, значительно увеличив производительность этого механизма.

> В качестве занимательного факта: Дмитрий, помимо прочего, ещё и автор FFI для PHP.

Но зачем именно liblua? Есть две основные причины:

1. Это уникальный и довольно сложный пример для FFI. Полезен в образовательных целях.
2. В компилируемом KPHP полезно иметь возможность использовать динамические плагины.

## Подготовка к началу

На Хабре я уже несколько описывал, как создавать composer-пакеты для FFI-библиотек, поэтому сегодня мы сразу перейдём к делу. Я буду придерживаться практик, изложенных в статье [«Используем SQLite в KPHP и PHP через FFI»](https://habr.com/ru/post/653677/).

Нам потребуется установить liblua. Подойдут любые версии в диапазоне 5.1-5.4. Затем находим в системе эту библиотеку. На Linux нам может помочь утилита `ldconfig`.

```bash
# Запомним путь, по которому можно найти библиотеку,
# он нам скоро понадобится.
$ ldconfig -p | grep lua
	liblua5.3.so.0 (libc6,x86-64) => /lib/x86_64-linux/liblua5.3.so.0
```

Нам также потребуются полифилы для KPHP:

```bash
$ composer require vkcom/kphp-polyfills
```

## Hello, world!

Чтобы использовать Lua, надо получить [`lua_State`](https://www.lua.org/manual/5.4/manual.html#lua_State). Для этого можно воспользоваться функцией [`luaL_newstate`](https://www.lua.org/manual/5.3/manual.html#luaL_newstate).

Для запуска какого-нибудь кода на Lua можно было бы взять [`luaL_dostring`](https://www.lua.org/manual/5.3/manual.html#luaL_dostring), но это макрос. Макросы использовать у нас не получится, но мы можем подсмотреть в его определение:

```cpp
#define luaL_dostring(L, str) \
    (luaL_loadstring(L, str) || lua_pcall(L, 0, LUA_MULTRET, 0))

#define lua_pcall(L, n, r, f) \
    lua_pcallk(L, (n), (r), (f), 0, NULL)
```

Добавляем в список функции [`luaL_loadstring`](https://www.lua.org/manual/5.3/manual.html#luaL_loadstring) и [`lua_pcallk`](https://www.lua.org/manual/5.3/manual.html#lua_pcallk).

Без стандартной библиотеки будет сложновато вывести сообщение на экран, поэтому возьмём ещё и [`luaL_openlibs`](https://www.lua.org/manual/5.4/manual.html#luaL_openlibs).

Наш минимальный заголовочный файл для FFI, `lua.h`, будет выглядеть так:

```cpp
#define FFI_LIB "./ffilibs/liblua5"
#define FFI_SCOPE "lua"

typedef struct lua_State lua_State;
typedef intptr_t lua_KContext;

int luaL_loadstring(lua_State *L, const char *s);

int lua_pcallk(lua_State *L,
               int nargs,
               int nresults,
               int errfunc,
               lua_KContext ctx,
               void *k);

lua_State *luaL_newstate();

void luaL_openlibs(lua_State *L);
```

Теперь разместим liblua там, где его сможет найти наша библиотека:

```bash
$ mkdir ffilibs
$ cp /lib/x86_64-linux/liblua5.3.so.0 ffilibs/liblua5
```

Для PHP [`FFI::load`](https://www.php.net/manual/ru/ffi.load.php) будет работать с [`FFI::scope`](https://www.php.net/manual/ru/ffi.scope.php) только при использовании внутри opcache preload. В случае с KPHP у нас нет opcache preload, но `FFI::load` является более быстрой операцией и должен выполняться где-то в начале скрипта.

Создадим два скрипта: `main.php` и `preload.php`.

`preload.php`:

```php
<?php

// Для PHP выполняем load в preload-контексте.
FFI::load(__DIR__ . '/lua.h');
```

`main.php`:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

if (KPHP_COMPILER_VERSION) {
    // Для KPHP выполняем load в начале скрипта.
    FFI::load(__DIR__ . '/lua.h');
}

// Чтобы было лаконичнее, не проверяем статусы операций ниже и не
// обрабатываем ошибки.
$lib = FFI::scope('lua');
$state = $lib->luaL_newstate();
$lib->luaL_openlibs($state);
$lib->luaL_loadstring($state, 'print("Hello, World!")');
$lib->lua_pcallk($state, 0, 0, 0, 0, null);
```

Попробуем запустить нашу программу через PHP:

```bash
$ php -d opcache.enable_cli=1 \
      -d opcache.preload=preload.php \
      -f main.php
Hello, World!
```

Запустим на KPHP:

```bash
$ kphp --mode cli --composer-root $(pwd) main.php
$ ./kphp_out/cli
Hello, World!
```

Отлично! Мы уже можем исполнять произвольные Lua-фрагменты в наших программах. Дальше будем дорабатывать свою библиотеку, делая её удобнее, эффективнее и функциональнее.

## Аллокатор памяти

Есть два способа создать `lua_State`:

* `luaL_newstate` (то, что мы использовали ранее)
* `lua_newstate`

Сигнатура у [`lua_newstate`](https://www.lua.org/manual/5.4/manual.html#lua_newstate) более сложная:

```cpp
typedef void* (*lua_Alloc) (void *ud,
                            void *ptr,
                            size_t osize,
                            size_t nsize);

lua_State *lua_newstate(lua_Alloc f, void *ud);
```

Через `lua_newstate` мы можем контролировать, как среда исполнения Lua будет выделять и очищать память. `luaL_newstate` использует для работы с памятью системный `realloc`.

Есть пара недостатков у использования стандартного аллокатора. Если скрипт получит таймаут и его работа будет прекращена до завершения Lua-скрипта, может произойти утечка памяти. Помочь избежать этого может вызов [`lua_close`](https://www.lua.org/manual/5.4/manual.html#lua_close) где-то внутри [shutdown function](https://www.php.net/manual/en/function.register-shutdown-function.php).

Другой минус — появляется отдельный пул памяти, поэтому становится сложнее подсчитывать и контролировать её потребление скриптом.

Передавая свой аллокатор, мы можем собирать статистику по аллокациям, выделять через скриптовую «кучу» (которая будет очищена после обработки запроса), а также ограничивать максимальное потребление памяти для исполняемого Lua-скрипта.

Я покажу, как можно реализовать простой аллокатор через FFI:

```php
// Наш аллокатор должен эмулировать поведение realloc.
$state = $lib->lua_newstate(function ($ud, $ptr, $orig_size, $new_size) {
    // Так как у нас нет настоящего FFI::realloc, мы будем распознавать
    // три случая: очищение памяти, выделение нового блока и
    // настоящий realloc (когда нужно выделить более крупный блок и
    // скопировать туда данные из старого блока, не забыв при этом
    // освободить ранее выделенную память). 
    if ($new_size === 0) {
        if ($orig_size !== 0 && $ptr !== null) {
            // 1. free()
            \FFI::free(\FFI::cast("uint8_t[$orig_size]", $ptr));
        }
        return null;
    }
    if ($ptr === null) {
        // 2. malloc()
        $mem = \FFI::new("uint8_t[$new_size]", false);
        return \FFI::cast('void*', \FFI::addr($mem));
    }
    // 3. realloc()
    $copy_size = ($new_size > $orig_size) ? $orig_size : $new_size;
    $mem = \FFI::new("uint8_t[$new_size]", false);
    \FFI::memcpy($mem, $ptr, $copy_size);
    \FFI::free(\FFI::cast("uint8_t[$orig_size]", $ptr));
    return \FFI::cast('void*', \FFI::addr($mem));
}, null);
```

Мы используем [`FFI::new`](https://www.php.net/manual/en/ffi.new.php) с аргументом `$owned=false`, так как мы хотим вернуть память в C, передавая владение. Другими словами, создаваемый объект не будет очищать память, когда счётчик его ссылок достигает нуля.

[`FFI::free`](https://www.php.net/manual/ru/ffi.free.php) предназначен для очищения памяти, которая выделялась с флагом `$owned=false`. Память, которой владеет PHP, очищать через `FFI::free` нельзя.

В PHP FFI нет способа сделать настоящий `realloc`, но мы можем эмулировать это поведение через комбинацию вызовов `FFI::new`, [`FFI::memcpy`](https://www.php.net/manual/ru/ffi.memcpy.php) и `FFI::free`.

Внутри функции [`lua_Alloc`](https://www.lua.org/manual/5.4/manual.html#lua_Alloc) можно разместить уместные для приложения ограничения и подсчёт статистики. В случае с KPHP при использовании своего аллокатора мы также можем отслеживать выделение памяти через [ktest](https://github.com/VKCOM/ktest)-бенчмарки. Но к ним мы вернёмся позднее.

Далее я буду считать, что у нас есть класс `MyLua`, который содержит в себе `$lib` и `$state`. В него мы будем добавлять всю новую функциональность.

```php
class MyLua {
    /** @var ffi_scope<lua> */
    public static $lib;

    /** @var ffi_cdata<lua, struct lua_State*> */
    public static $state = null;

    public static function eval(string $code) {
        // Код, через который мы выводили hello world.
    }
}
```

## Предостережения при работе с FFI::free

Есть несколько способов завалить PHP (и KPHP) в segfault через `FFI::free`. Кроме базовых правил, известных нам ещё из C, есть менее очевидные нюансы, которые легко проглядеть.

Я приведу несколько примеров, как делать точно не стоит.


```php
// ПЛОХО: вызываем free() применительно к результату FFI::addr()
$obj = FFI::new('uint64_t', false);
FFI::free(FFI::addr($obj));

// ХОРОШО: вызываем free() применительно к самому CData-объекту
$obj = FFI::new('uint64_t', false);
FFI::free($obj);
```

```php
// ПЛОХО: преобразуем массив к void* без использования addr()
$obj = FFI::new("int[$size]", false);
$ptr = FFI::cast('void*', $obj);
FFI::free($ptr);

// ХОРОШО: используем addr() при преобразовании массива к указателю
$obj = FFI::new("int[$size]", false);
$ptr = FFI::cast('void*', FFI::addr($obj));
FFI::free($ptr);
```

```php
// ПЛОХО: освобождаем массив с указанием неправильного размера
$arr = FFI::new('int[10]', false);
$arr_ptr = FFI::cast('void*', FFI::addr($arr));
$arr2 = FFI::cast('int[5]', $arr_ptr);
FFI::free($arr2);

// ХОРОШО: освобождаем массив с правильным размером
$arr = FFI::new('int[10]', false);
$arr_ptr = FFI::cast('void*', FFI::addr($arr));
$arr2 = FFI::cast('int[10]', $arr_ptr);
FFI::free($arr2);
```

## Конвертация значений из PHP в Lua

Чтобы передавать в Lua какие-то осмысленные значения, нужно научиться конвертировать PHP-значения в эквиваленты Lua. Для этого пишем метод `MyLua::php2lua`. Он принимает на вход `mixed` и пытается положить это значение в Lua-стек.

Вот таблица PHP-типов, которые будем поддерживать, а также Lua-функции для размещения этих данных в стеке:

| PHP-тип | Lua C API |
|---|---|
| null | [`lua_pushnil`](https://www.lua.org/manual/5.4/manual.html#lua_pushnil) |
| bool | [`lua_pushboolean`](https://www.lua.org/manual/5.4/manual.html#lua_pushboolean) |
| int, float | [`lua_pushnumber`](https://www.lua.org/manual/5.4/manual.html#lua_pushnumber) |
| string | [`lua_pushlstring`](https://www.lua.org/manual/5.4/manual.html#lua_pushlstring) |
| array | [`lua_createtable`](https://www.lua.org/manual/5.4/manual.html#lua_createtable), [`lua_rawset`](https://www.lua.org/manual/5.4/manual.html#lua_rawset), [`lua_rawseti`](https://www.lua.org/manual/5.4/manual.html#lua_rawseti) |

В наш заголовочный файл `lua.h` добавим вышеуказанные функции:

```cpp
typedef double lua_Number;
typedef int64_t lua_Integer;

void lua_pushnil(lua_State *L);
void lua_pushboolean(lua_State *L, int b);
void lua_pushnumber(lua_State *L, lua_Number n);
const char *lua_pushlstring(lua_State *L, const char *s, size_t len);

void lua_createtable(lua_State *L, int narr, int nrec);
void lua_rawset(lua_State *L, int index);
void lua_rawseti(lua_State *L, int index, lua_Integer i);
```

Далее я не буду акцентировать внимание на новых функциях, которые нужно добавить в `lua.h`. Процесс всегда довольно предсказуемый: если хотим использовать функцию из PHP, то находим её сигнатуру в документации и добавляем в заголовочный файл.
 
Первый набросок `php2lua` будет выглядеть так:

```php
public static function php2lua($value) {
    if (is_string($value)) {
        self::$lib->lua_pushlstring(self::$state, $value, strlen($value));
    } else if (is_int($value) || is_float($value)) {
        self::$lib->lua_pushnumber(self::$state, (float)$value);
    } else if (is_bool($value)) {
        self::$lib->lua_pushboolean(self::$state, (int)$value);
    } else if (is_array($value)) {
        // TODO: будет реализовано ниже.
    } else {
        // Какие-то непонятные значения (в том числе null),
        // будем пушить как nil; это не самое правильное решение,
        // но оно безопаснее, чем кидать исключение (читайте ниже).
        self::$lib->lua_pushnil(self::$state);
    }
}
```

Отдельно стоит сказать про исключения в контексте этой библиотеки. Поскольку мы работаем со стеком Lua, нужно быть осторожными и не оставлять его в неопределённом состоянии. Если при попытке вызова какой-то функции мы не смогли преобразовать один из аргументов, то все уже добавленные в стек аргументы должны быть удалены. То же самое верно и для всех остальных операций, которые нужно выполнять атомарно. Применять исключения можно только в том случае, если верхнеуровневые (публичные) методы всегда перехватывают стек и восстанавливают его изначальное состояние. Чтобы это работало, нужно всегда пессимистично записывать глубину стека перед исполнением логики метода, что добавляет лишние накладные расходы.

Конвертировать PHP-массив в Lua-таблицу — с одной стороны, понятная задача. Каждое значение элемента будет преобразовываться через `php2lua`. А с другой, хочется уметь создавать [sequence-like таблицы](https://www.lua.org/pil/11.1.html) для Lua, если в PHP массив был без пропусков.

```php
if (array_is_list($value)) {
    self::$lib->lua_createtable(self::$state, count($value), 0);
    $table_index = 1; // В Lua "массивах" индексы начинаются с 1
    foreach ($value as $elem) {
        self::php2lua($elem);
        self::$lib->lua_rawseti(self::$state, -2, $table_index);
        $table_index++;
    }
    return;
}
// Создаём таблицу более прямолинейным способом.
self::$lib->lua_createtable(self::$state, 0, 0);
foreach ($value as $key => $elem) {
    self::php2lua($key);
    self::php2lua($elem);
    self::$lib->lua_rawset(self::$state, -3);
}
```

## Конвертация значений из Lua в PHP

Метод `MyLua::lua2php` производит операцию, обратную `MyLua::php2lua`. `lua2php` принимает на вход индекс внутри Lua-стека и возвращает данные из этой ячейки, преобразовав их в PHP-формат. Эта функция не удаляет элемент из стека, поэтому, если требуется операция типа `pop()`, нужно сначала извлечь значение, а затем уже выполнить `stackDiscard(1)`.

```php
/**
 * @param int $n
 */
public static function stackDiscard($n) {
    // lua_pop - это макрос, поэтому используем lua_settop.
    self::$lib->lua_settop(self::$state, -($n) - 1);
}
```

Чтобы понять, что за тип данных хранится по индексу, нам потребуются константы тегов типа:

```php
public const TNIL = 0;
public const TBOOLEAN = 1;
public const TLIGHTUSERDATA = 2;
public const TNUMBER = 3;
public const TSTRING = 4;
public const TTABLE = 5;
public const TFUNCTION = 6;
public const TUSERDATA = 7;
public const TTHREAD = 8;
```

```php
/**
 * @param int $index
 * @return mixed
 */
public static function lua2php($index) {
    switch (self::$lib->lua_type(self::$state, $index)) {
    case self::TNIL:
        return null;
    case self::TBOOLEAN:
        return (bool)self::$lib->lua_toboolean(self::$state, $index);
    case self::TNUMBER:
        return self::$lib->lua_tonumberx(self::$state, $index, null);
    case self::TSTRING:
        return self::$lib->lua_tolstring(self::$state, $index, null);
    case self::TTABLE:
        return self::lua2phpTable($index);
    default:
        return ['_error' => "unsupported Lua->PHP type"];
    }
}
```

Таблицы будем возвращать как есть, без попыток распознать там sequence/array table.
 
> В библиотеке [KLua](https://github.com/quasilyte/KLua) реализован более сложный алгоритм, который может преобразовать `{"a", "b"}` в `["a", "b"]` вместо `[1 => "a", 2 => "b"]`. Но это довольно много кода с эвристиками, которые не обязательны для этой статьи.

```php
/**
 * @param int $index
 * @return mixed[]
 */
public static function lua2phpTable($index) {
    $result = [];
    // Кладём на стек первый ключ - nil.
    self::$lib->lua_pushnil(self::$state);
    while (self::$lib->lua_next(self::$state, $index) !== 0) {
        $value = self::lua2php(-1);
        self::stackDiscard(1);
        $result[self::lua2php(-1)] = $value;
        // Верхушка стека (ключ) остаётся для следующей итерации.
    }
    return $result;
}
```

`lua2php` понадобится как минимум в двух местах:
 
* Для метода `MyLua::call`, который мы скоро напишем.
* Для возвращаемых значений из скриптов, которые исполняются через `MyLua::eval`.

В методах типа `MyLua::getGlobalVar` также понадобилась бы конвертация.

## Соединяем два мира

Мы умеем конвертировать значения в обе стороны. Это пригодится, чтобы вызывать из PHP функции на Lua, получая при этом результат, с которым тоже можно работать из PHP.

Процесс вызова будет выглядеть примерно так:

1. Кладём Lua-функцию на стек.
2. Перемещаем все PHP-аргументы в стек через `php2lua`.
3. Вызываем Lua-функцию через `lua_pcallk`.
4. Результаты функции забираем со стека через `lua2php`.

Для простоты вызывать будем только глобальные функции. Вызов функции из таблицы отличается лишь тем, что нужно сначала положить в стек таблицу, а потом извлечь из неё функцию по нужному ключу.

```php
/**
 * @param string $func_name
 * @param int $num_results
 * @param mixed[] $args
 */
public static function call($func_name, $num_results, ...$args) {
    $type = self::$lib->lua_getglobal(self::$state, $func_name);
    if ($type !== self::TFUNCTION) {
        self::stackDiscard(1); // Значение переменной $func_name.
        throw new \Exception("can't find $func_name function");
    }
    foreach ($args as $arg) {
        self::php2lua($arg);
    }
    $status = self::$lib->lua_pcallk(self::$state,
        count($args), $num_results,
        0, 0, null);
    if ($status) {
        // Lua кладёт ошибку на стек.
        $err = self::lua2php(-1);
        self::stackDiscard(1);
        throw new \Exception("$func_name: $err");
    }
    return self::collectCallResults($num_results);
}

public static function collectCallResults($num_results) {
    switch ($num_results) {
        case 0:
            return null;
        case 1:
            $result = self::lua2php(-1);
            self::stackDiscard(1);
            return $result;
        default:
            // Здесь либо цикл с lua2php с добавлением в массив,
            // либо более эффективный способ с индексацией стека.
    }
}
```

Использовать это сможем так:

```php
$result = MyLua::call('type', 1, 43.5);
var_dump($result); // "number"
```

## Автоматический подсчёт $num_results

Каждый раз указывать количество результатов при вызове функции не очень удобно. К тому же некоторые функции могут возвращать разное количество результатов в зависимости от входных аргументов. Мы можем реализовать более умный способ извлечения результатов и избавиться от явного параметра `$num_results`.

Меняем сигнатуру:

```diff
- public static function call($func_name, $num_results, ...$args) {
+ public static function call($func_name, ...$args) {
```

Перед тем как положить вызываемую функцию на стек, запишем его текущую глубину:

```diff
+ $stack_top = self::$lib->lua_gettop(self::$state);
  $type = self::$lib->lua_getglobal(self::$state, $func_name);
```

В `lua_pcallk` нужно передать `MULTRET` (-1) вместо `$num_results`:

```diff
  $status = self::$lib->lua_pcallk(self::$state,
-     count($args), $num_results,
+     count($args), -1,
      0, 0, null);
```

Сразу после `lua_pcallk` мы можем вычислить количество результатов:

```diff
+ $num_results = self::$lib->lua_gettop(self::$state) - $stack_top;
  return self::collectCallResults($num_results);
```

## Вызываем PHP из Lua

Чтобы вызвать PHP-функцию из Lua, нужно передать её как [`lua_CFunction`](https://www.lua.org/manual/5.3/manual.html#lua_CFunction) в [`lua_pushcclosure`](https://www.lua.org/manual/5.3/manual.html#lua_pushcclosure) и сохранить где-нибудь (например, в глобальной переменной). Эти функции будут вызываться из внешнего контекста. В нашем случае это C-код, интерпретирующий Lua-скрипты.

При исполнении в таком внешнем контексте запрещается кидать исключения. В PHP это будет ошибкой исполнения, а в KPHP такой код просто не скомпилируется. KPHP также накладывает дополнительное ограничение: можно использовать только статические методы, глобальные функции и лямбды без замыкаемых переменных.

`lua_CFunction` — это низкий уровень абстракции. Параметры вызова мы извлекаем из стека сами, а результаты кладём в стек. Предлагаю упростить задачу и создавать обёртки для всех PHP-функций, которые хотим сделать доступными в Lua. Обёртка будет делать следующее:

1. Забирать со стека нужное количество аргументов через `lua2php`.
2. Вызывать зарегистрированную PHP-функцию.
3. Преобразовывать возвращённое значение через `php2lua` (оно попадает на стек).

При этом с точки зрения публичного API можно будет использовать замыкания с переменными.

Я покажу реализацию для PHP функций с двумя аргументами, но в реальности нам потребуются несколько схожих функций, для учёта разной арности.

Вот первая попытка:

```php
/**
 * @param string $func_name
 * @param callable(mixed,mixed):mixed $fn
 */
public static function registerFunction2($func_name, $fn) {
    self::$lib->lua_pushcclosure(self::$state, function ($s) use ($fn) {
        // 1. Извлекаем и конвертируем аргументы.
        $arg1 = self::lua2php(1);
        $arg2 = self::lua2php(2);
        // 2. Вызываем функцию.
        $result = $fn($arg1, $arg2);
        // 3. Конвертируем результат.
        self::php2lua($result);
        return 1;
    }, 0);
    // Присваиваем созданную функцию Lua переменной.
    self::$lib->lua_setglobal(self::$state, $func_name);
}
```

К сожалению, в KPHP нельзя использовать `$fn` из тела лямбды: такой код не скомпилируется. А к чему у нас есть доступ из этой лямбды? К глобальному состоянию, в частности, к статическим полям классов. Этим и воспользуемся. Добавим в `MyLua` статический массив лямбд.

```php
/** @var (callable(mixed,mixed):mixed)[] */
public static $phpfuncs2 = [];
```

Теперь можем доработать метод `registerFunction2`:

```diff
+ $id = count(self::$phpfuncs2);
+ self::$phpfuncs2[] = $fn;

- self::$lib->lua_pushcclosure(self::$state, function ($s) use ($fn) {
+ self::$lib->lua_pushcclosure(self::$state, function ($s) {
      // 1. Извлекаем и конвертируем аргументы.
      $arg1 = self::lua2php(1);
      $arg2 = self::lua2php(2);
      // 2. Вызываем функцию.
+     $fn = self::$phpfuncs2[$id];
      $result = $fn($arg1, $arg2);
      // 3. Конвертируем результат.
      self::php2lua($result);
      return 1;
  }, 0);
```

Но подождите, а как получить этот самый `$id`, чтобы найти функцию в массиве? На помощь придут [upvalues](https://www.lua.org/pil/27.3.3.html) из Lua API. Правда, прежде чем ими воспользоваться, предстоит решить загадку. Вот определение `lua_upvalueindex`:

```cpp
#if LUAI_BITSINT >= 32
#  define LUAI_MAXSTACK 1000000
#else
#  define LUAI_MAXSTACK 15000
#endif

#define LUA_REGISTRYINDEX   (-LUAI_MAXSTACK - 1000)
#define lua_upvalueindex(i) (LUA_REGISTRYINDEX - (i))
```

Это не простой макрос, ведь он зависит от константы препроцессора. А та, в свою очередь, вообще может конфигурироваться при сборке liblua.

На этот раз красиво решить задачу не получится. Лучшее, что можно сделать, это предположить для `LUAI_MAXSTACK` значение по умолчанию для 64-битных систем и предоставить пользователю возможность переопределить его, если liblua компилировался с другими параметрами.

```php
/**
 * @param int $i
 */
public static function upvalueIndex($i) {
    // $lua_max_stack - конфигурируемое значение,
    // по умолчанию равно 1000000.
    $registry_index = (-self::$lua_max_stack - 1000);
    return $registry_index - $i;
}
```

Сложная часть позади. Теперь можем написать финальный вариант `registerFunction2`.

```php
/**
 * @param string $func_name
 * @param callable(mixed,mixed):mixed $fn
 */
public static function registerFunction2($func_name, $fn) {
    // Сохраняем PHP-функцию для дальнейшего использования.
    // $id выдаём последовательные.
    $id = count(self::$phpfuncs2);
    self::$phpfuncs2[] = $fn;

    // Кладём $id на стек, чтобы сохранить его как upvalue.
    self::$lib->lua_pushnumber(self::$state, (float)$id);
    self::$lib->lua_pushcclosure(self::$state, function ($s) {
        // 1. Извлекаем и конвертируем аргументы.
        $arg1 = self::lua2php(1);
        $arg2 = self::lua2php(2);
        // 2. Вызываем функцию.
        $up_index = self::upvalueIndex(1);
        $id = (int)self::$lib->lua_tonumberx($s, $up_index, null);
        $fn = self::$phpfuncs2[$id];
        $result = $fn($arg1, $arg2);
        // 3. Конвертируем результат.
        self::php2lua($result);
        return 1;
    }, 1); // Обратите внимание: теперь у нас 1 upvalue, а не 0.

    // Присваиваем созданную функцию Lua-переменной.
    self::$lib->lua_setglobal(self::$state, $func_name);
}
```

Попробуем это всё в деле!

```php
MyLua::registerFunction2('phpconcat', function ($s1, $s2) {
    return $s1 . $s2;
});

$result = MyLua::call('phpconcat', 'a', 'b');
var_dump($result); // "ab"
```

Это выглядит так естественно, что даже не задумываешься о том, какой путь проделали строчки `"a"` и `"b"`, прежде чем мы распечатали их вместе как `"ab"`.
 
1. Сначала преобразовали PHP-строки в Lua-строки через `php2lua`.
2. Затем аргументы `phpconcat` из Lua-строк превратились в PHP-строки.
3. Функция `phpconcat` приняла PHP-строку и вернула PHP-строку.
4. Наша обёртка преобразовала результат из PHP-строки в Lua-строку.
5. И в самом конце `MyLua::call` преобразовала результат в PHP-строку.

Вызывать PHP-функции через `MyLua::call` смысла особого нет, а вот внутри полноценных скриптов это уже гораздо полезнее.

```php
MyLua::eval('
    print(phpconcat("a", "b"))
');
```

Здесь мы делаем почти то же самое, но строки изначально создаются в Lua-контексте. Да и результат не нужно преобразовывать в PHP-значения.

Заметим, что ограничения на замыкаемое лямбдами состояние в нашем API теперь нет:

```php
class MyContext {
    public $value = 0;
}

$context = new MyContext();
MyLua::registerFunction0('next_id', function () use ($context) {
    return $context->value++;
});

MyLua::eval('
    print(next_id()); -- 0
    print(next_id()); -- 1
');
```

## Ограничиваем доступ к стандартной библиотеке Lua

Ранее мы всегда использовали `luaL_openlibs` для загрузки стандартной библиотеки Lua. Это не всегда предпочтительный способ, так как он подключает абсолютно все библиотеки. Перед тем как перейдём к выборочной загрузке стандартной библиотеки для Lua, рассмотрим более простой способ. Допустим, вы хотите заменить функцию `print`, чтобы скрипты писали не в `stdout`, а в ваш буфер. Для этого достаточно заменить глобальную переменную `print`. Как известно, `MyLua::registerFunction` записывает функцию в глобальную переменную — этим и воспользуемся.

```php
class LuaLogger {
    public $messages = [];

    public function doPrint($arg) {
        $this->messages[] = $arg;
        return null;
    }
}

$logger = new LuaLogger();

// Методы вместе с замыкаемыми объектами использовать тоже можно.
KLua::registerFunction1('print', [$logger, 'doPrint']);

// Вызовы print из Lua теперь добавляют сообщения в
// массив $logger->messages.
KLua::eval('
    print(1)
    print("hello")
');
```

Вернёмся к предыдущей задаче. Перечислим модули, которые есть в стандартной библиотеке Lua:

| Название модуля | Функция-загрузчик |
|---|---|
| `base` (заполняет `_G`) | `luaopen_base` |
| `package` | `luaopen_package` |
| `coroutine` | `luaopen_coroutine` |
| `table` | `luaopen_table` |
| `io` | `luaopen_io` |
| `os` | `luaopen_os` |
| `string` | `luaopen_string` |
| `math` | `luaopen_math` |
| `utf8` | `luaopen_utf8` |
| `debug` | `luaopen_debug` |

Уже известный нам `luaL_openlibs` делает [`luaL_requiref`](https://www.lua.org/manual/5.4/manual.html#luaL_requiref) для каждого из модулей. Если дать пользователю возможность выбрать массив подключаемых модулей, то мы сможем реализовать выборочную инициализацию.

Для начала попробуем подключить модуль `base` без `luaL_openlibs`:

```php
self::$lib->luaL_requiref(
    self::$state, "_G", self::$lib->luaopen_base, 1);
```

Если запустим этот код, PHP может быть недоволен:

```
# Я отформатировал сообщение ошибки для простоты восприятия.
FFI\Exception:
    Passing incompatible argument 3 of C function 'luaL_requiref',
    expecting 'int32_t(*)()',
    found     'int32_t(*)()'
```

![](https://habrastorage.org/webt/xt/lu/t2/xtlut2xuun4mvqyujd_esmnjwvy.jpeg)

Рабочим вариантом будет введение дополнительной лямбды:

```php
self::$lib->luaL_requiref(self::$state, "_G", function ($s) {
    return self::$lib->luaopen_base($s);
}, 1);
```

Я приведу фрагмент метода загрузки всех модулей по имени, но опущу однотипную часть:

```php
// Где-то около инициализации lua_State.
if ($config->preload_stdlib !== null) {
    foreach ($config->preload_stdlib as $lib_name) {
        self::openLib($lib_name);
    }
} else {
    self::$lib->luaL_openlibs(self::$state);
}

/**
 * @param string $lib_name
 */
private static function openLib($lib_name) {
    switch ($lib_name) {
    case "base":
        self::$lib->luaL_requiref(self::$state, "_G", function ($s) {
            return self::$lib->luaopen_base($s);
        }, 1);
        break;
    case "package":
        self::$lib->luaL_requiref(self::$state, $lib_name, function ($s) {
            return self::$lib->luaopen_package($s);
        }, 1);
        break;
    case "coroutine":
        // Аналогично...

    // + все оставшиеся модули из списка выше.

    default:
        throw new \Exception("can't load $lib_name");
    }
    self::stackDiscard(1); // lib
}
```

## Предоставляем плагинам красивый SDK

`MyLua::registerFunction` позволяет регистрировать глобальные функции. При этом мы можем предоставить красивый доступ к этим функциям через таблицу, загружая перед плагинами наш собственный Lua-скрипт.

```php
// Мы будем использовать префикс php_, чтобы избежать
// возможных коллизий имён.
MyLua::registerFunction2('php_preg_match', function ($pat, $s) {
    return preg_match($pat, $s) === 1;
});

// Наш скрипт с таблицами будет загружаться до
// пользовательского кода.
MyLua::eval('
    pcre = {}

    function pcre.match(pat, s)
        return php_preg_match(pat, s)
    end
');

// Пользовательский код может использовать функции
// через таблицу pcre.
MyLua::eval('
    print(pcre.match("/[0-9]+/", "abc")) -- true
    print(pcre.match("/[0-9]+/", "435")) -- false
');
```

Альтернативный путь — добавлять в интерфейс нашей библиотеки дополнительные способы регистрации PHP-функций. Например, дополнительным аргументом мы могли бы принимать имя глобальной таблицы, в которую стоит добавить новую функцию.

## Тюним производительность

Самый простой способ проверить производительность кода на PHP или KPHP — это запустить бенчмарк [ktest](https://github.com/VKCOM/ktest). Его можно установить при помощи composer:

```bash
$ composer require --dev vkcom/ktest-script
```

Сразу же проверяем, что всё хорошо:

```bash
$ ./vendor/bin/ktest --help
Usage:

  ktest COMMAND

Possible commands are:

  phpunit         run phpunit tests using KPHP
  compare         test that KPHP and PHP scripts output is identical
  benchstat       compute and compare statistics about benchmark results
  bench           run benchmarks using KPHP
  bench-ab        run two selected benchmarks using KPHP, compare results
  bench-php       run benchmarks using PHP
  bench-vs-php    run benchmarks using KPHP and PHP, compare results
  env             print ktest-related env variables
  version         print ktest version info

Run 'ktest COMMAND -h' to see more information about a command.
```

Создадим файл `benchmarks/BenchmarkMyLua.php`:

```php
<?php

class BenchmarkMyLua {
    public function __construct() {
        if (KPHP_COMPILER_VERSION) {
            FFI::load(__DIR__ . '/lua.h');
        }

        MyLua::init();

        MyLua::registerFunction2('phpconcat', function ($x, $y) {
            return $x . $y;
        });
        MyLua::registerFunction2('phpmin', function ($x, $y) {
            return min($x, $y);
        });
    }

    public function benchmarkCall2PHPMin() {
        return MyLua::call('phpmin', 1, 2);
    }

    public function benchmarkCall2PHPConcat() {
        return MyLua::call('phpconcat', 'a', 'b');
    }

    public function benchmarkEvalPHPConcat() {
        return MyLua::eval('return phpconcat("a", "b")');
    }
}
```

Эти бенчмарки можно запускать в нескольких режимах. Рассмотрим самые интересные.

Запуск через KPHP:

```bash
$ ./vendor/bin/ktest bench --benchmem ./benchmarks
class: BenchmarkMyLua
BenchmarkMyLua::Call2PHPMin	126440	507.0 ns/op	0 B/op	0 allocs/op
BenchmarkMyLua::Call2PHPConcat	68980	924.0 ns/op	32 B/op	2 allocs/op
BenchmarkMyLua::EvalPHPConcat	13260	7580.0 ns/op	1662 B/op	56 allocs/op
ok BenchmarkMyLua 972.144303ms
```

Для KPHP доступен флаг `--benchmem`, который добавляет в результаты бенчмарков информацию о том, сколько памяти было выделено. Как видите, вызывать Lua через `MyLua::call` гораздо быстрее, чем через `eval`. Как минимум, не нужно парсить и компилировать исходники в байт-код. У ваших плагинов, скорее всего, будет понятная точка входа, вроде функции `run` или `main`, поэтому запускать их рекомендуется именно через `MyLua::call`. При этом точка входа может быть автоматически сгенерирована вами, чтобы правильно изолировать окружение (`_ENV`) плагинов.

Запуск через PHP (пока нет поддержки для `--benchmem`):

```bash
$ php --version
PHP 8.1.8 (cli) (built: Jul 11 2022 08:29:57) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.8, Copyright (c) Zend Technologies
    with Zend OPcache v8.1.8, Copyright (c), by Zend Technologies

$ ./vendor/bin/ktest bench-php --preload preload.php ./benchmarks
class: BenchmarkMyLua
BenchmarkMyLua::Call2PHPMin	5260	9986.0 ns/op
BenchmarkMyLua::Call2PHPConcat	8120	10120.0 ns/op
BenchmarkMyLua::EvalPHPConcat	1460	74153.0 ns/op
ok BenchmarkMyLua 456.230781ms
```

По умолчанию для PHP8 тесты запускаются с такими настройками JIT:

```ini
opcache.enable_cli=1
opcache.jit_buffer_size=96M
opcache.jit=on
```

При желании можно гонять PHP8 без JIT:

```bash
$ ./vendor/bin/ktest bench-php --no-jit --preload preload.php ./benchmarks
class: BenchmarkMyLua
BenchmarkMyLua::Call2PHPMin	6840	9413.0 ns/op
BenchmarkMyLua::Call2PHPConcat	7880	9980.0 ns/op
BenchmarkMyLua::EvalPHPConcat	1460	79526.0 ns/op
ok BenchmarkMyLua 442.669261ms
```

А ещё можно запустить режим сравнения KPHP-vs-PHP:

```bash
# Разбил команду на две строки, чтобы уместить по ширине.
$ ./vendor/bin/ktest bench-vs-php --geomean\
    --preload preload.php ./benchmarks
name                   PHP time/op  KPHP time/op  delta
MyLua::Call2PHPMin     9.35µs ± 1%  0.49µs ± 0%  -94.71%  (p=0.000 n=9+9)
MyLua::Call2PHPConcat  10.3µs ± 4%   0.9µs ± 2%  -91.36%  (p=0.000 n=10+10)
MyLua::EvalPHPConcat   79.3µs ± 1%   7.5µs ± 0%  -90.53%  (p=0.000 n=10+9)
[Geo mean]             19.7µs        1.5µs       -92.43%
```

Как видите, PHP FFI действительно не очень эффективен, и JIT здесь помочь пока не способен. Любопытно посмотреть, как изменится ситуация, когда JIT начнёт оптимизировать подобный код.

В KPHP вызовы FFI относительно быстрые. Можно даже не задумываться о накладных расходах, если только речь идёт не о простейших getter-функциях. Мы можем измерить затраты на вызов с помощью бенчмарка.

```php
<?php

class BenchmarkFFI {
    /** @var ffi_scope<lua> */
    private $lib;
    /** @var ffi_cdata<lua, struct lua_State*> */
    private $state;

    public function __construct() {
        if (KPHP_COMPILER_VERSION) {
            FFI::load(__DIR__ . '/../src/lua.h');
        }
        $this->lib = FFI::scope('lua');
        $this->state = $this->lib->luaL_newstate();
    }

    public function benchmarkGettop() {
        return $this->lib->lua_gettop($this->state);
    }
}
```

```bash
$ ./vendor/bin/ktest bench ./benchmarks/BenchmarkFFI
class: BenchmarkFFI
BenchmarkFFI::Gettop	689660	17.0 ns/op
ok BenchmarkFFI 260.789974ms
```

Около 17 наносекунд на вызов `lua_gettop`. Неплохо, но можно лучше.
 
Дело в том, что компилятор KPHP генерирует критические секции для каждого FFI-вызова. Так он защищается от неприятностей, которые могут возникнуть в случае вызова произвольного нативного кода. Для простейших и безопасных функций вроде `lua_gettop` мы можем применить в `lua.h`-файле специальную аннотацию, которая отключит эти критические секции для выбранной функции.

```diff
+ // @kphp-ffi-signalsafe
  int lua_gettop(lua_State *L);
```

Запустим бенчмарк ещё раз:

```bash
$ ./vendor/bin/ktest bench ./benchmarks/BenchmarkFFI
class: BenchmarkFFI
BenchmarkFFI::Gettop	862080	6.0 ns/op
ok BenchmarkFFI 253.012341ms
```

Примерно 6 наносекунд! Это хороший результат. Накладные расходы на критическую секцию близки к константе — около 10 наносекунд. В зависимости от паттернов использования это может быть много или мало. Чаще всего — капля в море. Для `lua_gettop` считаю эту оптимизацию оправданной.

## Производительность расширений по сравнению с FFI в PHP

Для сравнения посмотрим, что там у PHP:

```bash
$ ./vendor/bin/ktest bench-vs-php --preload preload.php\
    ./benchmarks/BenchmarkFFI.php 
name         PHP time/op  KPHP time/op  delta
FFI::Gettop   301ns ± 1%     6ns ± 0%  -98.00%  (p=0.000 n=8+10)
```

Текущая поддержка FFI в PHP имеет довольно высокие накладные расходы на взаимодействие между PHP и внешним кодом. Однако после выполнения вызова имеем ту же производительность, с которой работают нативные библиотеки.
 
Это ограничивает применимость, потому что какую-нибудь вспомогательную математическую библиотеку использовать будет уже не так приятно: больше половины времени исполнения может прийтись на вызов функции, а не на её работу. Однако расстраиваться по этому поводу рано, ведь проблема на радаре у нескольких людей и можно рассчитывать на улучшения.
 
Для некоторых ситуаций даже 300 наносекунд на вызов — не катастрофа. Например, если функция выполняет какую-то значительную работу, то мы даже не заметим эти лишние 0,0000003 секунды.
 
В нашем случае планируем исполнять скрипты на Lua. Высока вероятность, что даже не заметим влияния FFI на производительность. Предлагаю устроить сравнение с PHP-расширением и посмотреть, сможем ли мы увидеть разницу.

Мы будем запускать [spectral norm](https://benchmarksgame-team.pages.debian.net/benchmarksgame/program/spectralnorm-lua-1.html) с параметром N=25 (это очень мало). Для `use_ffi_allocator=false` результаты будут следующими:

```
name                ext time/op  ffi time/op  delta
LuaExtension::Eval  3.45ms ± 1%  3.47ms ± 1%  +0.53%  (p=0.002 n=10+10)
```

Считаю это идентичной производительностью. Обе реализации запускают Lua-интерпретатор и используют [стандартный аллокатор](https://github.com/laruence/php-lua/blob/a6c5162a7cbdf4b48b104f303acc022a3f93caf2/lua.c#L196) (из glibc).
 
Можем сравнить и FFI-менеджер памяти:

```
name                ext time/op  ffi time/op  delta
LuaExtension::Eval  3.45ms ± 1%  3.93ms ± 0%  +13.81%  (p=0.000 n=10+10)
```

## Поддержка light userdata

Ранее мы обошли тип [light userdata](https://www.lua.org/pil/28.5.html) стороной. Реализовать его поддержку довольно сложно, но он может быть полезен. Предположим, у нас есть большой массив данных. Если нам потребуется перемещать его из PHP в Lua и обратно, то это будет копирование большого количества данных при каждом таком преобразовании. Light userdata позволяет нам написать представление, которое будет использоваться в обоих языках без неявной конвертации. Так мы полностью избегаем копирования.

Здесь нужно понимать, что массив будет храниться в виде C-данных, а не как PHP-массив. Это значит, что как минимум один раз придётся перекопировать данные при создании C-массива из нашего PHP-массива.
 
Для начала опишем наш вспомогательный класс для userdata:

```php
class UserData {
    /** @var ffi_scope<lua_userdata> */
    public static $lib;

    public static function init() {
        self::$lib = FFI::cdef('
            #define FFI_SCOPE "lua_userdata"
            struct ContextData {
                int important_data[100];
            };
        ');
    }

    /** @return ffi_cdata<lua_userdata, struct ContextData> */
    public static function newContextData($important_data) {
        $ctx = self::$lib->new('struct ContextData');
        for ($i = 0; $i < count($ctx->important_data); $i++) {
            ffi_array_set($ctx->important_data, $i, $i * 2);
        }
        return $ctx;
    }
}
```

Теперь нужно доработать `lua2php`, чтобы обрабатывался новый тип:

```php
case self::TLIGHTUSERDATA:
    $void_ptr = self::$lib->lua_touserdata(self::$state, $index);
    return ffi_cast_ptr2addr($void_ptr);
```

`lua2php` возвращает `mixed`. В KPHP нельзя просто так совместить тип экземпляра класса и `mixed`, поэтому `mixed|CData` нам не подходит. Так что полученный указатель `void*` мы превращаем в числовое значение типа `int`, которое будет хранить адрес. Его потом можно будет использовать для восстановления указателя [`CData`](https://www.php.net/manual/ru/class.ffi-cdata.php).

```php
KLua::registerFunction2('ctx_get', function ($ctx_addr, $index) {
    // userdata-аргументы передаются как PHP int.
    // Нам нужно получить указатель по этому адресу,
    // для этого мы используем ffi_cast_addr2ptr.
    $ptr = ffi_cast_addr2ptr((int)$vec_addr);
    // Так как $ptr - это void*, нам нужно выполнить
    // ещё один cast перед тем, как использовать этот указатель.
    $ctx = UserData::$lib->cast('struct ContextData*', $ptr);
    return ffi_array_get($ctx->important_data, $index);
});
```

Самый простой способ передать в Lua значение userdata - через глобальную переменную.

```php
/**
 * @param string $var_name
 * @param ffi_cdata<C, void*>
 */
public static function setVarUserData($var_name, $ptr) {
    self::$lib->lua_pushlightuserdata(self::$state, $ptr);
    self::$lib->lua_setglobal(self::$state, $var_name);
}
```

```php
UserData::init();
$ctx = UserData::newContextData();
MyLua::setVarUserData('global_ctx', FFI::addr($ctx));
```

Для Lua значения userdata непрозрачны, поэтому всё, что мы можем сделать с ними, это передавать их в функции, предоставляемые встраиваемым приложением.

```php
MyLua::eval('
    print(ctx_get(global_ctx, 0)) -- 0.0
    print(ctx_get(global_ctx, 1)) -- 2.0
    print(ctx_get(global_ctx, 2)) -- 4.0
');
```

Преобразование между Lua и PHP теперь практически бесплатное, никаких копирований массивов.

Как вы могли заметить, я использовал странные функции типа `ffi_array_set` и `ffi_cast_addr2ptr`. Они встроены в KPHP и реализуют некоторые вариации FFI-операций; в PHP они доступны через [kphp-polyfills](https://github.com/VKCOM/kphp-polyfills).

## KLua

![](https://habrastorage.org/webt/91/ku/eo/91kueo4r2em2cwlymh5ff8bqv74.png)

Пакет [quasilyte/klua](https://github.com/quasilyte/KLua) реализует всё то, о чём мы говорили выше, и даже больше. Работает как для PHP, так и для KPHP. Теперь содержимое библиотеки и её API не должны показаться вам чем-то необычным. Установить этот пакет можно через composer:

```bash
$ composer require quasilyte/klua
```

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use KLua\KLua;
use KLua\KLuaConfig;

if (KPHP_COMPILER_VERSION) { KLua::loadFFI(); }

KLua::init(new KLuaConfig());

KLua::eval('
    function example(x)
        return x + 1
    end
');

var_dump(KLua::call('example', 10)); // => 11
```

KLua тестировалась с Lua версий 5.2, 5.3 и 5.4.

Примеры использования библиотеки KLua:
* [simple.php](https://github.com/quasilyte/KLua/blob/master/examples/1_simple.php) — базовый hello world.
* [phpfunc.php](https://github.com/quasilyte/KLua/blob/master/examples/2_phpfunc.php) — пример с `registerFunction`.
* [override_print.php](https://github.com/quasilyte/KLua/blob/master/examples/3_override_print.php) — переопределение `print` в Lua.
* [limited_stdlib.php](https://github.com/quasilyte/KLua/blob/master/examples/4_limited_stdlib.php) — ограничение подгружаемой стандартной библиотеки.
* [plugin_sandbox.php](https://github.com/quasilyte/KLua/blob/master/examples/5_plugin_sandbox.php) — загрузка плагинов с изоляцией.
* [phpfunc_table.php](https://github.com/quasilyte/KLua/blob/master/examples/6_phpfunc_table.php) — как обернуть PHP-функции в таблицу.
* [userdata.php](https://github.com/quasilyte/KLua/blob/master/examples/7_userdata.php) — примеры использования light userdata.
* [memory_limit.php](https://github.com/quasilyte/KLua/blob/master/examples/8_memory_limit.php) — как ограничивать память, доступную Lua-скриптам.
* [time_limit.php](https://github.com/quasilyte/KLua/blob/master/examples/9_time_limit.php) — как ограничить время исполнения запускаемых Lua-скриптов.

## Полезные источники

* [KLua](https://github.com/quasilyte/KLua) — FFI-библиотека для liblua5.
* [t.me/kphp_chat](https://t.me/kphp_chat) — чатик open-source сообщества KPHP.
* [awesome-kphp](https://github.com/quasilyte/awesome-kphp/)
* Статья [«Используем SQLite в KPHP и PHP через FFI»](https://habr.com/ru/post/653677/).
* Статья [«Создаём игру на KPHP с помощью FFI и SDL»](https://habr.com/ru/company/vk/blog/581238/).
* Статья [«Заметки KPHP: тестирование и бенчмарки»](https://habr.com/ru/company/vk/blog/572424/).
* Доклад про [KPHP FFI](https://speakerdeck.com/quasilyte/kphp-ffi).
* [Официальная документация KPHP FFI](https://vkcom.github.io/kphp/kphp-language/php-extensions/ffi.html).
