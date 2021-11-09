# Заметки KPHP: тестирование и бенчмарки

Перед вами первая статья из серии "Как использовать KPHP в open source?"

В рамках этой серии мы будем разбирать разные аспекты работы с [KPHP](https://habr.com/ru/company/vk/blog/527420/), расширяя ту информацию, что вы можете найти в [официальной документации](https://vkcom.github.io/kphp/).

В сегодняшнем выпуске:

* Базовое использование [composer](https://getcomposer.org/) с KPHP
* Как писать и запускать unit-тесты для KPHP
* Бенчмаркинг KPHP-кода (профилирование мы затронем в другой раз)
* Как правильно сравнивать результаты бенчмарков

<cut/>

## Предисловие

На данный момент использовать KPHP для настоящих "боевых" задач может быть проблематично по нескольким причинам:

* Нет готовых решений для работы с популярными базами данных
* Примеров и обучающего материала мало, осваивать особенности тулинга сложно
* Очень малое количество PHP-библиотек будет работать с KPHP без модификаций

Наша команда старается сделать так, чтобы каждый из этих аспектов со временем становился менее ярко выраженным. Мы хотим сделать возможным эффективное и удобное использование KPHP за пределами [ВК](https://vk.com): в компилятор и рантайм добавляются недостающие фичи, а статьи, вроде этой, могут помочь закрыть пробелы в документации.

## Подготавливаемся к работе

Сперва нам нужно [установить KPHP](https://vkcom.github.io/kphp/kphp-basics/installation.html). 

> Если у вас есть такая возможность, рекомендую ставить KPHP из [deb-пакета](https://vkcom.github.io/kphp/kphp-basics/installation.html#install-kphp-from-deb-packages). Работать через Docker-контейнер может быть менее удобно.

```bash
# Качаем Docker-образ
$ docker pull vkcom/kphp
```

Если мы разрабатываем в директории `~/kphp-uuid`, то эту директорию нужно будет пробросить в контейнер при его запуске.

```bash
# Запускаем контейнер, подключаемся к нему
$ docker run -ti -v ~/kphp-uuid/:/tmp/kphp-uuid:rw -p 8080:8080 vkcom/kphp
```

Внутри контейнера вам будет доступна команда `kphp`.

```bash
$ kphp --version
kphp2cpp compiled at ...
```

По ходу статьи нам понадобится утилита [ktest](https://github.com/quasilyte/ktest) внутри контейнера. Я предлагаю положить бинарник `ktest` в директорию `~/kphp-uuid`, которую мы уже подключаем к контейнеру.

Далее по ходу статьи я буду делать вид, что `kphp` у вас установлен локально, хотя, скорее всего, вы будете его использовать из Docker-контейнера.

## Создаём минимальный проект

Подготовим директорию, в которой мы будем работать:

```bash
$ mkdir kphp-uuid
$ cd kphp-uuid
```

Любой уважающий себя `[K]PHP` пакет хочет дружить с composer, поэтому нам нужен `composer.json` в самом корне:

```json
{
    "name": "quasilyte/kphp-uuid",
    "description": "A simple UUID generation package for KPHP",
    "type": "library",
    "license": "MIT",
    "autoload": {
        "psr-4": {"Quasilyte\\KPHP\\Uuid\\": "src/"}
    },
    "require": {}
}
```

> KPHP так же поддерживает `autoload.files`. Запланировано добавить поддержку `autoload.psr-0` и `autoload.classmap` в будущем. Если вам нужны эти возможности прямо сейчас, [дайте нам знать](https://github.com/VKCOM/kphp/issues/49).

Как можно догадаться по описанию, наш пакет будет предоставлять функции создания [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier). Основной наш класс мы разместим в файле `src/UUID.php`:

```php
<?php

namespace Quasilyte\KPHP\Uuid;

class UUID {
    // Реализация позаимствована отсюда:
    // https://www.php.net/manual/en/function.uniqid.php#94959
    // Оригинальный код написан Andrew Moore в качестве
    // занимательного примера как это можно сделать на "чистом PHP".
    public static function v4(): string {
        return sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
            // 32 bits for "time_low"
            mt_rand(0, 0xffff), mt_rand(0, 0xffff),
            // 16 bits for "time_mid"
            mt_rand(0, 0xffff),
            // 16 bits for "time_hi_and_version",
            // four most significant bits holds version number 4
            mt_rand(0, 0x0fff) | 0x4000,
            // 16 bits, 8 bits for "clk_seq_hi_res",
            // 8 bits for "clk_seq_low",
            // two most significant bits holds zero and
            // one for variant DCE1.1
            mt_rand(0, 0x3fff) | 0x8000,
            // 48 bits for "node"
            mt_rand(0, 0xffff), mt_rand(0, 0xffff), mt_rand(0, 0xffff)
        );
    }
}
```

> **Внимание**: приведённая в примере выше генерация UUIDv4 не должна использоваться в реальных приложениях. Этот код стоит рассматривать только как компактный и удобный пример для демонстрации.

Библиотека [выложена на packagist](https://packagist.org/packages/quasilyte/kphp-uuid), поэтому её можно установить в своих PHP/KPHP программах.

## Устанавливаем и используем kphp-uuid

Теперь создадим отдельный проект, в котором установим `kphp-uuid` и запустим наш hello world, печатающий сгенерированный UUIDv4.

```bash
$ mkdir kphp-helloworld
$ cd kphp-helloworld

# Либо можете запустить `composer init`
$ echo '{}' > composer.json

$ composer require quasilyte/kphp-uuid:dev-master
```

В самом корне положим наш `main.php`:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use Quasilyte\KPHP\Uuid\UUID;

var_dump(UUID::v4());
```

Соберём исполняемый файл:

```bash
$ kphp --composer-root=$(pwd) --mode=cli main.php
```

* `--composer-root` включает composer-режим. Аргументом передаём корень проекта
* `--mode` указывает что мы собираем; `cli` подходит для простых скриптов

По умолчанию результат компиляции размещается в `./kphp_out`.

Мы собирали с режимом `cli`, поэтому созданный бинарник можно найти в `./kphp_out/cli`. Запустим его:

```bash
$ ./kphp_out/cli
string(36) "9f21785c-01a5-44aa-bc72-5fe262c10ca6"
```

Вы так же можете запускать этот main файл как обычный PHP-скрипт:

```bash
$ php -f main.php
string(36) "489ee573-5f1a-4b1f-8090-162029e0f428"
```

## PHPUnit + PHP

Вернёмся к нашему коду в `kphp-uuid`.

Теперь, когда мы научились писать и запускать простой код на KPHP, мы можем приступить к главной теме этой статьи - тестированию.

Так как код на KPHP легко запускать через `php`, естественной идеей является тестирование KPHP-кода через [phpunit](https://phpunit.de/). Это действительно рабочая стратегия, поэтому начнём мы именно с неё.

Тесты мы будем размещать внутри директории `tests`. Следуя правилам хорошего тона, тестовый класс для `UUID` мы разместим в`tests/UUIDTest.php`:

```php
<?php

use PHPUnit\Framework\TestCase;
use Quasilyte\KPHP\Uuid\UUID;

class UUIDTest extends TestCase {
    // Проверяем уникальность сгенерированных значений
    public function testUnique() {
        $set = [];
        $n = 10;
        for ($i = 0; $i < $n; $i++) {
            $set[UUID::v4()] = true;
        }
        $this->assertSame(count($set), $n);
    }

    public function testLength() {
        $this->assertSame(strlen(UUID::v4()), 36);
    }

    // Проверяем что UUID генерируются валидные
    public function testValid() {
        // Sanity test на то что метод isValid может
        // определить заведомо валидные/плохие строки 
        $example = 'f37ac10b-58cc-4372-a567-0e02b2c3d170';
        $this->assertTrue(self::isValid($example));
        $this->assertFalse(self::isValid('foo'));

        // А теперь проверим сгенерированные значения
        for ($i = 0; $i < 100; $i++) {
            $this->assertTrue(self::isValid(UUID::v4()));
        }
    }

    private static function isValid(string $uuid): bool {
        $w = '[0-9a-f]';
        $pattern = "/^$w{8}-$w{4}-4$w{3}-[89ab]$w{3}-$w{12}$/";
        return preg_match($pattern, $uuid) === 1;
    }
}
```

Теперь мы можем запустить тесты привычным для нас образом:

```bash
$ phpunit --bootstrap vendor/autoload.php tests

OK (3 tests, 104 assertions)
```

> Я разместил тестовый класс `UUIDTest` в глобальном пространстве имён. Есть и [другие способы](https://stackoverflow.com/questions/12117254/organizing-phpunit-tests-in-namespaces), но нет единственно правильного.

Мы протестировали наш KPHP-код как обычный PHP-код. Чаще всего этого достаточно, ведь в большинстве случаев результаты будут идентичными.

Однако, есть несколько преимуществ запуска тестов на настоящем KPHP:

1. Вы можете раньше обнаружить некоторые неожиданные различия в поведении PHP и KPHP
2. Вы получаете простой способ проверять на CI то, что ваш код всё ещё успешно компилируется
3. Сам код тестов будет проверяться компилятором, а мы знаем, что он - наш лучший друг

## Встречайте - ktest

[ktest](https://github.com/quasilyte/ktest) - это утилита командной строки для тестирования корректности и производительности KPHP программ.

Проще всего её установить, скачав [релизный бинарник](TODO) последней версии.

Однако если в вашей системе установлен [Go](golang.org/), то можно установить ktest и из исходников:

```bash
# Традиционный способ (до Go 1.17)
$ go get github.com/quasilyte/ktest/cmd/ktest

# Новый способ (Go 1.17+)
$ go install github.com/quasilyte/ktest/cmd/ktest
```

Если ваш `$(go env GOPATH)/bin` добавлен в системный `$PATH`, то команда ktest сразу же станет доступной:

```bash
$ ktest --help
Usage:

  ./ktest COMMAND

Possible commands are:

  phpunit         run phpunit tests using KPHP
  benchstat       compute and compare statistics about benchmark results
  bench           run benchmarks using KPHP
  bench-php       run benchmarks using PHP
  bench-vs-php    run benchmarks using both KPHP and PHP, compare results
  env             print ktest-related env variables information

Run './ktest COMMAND -h' to see more information about a command.
```

## ktest: тестируем KPHP вместе с PHP

В наш пакет `kphp-uuid` нужно будет установить вспомогательный пакет [kphpunit](https://github.com/quasilyte/kphpunit):

```bash
$ composer require --dev quasilyte/kphpunit:dev-master
```

Использовать его напрямую нам не нужно. Этот пакет требуется для работы создаваемого утилитой ktest тестирующего кода.

Теперь мы готовы запустить те же самые тесты на KPHP:

```bash
$ ktest phpunit tests

OK (3 tests, 104 assertions)
```

Как это работает:

* Каждый файл с тестами анализируется утилитой ktest
* Для каждого из классов генерируется модифицированная версия (*)
* Затем для каждого такого класса создаётся отдельный скрипт-точка входа
* Результат работы скрипта анализируется утилитой ktest

> (*) Заменяем `PHPUnit\Framework\TestCase` -> `KPHPUnit\Framework\TestCase`
> и выполняем некоторые другие преобразования.

Формат вывода у `ktest phpunit` очень близок к обычному `phpunit`.

## ktest: бенчмарки для KPHP

Допустим, вы решили написать немного более оптимизированную версию метода `v4()`. В ней вы используете меньше вызовов `mt_rand`. Получилось что-то такое:

```php
public static function v4fast(): string {
    return sprintf('%08x-%04x-%04x-%04x-%08x%04x',
        mt_rand(0, 0xffffffff),
        mt_rand(0, 0xffff),
        mt_rand(0, 0x0fff) | 0x4000,
        mt_rand(0, 0x3fff) | 0x8000,
        mt_rand(0, 0xffffffff), mt_rand(0, 0xffff)
    );
}
```

Тесты добавлены, вроде бы работает нормально. Но насколько мы реально ускорили код?

Команда `ktest bench` позволяет найти ответ на этот вопрос.

Тесты производительности (бенчмарки) пишутся в похожем на тесты стиле: создаём особый класс, описываем там методы. В своём примере я размещу бенчмарки в директории `benchmarks` в корне проекта.

Содержимое файла `benchmarks/UUIDBenchmark.php`:

```php
<?php

use Quasilyte\KPHP\Uuid\UUID;

class UUIDBenchmark {
    public function benchmarkV4() {
        return UUID::v4();
    }

    public function benchmarkV4Fast() {
        return UUID::v4fast();
    }
}
```

Обратите внимание:

* Название файла должно соответствовать названию класса
* Название класса должно иметь суффикс `Benchmark`
* Название каждого метода-бенчмарка должно иметь префикс `benchmark`
* Все методы-бенчмарки нестатические

Запустим бенчмарки:

```bash
$ ktest bench ./benchmarks/UUIDBenchmark.php
BenchmarkV4      72553   5507.0 ns/op
BenchmarkV4Fast  180310  4112.0 ns/op
```

> Как мы видим, `v4fast()` занимает меньше времени, чем `v4()`, но это только предварительные результаты. Более точный анализ мы проведём немного позднее.

Вы можете использовать статическое (и не только) состояние класса внутри кода бенчмарков. Например, тестовые данные можно держать внутри `static` поля класса.

Код бенчмарка должен выполнять именно ту операцию, которую вы хотите протестировать, ровно один раз (вам не нужно самим добавлять циклы). Старайтесь минимизировать использование хелпер-методов.

Если вам нужно подготовить входные данные для бенчмарков, делайте это в конструкторе, а не внутри метода-бенчмарка.

<spoiler title="Уменьшаем количество шума"><hr>

Настройка своей машины для стабильной работы бенчмарков - это тема для отдельного рассказа. Но я раскрою один простой лайфхак.

Для Intel процессоров вы захотите выключить Turbo Boost.

На некоторых Линуксах это делается, например, вот так:

```bash
$ echo "1" | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
```

Если разброс времени исполнения выше, чем 3-5%, то проблема либо в плохо настроенном окружении, либо в самом коде бенчмарка. Желательно достигать разброса в 1-2%.

Если вы сравниваете результаты бенчмарков, делать это корректно только если все наборы результатов были получены на одной и той же машине (либо если они были идентично настроены и имеют одинаковое железо).

**Корректно**: запустить бенчмарк для PHP и KPHP на своей машине, сравнить числа.

**Некорректно**: найти в интернете результаты для PHP/KPHP и сравнить их с вашими результатами.

<hr></spoiler>

## Правильная интерпретация результатов бенчмарков

`ktest bench` будет запускать тестируемый метод много раз, пытаясь подобрать правильное значение повторений `N`. Однако даже это не даёт нам необходимой точности.

Если вы знакомы с бенчмарками в Go (`go test -bench`), то вы, наверное, уже догадались, о чём сейчас пойдёт речь.

Нам нужно собрать ещё больше результатов, выявить статистическую значимость и оценить погрешности. После этого нам нужно правильно соотнести результаты `v4` и `v4fast`.

Чтобы получить больше сэмплов, `ktest bench` стоит запускать с параметром `count`:

```bash
$ ktest bench -count 5 ./benchmarks/UUIDBenchmark.php | tee results.txt
BenchmarkV4	48574	7408.0 ns/op
BenchmarkV4Fast	129920	5415.0 ns/op
BenchmarkV4	19351	7672.0 ns/op
BenchmarkV4Fast	136986	5367.0 ns/op
BenchmarkV4	24950	7799.0 ns/op
BenchmarkV4Fast	133315	5450.0 ns/op
BenchmarkV4	17505	7755.0 ns/op
BenchmarkV4Fast	137931	5408.0 ns/op
BenchmarkV4	27552	7801.0 ns/op
BenchmarkV4Fast	111049	5512.0 ns/op
```

Есть такая замечательная утилита [benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat), которая позволяет анализировать и сравнивать результаты бенчмарков.

`benchstat` встроен в ktest, поэтому мы можем очень легко оценить разброс результатов:

```bash
$ ktest benchstat results.txt
name    time/op
V4      7.54µs ± 3%
V4Fast  5.33µs ± 2%
```

Разброс в `2-3%` - это неплохой результат, с этим можно работать.

`benchstat` может помочь нам сравнить `V4` и `V4Fast` между собой, но нам придётся немного поработать ручками для этого.

1. Все результаты для `V4Fast` мы кладём в отдельный файл, `new.txt`
2. Все результаты для `V4` мы кладём в отдельный файл, `old.txt`
3. В `new.txt` заменяем `BenchmarkV4Fast` на `BenchmarkV4`

Как-то так:

```bash
$ grep 'V4Fast\b' results.txt > new.txt
$ grep 'V4\b' results.txt > old.txt
$ sed -i 's/V4Fast/V4/g' new.txt
```

Теперь у вас есть два файла похожей структуры: `old.txt` и `new.txt`. Мы можем их сравнить:

```bash
$ ktest benchstat old.txt new.txt 
name  old time/op  new time/op  delta
V4    7.54µs ± 3%  5.33µs ± 2%  -29.27%  (p=0.008 n=5+5)
```

Видим, что `fast` версия (new) на **29.27%** быстрее.

Низкий `p=0.008` говорит о том, что результаты достаточно стабильны.

В целом, я рекомендую использовать `-count` со значением выше, чем 5. Например, неплохо работает значение 10. Чем более "шумный" ваш бенчмарк, тем больше сэмплов вам может захотеться.

Я считаю правилом хорошего тона прикладывать вот такую информацию со сравнением производительности в формате `benchstat` в каждом коммите, который внедряет какую-то оптимизацию. Эту традицию я перенял когда работал над [golang/go](https://github.com/golang/go).

## Запуск бенчмарка в режиме PHP vs KPHP

ktest умеет запускать бенчмарки через PHP:

```bash
$ ktest bench-php ./benchmarks/UUIDBenchmark.php
BenchmarkV4	3617        2057.0 ns/op
BenchmarkV4Fast	185288  850.0 ns/op
```

Если мы соберём одни результаты в файлик `php.txt`, а другие в `kphp.txt`, то через `benchstat` их можно будет между собой сравнить.

К счастью, подкоманда `bench-vs-php` сделает это всё за нас:

```bash
$ ktest bench-vs-kphp ./benchmarks/UUIDBenchmark.php
name    PHP time/op  KPHP time/op  delta
V4       1.14µs ± 2%  7.54µs ± 3%  +558.25%  (p=0.000 n=9+10)
V4Fast   839ns ± 1%   5359ns ± 1%  +539.10%  (p=0.000 n=9+10)
```

## Настраиваем GitHub Actions

Общие рекомендации:

* Максимальное покрытие кода тестами через `phpunit`
* То, что можно, дополнительно запускаем через `ktest phpunit`

Самый простой вариант тестов интеграции в [GitHub actions](https://github.com/features/actions) - запуск в PHP-only режиме. Подразумевается, что KPHP тесты будут запускаться через отдельную команду, локально на машине разработчика (где KPHP точно должен быть установлен).

Для упрощения запуска тестов, создадим `Makefile` в корне проекта:

```makefile
.PHONY: ci-tests test php-test kphp-test

test:
	@echo "Run PHP tests"
	@make php-test
	@echo "Run KPHP tests"
	@make kphp-test
	@echo "Everything is OK"

php-test:
	@phpunit --bootstrap vendor/autoload.php tests

kphp-test:
	@ktest phpunit tests

ci-tests:
	@curl -L -o phpunit.phar https://phar.phpunit.de/phpunit.phar
	@php phpunit.phar --bootstrap vendor/autoload.php tests
```

Работая с репозиторием локально, мы запускаем все тесты:

```bash
$ make test 
```

В GitHub Actions мы будем запускать `make ci-tests`.

Создадим файл `.github/workflows/php.yml`:

```yaml
name: PHP
on: [push, pull_request]

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: php-actions/composer@v5
    - name: Test
      run: |
        make ci-tests
```

Теперь при каждом пуше и создании Pull Request у нас будут запускаться тесты.

![](https://habrastorage.org/webt/ut/8b/zl/ut8bzl3ac9v0kna4ostvefit--4.png)

Не забудьте добавить build status badge в свой README:

```
![Build Status](https://github.com/$user/$repo/workflows/PHP/badge.svg)
```

В моём случае `$user/$repo` будет `quasilyte/kphp-uuid`.

## Заключение

Весь исходный код `kphp-uuid` вместе с Makefile и прочими интеграциями можно найти тут: [github.com/quasilyte/kphp-uuid](https://github.com/quasilyte/kphp-uuid).

Полезные ресурсы:

* [Официальная документация KPHP](https://vkcom.github.io/kphp)
* Анонс перехода разработки KPHP в Open Source: [ВКонтакте снова выкладывает KPHP](https://habr.com/ru/company/vk/blog/527420/)
* Запись доклада [KPHP внутри VK: что там у нас происходит](https://www.youtube.com/watch?v=3vO2TAkq7zE)
* Запись доклада [PHP scripts -> Release binaries](https://youtu.be/nr1883za8tM?t=306)
* Полезные KPHP-сниппеты и обёртки: https://github.com/VKCOM/kphp-snippets
* [KPHP-полифиллы](https://github.com/VKCOM/kphp-polyfills)
