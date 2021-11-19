# Создаём игру на KPHP с помощью FFI и SDL

KPHP теперь поддерживает механизм [Foreign Function Interface](https://www.php.net/manual/ru/class.ffi.php) (FFI). Мы с [Владом](https://github.com/troy4eg) решили продемонстрировать его возможности и за сутки написали первую в мире графическую игру на [KPHP](https://github.com/VKCOM/kphp/).

Игру делали с использованием библиотеки [SDL](https://www.libsdl.org/):

* работали со звуком,
* обрабатывали события клавиатуры,
* рисовали шрифты, спрайты, UI.

API и семантика нашего FFI идентичны PHP. Поэтому созданная игра запустится и на KPHP, и на PHP.

Если вам интересны детали реализации, заглядывайте под кат!

![](https://habrastorage.org/webt/6l/va/vh/6lvavhkwuv6dehyxyq_rr2jec0a.jpeg)

<cut/>

## Предисловие

Немного контекста о том, как KPHP поддерживает Foreign Function Interface и чем это полезно.

KPHP теперь разрабатывается как проект с открытым исходным кодом. Любой может клонировать репозиторий и собрать новую версию компилятора и рантайма. Но в ядро KPHP входят не все PHP-расширения, а добавить недостающие было сложно: для этого нужно разобраться во внутренностях самого KPHP, а также пройти процесс внедрения своего патча в чужой репозиторий. Конечно, можно держать свой форк с дополнительными фичами, но это тоже задача не из лёгких — особенно если планируется время от времени обновлять версию KPHP.

Исправить эту ситуацию призван Foreign Function Interface (FFI). В PHP уже с версии 7.4 можно создавать обёртки для C-библиотек без необходимости написания C-кода и последующей компиляции PHP-расширений.  Теперь в KPHP есть аналогичный функционал, созданный по тому же [RFC](https://wiki.php.net/rfc/ffi). Так что к KPHP можно подключать расширения, не включая их в основной репозиторий. А само расширение можно распространять как composer-пакет.

Мне захотелось продемонстрировать возможности этого нового механизма коллегам на грядущем внутреннем хакатоне. После мозгового штурма мы с Владом решили взять SDL и написать какую-нибудь игрушку.

Исходные коды самой игры размещены в репозитории [github.com/quasilyte/kphp-game](https://github.com/quasilyte/kphp-game), а обёртки к SDL в composer-пакете [kphp-sdlite](https://packagist.org/packages/quasilyte/kphp-sdlite). Это разделение не было обязательным, но оно помогло проверить, что писать переиспользуемые биндинги возможно.

Итак, пишем рогалик на KPHP за сутки, без игрового движка.

> На момент создания игры поддержка FFI всё ещё не была влита в master, поэтому мы использовали свой билд KPHP. Начиная с 29 октября 2021, FFI доступен в основном KPHP под флагом `--enable-ffi`.

## Игровой дизайн

Основные игровые идеи выглядят следующим образом:

* Пошаговое перемещение (действие игрока завершает ход).
* Рандомная генерация игровых уровней.
* В каждой сессии несколько игровых уровней, соединённых порталом.
* Главный герой побеждает противников с помощью заклинаний.
* Персонаж получает опыт, становится сильнее.
* Запас магических сил (мана) не восполняется автоматически.

Я успел придумать и реализовать три заклинания.

| Название | Урон | Дальность | Стоимость | Скорость снаряда |
|---|---|---|---|
| Fireball | 20-30 | 4 | 10 MP | Мгновенно |
| Ice shards | 20-35 | 7 | 15 MP | 1 тайл за ход |
| Thunder | 15-55 | 1 | 20 MP | Мгновенно |

Паттерны поражения заклинаний:

```
Fireball:
   [>][x][x][x][x]

Ice shards:
      [x][x][x][x][x][x][x]
   [>][x][x][x][x][x][x][x]
      [x][x][x][x][x][x][x]

Thunder:
[x][x][x]
[x][>][x]
[x][x][x]
```

![](https://habrastorage.org/webt/iy/4o/5i/iy4o5izrciza-za9zd2mcv6dj00.gif)

Может показаться, что Ice shards во всём превосходит Fireball, однако поскольку мана не восстанавливается, грамотное её распределение — одна из основных механик. Если вы можете победить одним лишь огненным шаром, именно это и будет выгодной стратегией. Более того, ледяные осколки пролетают по одному тайлу за ход, а огненный шар атакует мгновенно, не давая противнику шанса увернуться.

Thunder интересен тем, что позволяет атаковать противника, не поворачиваясь к нему, а атака по диагонали безопасна (финальный босс — единственное исключение).

Каждый уровень протагониста увеличивает базовые характеристики: магический урон, количество здоровья (HP) и маны (MP).

Есть и другие механики, без умелого использования которых пройти игру практически невозможно. Например, через тайлы с обломками нельзя пройти, но через них можно «стрелять» заклинаниями. Через обычные стены нельзя ни колдовать, ни перемещаться.

## Подготавливаем проект

Здесь нет ничего KPHP-специфичного. Создаём директорию, инициализируем composer, ставим зависимости.

```bash
$ mkdir kphp-game
$ cd kphp-game
$ composer init
$ composer require quasilyte/kphp-sdlite:dev-master
```

Вот так выглядит наш autoload (копия из `composer.json`):

```
    "autoload": {
        "psr-4": {
            "KPHPGame\\": "src/"
        }
    }
```

В корне проекта создадим `main.php`, который загрузит все composer-зависимости и запустит приложение.

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use KPHPGame\Game;

function main() {
    try {
        $game = new Game();
        $game->run();
    } catch (Exception $e) {
        $msg = "{$e->getFile()}:{$e->getLine()}: {$e->getMessage()}";
        echo "UNHANDLED EXCEPTION: $msg\n";
    }
}

main();
```

Чтобы протестировать сборку, создадим пустую реализацию класса `KPHPGame\Game` (`src/Game.php`):

```php
<?php

namespace KPHPGame;

class Game {
    public function run(): void {
        var_dump('Hello, world!');
    }
}
```

```bash
# Собираем наш проект:
$ kphp --mode cli --composer-root $(pwd) main.php

# Запускаем её:
$ ./kphp_out/cli
string(13) "Hello, world!"
```

Я рекомендую добавлять такие вещи в `Makefile`, чтобы упростить выполнение этих операций.

```makefile
.PHONY: game

PWD=$(shell pwd)

game:
	composer install
	./kphp2cpp --enable-ffi --mode cli --composer-root $(PWD) main.php
	mkdir -p bin
	mv kphp_out/cli bin/game
```

Сборка и запуск игры становятся проще:

```bash
$ make game && ./bin/game
```

> Процесс установки KPHP, использование composer'а и тестирование описаны в статье ["Заметки KPHP: тестирование и бенчмарки"](https://habr.com/ru/company/vk/blog/572424/).

## Создаём окно

Начнём с создания графического окна, в котором и будет отображаться вся графика.

Нам нужна функция [SDL_CreateWindow](https://wiki.libsdl.org/SDL_CreateWindow).

Чтобы воспользоваться ей из KPHP, нам нужно описать сигнатуру в формате, понятном FFI, — то есть с обычными C-декларациями.

Есть два основных способа размещения этих деклараций:

1. Прямо в PHP-коде, внутри строкового литерала: [FFI::cdef](https://www.php.net/manual/ru/ffi.cdef.php);
2. В отдельном заголовочном файле: [FFI::load](https://www.php.net/manual/ru/ffi.load.php).

Я буду использовать вариант с `FFI::load`. Создадим файл `sdl.h`:

```cpp
#define FFI_SCOPE "sdl"
#define FFI_LIB "libSDL2-2.0.so"

typedef uint32_t Uint32;

// forward declare для SDL_Window (opaque-тип)
typedef struct SDL_Window SDL_Window;

SDL_Window *SDL_CreateWindow(const char *title,
                             int x,
                             int y,
                             int w,
                             int h,
                             Uint32 flags);
```

В FFI по умолчанию определены многие популярные типы вроде `uint32_t`. Но `Uint32` — это алиас типа, специфичный для SDL. Поэтому нам нужно ввести этот псевдоним самостоятельно. `SDL_Window` — непрозрачный для нас тип, поэтому достаточно сделать forward declaration. [FFI_SCOPE](https://www.php.net/manual/en/ffi.scope.php#refsect1-ffi.scope-parameters) и [FFI_LIB](https://www.php.net/manual/en/ffi.load.php#refsect1-ffi.load-description) — это особые для PHP FFI определения.

Чтобы вывести окно по центру, нам потребуется константа `WINDOWPOS_CENTERED`. FFI поддерживает простые константы через `enum`. Но зачастую константы, которые нам хочется использовать в публичном коде, стоит выносить в класс-обёртку.

```php
class SDL {
    public const WINDOWPOS_CENTERED = 805240832;
}
```

Простейший скрипт для использования из KPHP:

```php
<?php

\FFI::load('sdl.h');
$lib = \FFI::scope('sdl');

$x = SDL::WINDOWPOS_CENTERED;
$y = SDL::WINDOWPOS_CENTERED;
$w = 640;
$h = 480;
$window = $lib->SDL_CreateWindow('KPHP Game', $x, $y, $w, $h, 0);
```

> Замечание: этот код будет работать для KPHP, но в PHP использовать `FFI::scope()` можно только в сочетании с opcache preload. Мы ещё вернёмся к этой особенности.

Теперь мы умеем создавать графические окна! Дело за малым.

![](https://habrastorage.org/webt/rq/r4/i3/rqr4i3k_uf_ehjevut6ue14ny64.png)

Использовать FFI напрямую не так приятно, поэтому лучше прятать его внутри PHP-класса:

```php
<?php

namespace Quasilyte\SDLite;

class SDL {
    public const WINDOWPOS_CENTERED = 805240832;

    public function __construct() {
        $this->sdl = \FFI::scope('sdl');
    }

    /** @return ffi_cdata<sdl, struct SDL_Window*> */
    public function createWindow(string $title,
                                 int $x,
                                 int $y,
                                 int $w,
                                 int $h,
                                 int $flags = 0) {
        return $this->sdl->SDL_CreateWindow($title,
                                            $x,
                                            $y,
                                            $w,
                                            $h,
                                            $flags);
    }

    /** @var ffi_scope<sdl> */
    private $sdl;
}
```

На что обратить внимание:

* FFI в KPHP статически типизирован; ошибки находим на момент компиляции;
* `ffi_scope<$scope_name>` — тип результата `\FFI::scope($scope_name)`;
* `ffi_cdata<$scope_name, $type>` — тип для [FFI\CData](https://www.php.net/manual/ru/class.ffi-cdata.php)-объектов.

Использовать SDL через KPHP становится приятнее:

```php
$sdl = new SDL();

$x = SDL::WINDOWPOS_CENTERED;
$y = SDL::WINDOWPOS_CENTERED;
$w = 640;
$h = 480;
$sdl->createWindow('KPHP Game', $x, $y, $w, $h);
```

Отдельное преимущество — теперь у нас есть хороший autocomplete для методов SDL.

Теперь закроем это окно нажатием на красный крестик... Подождите, **ничего не закрывается**!

Это мы поправим позже, когда научимся обрабатывать события. А пока закрываем через менеджер задач (или любым другим способом, который вы любите).

## Рисуем спрайт на экране

Что планируем использовать для отрисовки изображения:

* [IMG_Load](https://www.libsdl.org/projects/SDL_image/docs/SDL_image_11.html)
* [SDL_CreateRenderer](https://wiki.libsdl.org/SDL_CreateRenderer)
* [SDL_CreateTextureFromSurface](https://wiki.libsdl.org/SDL_CreateTextureFromSurface)
* [SDL_FreeSurface](https://wiki.libsdl.org/SDL_FreeSurface)
* [SDL_RenderClear](https://wiki.libsdl.org/SDL_RenderClear)
* [SDL_RenderCopy](https://wiki.libsdl.org/SDL_RenderCopy)
* [SDL_RenderPresent](https://wiki.libsdl.org/SDL_RenderPresent)

Функция `IMG_Load` находится в отдельной библиотеке, `libSDL2_image`. Поэтому создаём второй заголовочный файл, `sdl_image.h`:

```cpp
#define FFI_SCOPE "sdl_image"
#define FFI_LIB "libSDL2_image-2.0.so"

typedef struct SDL_Surface SDL_Surface;

SDL_Surface *IMG_Load(const char *file);
```

Добавим новые определения в уже существующий `sdl.h`:

```cpp
typedef struct SDL_Texture SDL_Texture;
typedef struct SDL_Renderer SDL_Renderer;
typedef struct SDL_Surface SDL_Surface;

typedef struct SDL_Rect {
    int x, y;
    int w, h;
} SDL_Rect;

SDL_Renderer *SDL_CreateRenderer(SDL_Window *window,
                                 int index,
                                 Uint32 flags);

SDL_Texture *SDL_CreateTextureFromSurface(SDL_Renderer *renderer,
                                          SDL_Surface *surface);

void SDL_FreeSurface(SDL_Surface *surface);

int SDL_RenderClear(SDL_Renderer *renderer);

int SDL_RenderCopy(SDL_Renderer *renderer,
                   SDL_Texture *texture,
                   const SDL_Rect *srcrect,
                   const SDL_Rect *dstrect);

void SDL_RenderPresent(SDL_Renderer *renderer);
```

Это **почти** рабочий вариант. Проблема в том, что `sdl` и `sdl_image` декларируют `SDL_Surface`. Эти типы будут несовместимыми. Вот простой пример, который демонстрирует это поведение:

```php
<?php

$cdef = FFI::cdef('
    struct Foo { int x; };
');
$cdef2 = FFI::cdef('
    typedef struct Foo Foo;
    struct Bar {
        struct Foo *fooptr;
    };
');

$foo = $cdef->new('struct Foo');
$bar = $cdef2->new('struct Bar');
$bar->fooptr = FFI::addr($foo);
```

Получаем ошибку:

```
$ php8.0 -f ./test.php
Incompatible types 'struct Foo*' and 'struct Foo*'
```

Хе-хе, `struct Foo*` несовместим с `struct Foo*`. Приятно.

Есть несколько путей обхода. Можно, например, заменить везде `struct Foo*` на `void*`, всё равно для нас это opaque-указатель на ресурс.

```diff
  SDL_Texture *SDL_CreateTextureFromSurface(SDL_Renderer *renderer,
-                                           SDL_Surface *surface);
+                                           void *surface);

- SDL_Surface *IMG_Load(const char *file);
+ void *IMG_Load(const char *file);

- void SDL_FreeSurface(SDL_Surface *surface);
+ void SDL_FreeSurface(void *surface);
```

Альтернативным решением может быть [FFI::cast](https://www.php.net/manual/ru/ffi.cast.php).

Всё, теперь мы готовы вывести картинку на экран!

```php
/**
 * @param ffi_cdata<sdl, struct SDL_Window*>
 */
function renderSomething(SDL $sdl, $window) {
    // Создаём рендерер:
    $rend_flags = SDL::RENDERER_ACCELERATED;
    $rend = $sdl->createRenderer($window, -1, $rend_flags);

    // Подготавливаем текстурку:
    $surface = $sdl->imgLoad('image.png');
    $texture = $sdl->createTextureFromSurface($rend, $surface);
    $sdl->freeSurface($surface);

    // Описываем позицию для рендеринга:
    $pos = $sdl->newRect();
    $pos->w = 64;  // ширина спрайта
    $pos->h = 64;  // высота спрайта
    $pos->x = 128; // смещение по оси X внутри окна
    $pos->y = 256; // смещение по оси Y внутри окна

    // Рендерим текстурку:
    $sdl->renderClear();
    $sdl->renderCopy($texture, null, \FFI::addr($pos));
    $sdl->renderPresent();
}
```

Обычно вы хотите переиспользовать текстурку, возвращаемую `createTextureFromSurface()`, и загружать её в самом начале. Аналогично с рендерером: вы создаёте его где-то в начале жизненного цикла игры, а потом передаёте объект рендерера во все места, где он может потребоваться.

Один из методов немного отличается от того, что мы делали ранее. Вот его внутренности:

```php
/** @return ffi_cdata<sdl, struct SDL_Rect> */
public function newRect() {
    return $this->sdl->new('struct SDL_Rect');
}
```

Так как `SDL_Rect` определён в `sdl.h` полностью, мы можем создавать эти объекты через метод `new` и обращаться с полями этой структуры так, будто бы это обычный PHP-класс.

## Тайлы, текстурные атласы, анимации

Было бы не очень эффективно каждое изображение загружать как отдельный png и ассоциировать с ним текстуру.

В своей игре мы использовали [текстурные атласы](https://en.wikipedia.org/wiki/Texture_atlas) для тайлов.

![](https://habrastorage.org/webt/jw/8t/ta/jw8ttav3ruoqm6kxxwoth8q21qe.png)

В одном изображении размера `128 × 128` пикселей мы можем разместить 16 тайлов размером `32 × 32` пикселя. Всё это будет одной текстурой. Чтобы рендерить такой фрагмент внутри атласа, нам нужно будет передать дополнительный аргумент в `renderCopy`, который ранее был `null`.

```php
$texture_pos = $sdl->newRect();
$texture_pos->w = 32;     // ширина фрагмента (тайла)
$texture_pos->h = 32;     // высота фрагмента (тайла)
$texture_pos->x = 0;      // смещение по оси X внутри атласа
$texture_pos->y = 32 * 3; // смещение по оси Y внутри атласа

// Рисуем алтарь, тайл из нижнего левого угла атласа:
$sdl->renderCopy($texture, \FFI::addr($texture_pos), \FFI::addr($pos));
```

Анимации можно делать по тому же принципу. Одна текстурка с несколькими кадрами. В зависимости от текущего кадра выбираем правильное смещение внутри текстуры.

![](https://habrastorage.org/webt/n9/hd/h_/n9hdh_3gyh8jqpbifggbbv8snns.png)

Кадр анимации переключается игровыми фреймами, а её скорость регулируется их количеством. В нашей игре мы стараемся выдавать 60 фреймов в секунду. 

## Обработка событий, event loop

Мы упомянули игровые фреймы. Самое время писать основной цикл обработки.

Сердцем игры будет вот такой цикл:

```php
while (true) {
    // Считываем и обрабатываем все события, которые
    // произошли за этот фрейм:
    $this->processInputs($sdl);
    // Если вдруг игрок активировал выход:
    if ($this->exit) {
        break;
    }
    // Непосредственно игровая логика:
    $this->processFrame($sdl);
    // Ждём следующего фрейма:
    $sdl->delay(1000 / 60); // ~60 fps
}
```

Этот цикл отрабатывает примерно 60 раз в секунду.

Помните, что у нас не закрывается игровое окно? Самое время это исправить.

```php
public function processInputs(SDL $sdl): void {
    $event = $sdl->newEvent();
    // Событий за фрейм может произойти несколько,
    // поэтому нам здесь нужен цикл.
    while ($sdl->pollEvent($event)) {
        if ($event->type === EventType::QUIT) {
            $this->exit = true;
        }
    }
}
```

Вот теперь приложение корректно закрывается через кнопку `[X]`.

Расширим цикл обработки событий так, чтобы мы понимали, какие действия хочет совершить игрок.

```php
public function processInputs(SDL $sdl): void {
    $event = $sdl->newEvent();
    while ($sdl->pollEvent($event)) {
        if ($event->type === EventType::QUIT) {
            $this->exit = true;
        } elseif ($event->type === EventType::KEYUP) {
            $scancode = $event->key->keysym->scancode;
            if ($scancode === Scancode::ESCAPE) {
                // Нажатие на escape — выход из игры.
                $this->exit = true;
            } elseif ($scancode === Scancode::UP) {
                $this->player_actions = PlayerAction::MOVE_UP;
            } elseif ($scancode === Scancode::DOWN) {
                $this->player_actions = PlayerAction::MOVE_DOWN;
            } elseif ($scancode === Scancode::LEFT) {
                $this->player_actions = PlayerAction::MOVE_LEFT;
            } elseif ($scancode === Scancode::RIGTH) {
                $this->player_actions = PlayerAction::MOVE_RIGHT;
            }
            // И так далее...
        }
    }
}
```

Исходя из наброска выше, нам потребуются функции:

* [SDL_PollEvent](https://wiki.libsdl.org/SDL_PollEvent)
* [SDL_Delay](https://wiki.libsdl.org/SDL_Delay)

Добавляем всё необходимое в `sdl.h`:

```cpp
typedef int32_t Sint32;

typedef Sint32 SDL_Keycode;

typedef struct SDL_Keysym {
    int scancode;
    SDL_Keycode sym;
    Uint16 mod;
    Uint32 unused;
} SDL_Keysym;

typedef struct SDL_KeyboardEvent {
    Uint32 type;
    Uint32 timestamp;
    Uint32 windowID;
    Uint8 state;
    Uint8 repeat;
    Uint8 padding2;
    Uint8 padding3;
    SDL_Keysym keysym;
} SDL_KeyboardEvent;

typedef struct SDL_QuitEvent {
    Uint32 type;
    Uint32 timestamp;
} SDL_QuitEvent;

typedef union SDL_Event {
    Uint32 type;
    SDL_KeyboardEvent key;
    SDL_QuitEvent quit;
} SDL_Event;

int SDL_PollEvent(SDL_Event *event);

void SDL_Delay(Uint32 ms);
```

В `union SDL_Event` входит больше вариантов, чем мы описали выше. Настоятельно рекомендуется заполнять union-типы идентично тому, как они определены в библиотеке, потому что это может влиять на их размер. В статье мы сократили определение, выбрав только нужные для нас события, однако в реальном приложении следовало бы перечислить более десяти структур, которые могут быть членами `SDL_Event`.

## Рендерим тексты

Для работы со шрифтами подключим [SDL_ttf](https://www.libsdl.org/projects/SDL_ttf/).

Нас в первую очередь интересуют UTF-8 варианты API, поэтому вместо [TTF_RenderText_Blended](https://www.libsdl.org/projects/docs/SDL_ttf/SDL_ttf_44.html) мы выбираем [TTF_RenderUTF8_Blended](https://www.libsdl.org/projects/SDL_ttf/docs/SDL_ttf_52.html) и дальше по аналогии.

Создаём `sdl_ttf.h`:

```cpp
#define FFI_SCOPE "sdl_ttf"
#define FFI_LIB "libSDL2_ttf-2.0.so"

typedef uint8_t Uint8;

typedef struct TTF_Font TTF_Font;

typedef struct SDL_Color {
    Uint8 r;
    Uint8 g;
    Uint8 b;
    Uint8 a;
} SDL_Color;

int TTF_Init();

TTF_Font *TTF_OpenFont(const char *file, int ptsize);

void *TTF_RenderUTF8_Blended(TTF_Font *font,
                             const char *text,
                             SDL_Color fg);

int TTF_SizeUTF8(TTF_Font *font, const char *text, int *w, int *h);
```

Используется это так: сначала делается `TTF_Init()`, чтобы инициализировать библиотеку; затем через `TTF_OpenFont()` открываем шрифт и держим его при себе, как и с другими ресурсами. После этого используем `TTF_RenderUTF8_Blended()` для рендеринга текста выбранным шрифтом.

Функция `TTF_SizeUTF8()` нужна для того, чтобы вычислить ширину текста, если его отрисовать указанным шрифтом. При этом она возвращает код ошибки. Настоящий результат функции записывается в `$w` и `$h`.

```php
$w = \FFI::new('int');
$h = \FFI::new('int');
$sdl->TTF_SizeUTF8($font, "example", \FFI::addr($w), \FFI::addr($h));

// Прочитать значения CData скалярных типов можно через var->cdata:
var_dump([$w->cdata, $h->cdata]);

// Через var->cdata также можно записывать новые значения:
$w->cdata = 64;
$h->cdata = 32;
```

## Создаём UI

Чтобы отрисовать простейшие элементы графического интерфейса, достаточно уметь отображать прямоугольники разных цветов и размеров.

Добавим следующие функции в `sdl.h`:

```cpp
int SDL_RenderFillRect(SDL_Renderer *renderer, const SDL_Rect *rect);

int SDL_SetRenderDrawColor(SDL_Renderer *renderer, 
                           Uint8 r,
                           Uint8 g,
                           Uint8 b,
                           Uint8 a);
```

Передавать цвета через несколько отдельных значений не всегда удобно. Особенно расстраивает увеличивающееся количество аргументов. Мы можем определить простой класс `Color`, который позволит более лаконично описывать связанные с цветом аргументы.

```php
class Color {
    public int $r;
    public int $g;
    public int $b;
    public int $a;

    public function __construct($r, $g, $b, $a = 255) {
        $this->r = $r;
        $this->g = $g;
        $this->b = $b;
        $this->a = $a;
    }
}
```

Удобную обёртку для `SDL_SetRenderDrawColor` можно написать, например, так:

```php
/**
 * @param ffi_cdata<sdl, struct SDL_Renderer*> $renderer 
 */
public function setRenderDrawColor($renderer, Color $color): bool {
    $result = $this->sdl->SDL_SetRenderDrawColor($renderer,
                                                 $color->r,
                                                 $color->g,
                                                 $color->b,
                                                 $color->a);
    return $result === 0;
}
```

Код ниже рисует красный квадрат `32 × 32`:

```php
$rend = $sdl->createRenderer();
$rect = $sdl->newRect();
$rect->w = 32;
$rect->h = 32;
$red_color = new Color(255, 0, 0);
$sdl->setRenderDrawColor($rend, $red_color);
$sdl->fillRect($rend, \FFI::addr($rect));
```

В нашей игре мы реализовали диалоговое окно с «рамкой» — для этого отрисовали два прямоугольника разного цвета, немного отличающихся по размеру.

![](https://habrastorage.org/webt/ml/ey/0t/mley0tmlx1riu6qi00qz9jwfkzw.png)

## Добавляем звуки в игру

Для работы со звуком мы взяли библиотеку [SDL_Mixer](https://www.libsdl.org/projects/SDL_mixer/).

Хочется уметь проигрывать два вида звуков: одноразовые (для спецэффектов) и длинные аудио с повтором (для музыки).

Примерно такой API у нас получился:

```php
// Инициализируем аудио (Mix_OpenAudio):
$freq = 22050;
$chunk_size = 4096;
$channels = 2;
$sdl->openAudio($freq, SDL::AUDIO_S16LSB, $channels, $chunk_size);

// Загружаем файл (Mix_LoadMUS):
$music = $sdl->loadMusic("music.ogg");

// Запускаем фоновую музыку (Mix_PlayMusic):
$sdl->playMusic($music);
```

Для отдельных звуковых эффектов, которые нужно повторять много раз во время игры, нужно сохранять загруженные ресурсы и переиспользовать их — подобно тому, как мы сохраняем текстуры.

```php
private function loadResources(SDL $sdl) {
    $this->fireballSound = $this->loadSound($sdl, "fireball.wav");
    // ... И так далее.
}

/** @return ffi_cdata<sdl_mixer, struct Mix_Chunk*> */
private function loadSound(SDL $sdl, string $path) {
    $rw = $sdl->mixerlib->SDL_RWFromFile($path, "rb");
    return $sdl->mixerlib->Mix_LoadWAV_RW($rw, 1);
}

/** @param ffi_cdata<sdl_mixer, struct Mix_Chunk*> $sound */
private function playSoundOnce(SDL $sdl, $sound) {
    $ch = -1;   // Канал для воспроизведения; -1 для автовыбора
    $loops = 0; // Сколько раз дополнительно повторить звук
    if (!$sdl->mixerlib->Mix_PlayChannelTimed($ch, $sound, $loops, -1)) {
        $err = $sdl->getError();
        if ($err === 'No free channels available') {
            Logger::info('trying to play too many sounds at once');
            return;
        }
        throw new \RuntimeException($err);
    }
}
```

Поскольку бездумно закидывать канал запросами на воспроизведение не очень умно, нам нужно либо ставить их в очередь, либо уметь как-то сказать «Горшочек, не вари». Поскольку код писался под хакатон, я решил просто игнорировать эту ошибку и считать, что если игрок зажимает все клавиши одновременно, то мы имеем право не воспроизвести какой-то из звуков.

Обращаем внимание на то, что в SDL_Mixer имеется символ [Mix_LoadWAV](https://www.libsdl.org/projects/SDL_mixer/docs/SDL_mixer_19.html). Вот только определён этот символ не как функция, а как макрос. Поэтому мы не можем загрузить `Mix_LoadWAV` из динамической библиотеки.

Макрос определён следующим образом:

```cpp
#define Mix_LoadWAV(file) \
    Mix_LoadWAV_RW(SDL_RWFromFile(file, "rb"), 1)
```

Мы можем определить в классе SDL метод с таким же API:

```php
/** @return ffi_cdata<sdl_mixer, struct Mix_Chunk*> */
public function loadWAV(string $filename) {
    $rw = $this->mixerlib->SDL_RWFromFile($filename, "rb");
    return $this->mixerlib->Mix_LoadWAV_RW($rw, 1);
}
```

Все функции для работы с SDL_Mixer должны быть определены в отдельном файле, назовём его `sdl_mixer.h`:

```cpp
#define FFI_SCOPE "sdl_mixer"
#define FFI_LIB "libSDL2_mixer-2.0.so"

typedef uint16_t Uint16;

typedef struct Mix_Chunk Mix_Chunk;
typedef struct Mix_Music Mix_Music;
typedef struct SDL_RWops SDL_RWops;

int Mix_OpenAudio(int frequency,
                  Uint16 format,
                  int channels,
                  int chunksize);

SDL_RWops *SDL_RWFromFile(const char *file, const char *mode);
Mix_Chunk *Mix_LoadWAV_RW(SDL_RWops *src, int freesrc);

Mix_Music *Mix_LoadMUS(const char *file);
int Mix_PlayMusic(Mix_Music *music, int loops);

int Mix_PlayChannelTimed(int channel,
                         Mix_Chunk *chunk,
                         int loops,
                         int ticks);
```

## Обработка ошибок

Здесь всё зависит от конкретной функции, которую мы используем.

Есть два основных способа, которыми функции из SDL сигнализируют об ошибках:

1. Возвращается NULL (напр. `SDL_CreateTextureFromSurface`)
2. Возвращается int, не равный 0 (напр. `Mix_PlayMusic`)

Если ошибка произошла, через [SDL_GetError](https://wiki.libsdl.org/SDL_GetError) можно получить текст этой ошибки.

Для второго способа часто можно упростить тип возврата с `int` до `bool`:

```php
/** @param ffi_cdata<sdl_mixer, struct Mix_Music*> $music */
public function playMusic($music, int $loops = -1): bool {
    return $this->mixerlib->Mix_PlayMusic($music, $loops) === 0;
}
```

Функции, которые возвращают null, проверять стоит обычным сравнением с null, а не через [\FFI::isNull](https://www.php.net/manual/ru/ffi.isnull.php).

```php
// 1.
$texture = $sdl->createTextureFromSurface($rend, $surface);
if ($texture === null) {
    throw new \Exception($sdl->getError());
}

// 2. 
if (!$sdl->playMusic($music)) {
    throw new \Exception($sdl->getError());
}
```

## Запускаем игру на PHP

В коде нашей игры мы использовали некоторые KPHP-специфичные возможности. Например, tuple-типы. Чтобы этот код заработал на PHP, достаточно подключить к проекту [KPHP-полифиллы](https://github.com/VKCOM/kphp-polyfills):

```bash
$ composer require vkcom/kphp-polyfills
```

PHP умеет регистрировать FFI scope только на этапе preload, поэтому создадим скрипт `php_preload.php`:

```php
<?php

$sdlite_path = __DIR__ . '/vendor/quasilyte/kphp-sdlite/src/';
\FFI::load("$sdlite_path/sdl.h");
\FFI::load("$sdlite_path/sdl_image.h");
\FFI::load("$sdlite_path/sdl_mixer.h");
\FFI::load("$sdlite_path/sdl_ttf.h");
```

Запускать игру нужно примерно так:

```bash
$ php8.0 \
    -d opcache.enable=1 \
    -d opcache.enable_cli=1 \
    -d opcache.preload=./php_preload.php \
    -f ./main.php
```

## Поддерживаемые платформы

Поддержка MacOS в KPHP частичная. Запустить полноценный сервер получится только на Linux. 

Однако в случае нашей игры веб-сервер не нужен, поэтому запустить её можно на обеих системах.

> Первые пару часов хакатона Влад как раз пытался собрать наш FFI под MacOS.
> Я тестировал только на Linux, поэтому с первого раза там ничего, конечно же, не завелось.

Здесь отдельное внимание стоит уделить тому, как игра ищет свои ресурсы при работе. Все доступы к ресурсам (изображения, звуковые файлы, шрифты и прочее) стоит делать через некоторый `AssetsManager`, который знает, как найти нужный asset в зависимости от окружения.

Простейший `AssetsManager` может выглядеть так:

```php
<?php

namespace KPHPGame;

class AssetsManager {
    public static function sound(string $name): string {
        return self::getRootByTarget() . "sounds/$name";
    }

    public static function sprite(string $name): string {
        return self::getRootByTarget() . "sprites/$name";
    }

    private static function getRootByTarget(): string {
        $target = $_ENV['KPHP_GAME_TARGET'] ?? '';
        if ($target === 'linux') {
            return "./assets/";
        }
        if ($target === 'macos') {
            return "./../Resources/";
        }
        // dev-режим:
        return __DIR__ . "/../assets/";
    }
}
```

Переменную окружения `KPHP_GAME_TARGET` мы задаём при запуске игры. Какое значение туда выставить, должен знать сам launcher для игры. Этот launcher можно генерировать при подготовке релизной сборки.

Доступ к изображениям в коде меняется следующим образом:

```diff
- $surface = $sdl->imgLoad('image.png');
+ $surface = $sdl->imgLoad(AssetsManager::sprite('image.png'));
```

## Заключение

![](https://habrastorage.org/webt/sg/v3/na/sgv3naqxa1cpnkhocznkgi2aaek.png)

Мы создали игру, которая запускается как на PHP, так и на KPHP. Однако это не всё, чего можно добиться благодаря FFI. С его помощью можно реализовать некоторые отсутствующие в KPHP расширения PHP.

Например, через FFI можно добавить в KPHP поддержку:

* [GMP](https://www.php.net/manual/en/book.gmp.php)
* [GD](https://www.php.net/manual/en/book.image.php)
* [ImageMagick](https://www.php.net/manual/en/book.imagick.php)

Это только то, на чём я успел протестировать текущую реализацию. Уверен, что список этим не ограничивается.

И напоследок - видео с игровым процессом:

<video>https://www.youtube.com/watch?v=L44l4Tqm4Fc</video>

В ролике вы видите именно ту версию, что была сделана к дедлайну хакатона. На следующий день я добавил в игру пару дополнительных механик для улучшения геймплея. Например, алтари на уровне, которые дают случайный бонус.

### Полезные ссылки

* [Исходный код игры](https://github.com/quasilyte/kphp-game).
* [Исходный код SDL2-биндингов](https://github.com/quasilyte/kphp-sdlite).
* [Документация FFI в KPHP](https://vkcom.github.io/kphp/kphp-language/php-extensions/ffi.html).
* [Заметки KPHP: тестирование и бенчмарки](https://habr.com/ru/company/vk/blog/572424/).
* [Презентация по этой же теме](https://speakerdeck.com/quasilyte/kphp-ffi).
