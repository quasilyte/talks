# Делаем RPG на Go: часть 0

Один из самых частых вопросов в нашем [сообществе разработке игр на Go](https://t.me/go_gamedev) - это с чего начать.

В этой серии статей мы будем изучать движок [Ebitengine](https://github.com/hajimehoshi/ebiten/) и создадим RPG в процессе.

![](https://habrastorage.org/webt/fl/vb/v-/flvbv-ox6a1prob1p3ag9xj3t7g.jpeg)

<cut/>

## Вступление

Что от вас ожидается:

* Вам интересна разработка игр на Go
* Вы уже владеете этим языком программирования
* Никаких шуточек про название движка Ebitengine

Это не курс по программированию на Go, а я не буду убеждать вас, что разработка игр на Go - нечто великолепное. Однако, если вам любопытна эта тема, то мне есть, чем с вами поделиться.

## Знакомимся с Ebitengine

До того, как мы начнём использовать Ebitengine, я предлагаю склонировать репозиторий и позапускать примеры.

```bash
$ git clone --depth 1 https://github.com/hajimehoshi/ebiten.git
$ cd ebiten
```

Перед тем, как мы сможем запускать игры, нужно [установить dev зависимости](https://ebitengine.org/en/documents/install.html). Они нужны только для компиляции игр, игрокам ставить ничего не придётся.

После установки зависимостей, запустите эти игры, находясь в директории `ebiten`:

```bash
$ go run ./examples/blocks
$ go run ./examples/flappy
$ go run ./examples/2048
$ go run ./examples/snake
```

Эти игры довольно простые, тем и хороши как объекты для исследования: там мало кода. Всего примеров около 80 и чаще всего они концентрируются на одной теме (например, на игровой камере).

Ресурсы для этих игр хранятся в `./examples/resources`.

Это традиционный способ начать знакомство с Ebitengine - запускать примеры, читать их код, модифицировать эти игры. Всякий раз, когда захочется сделать перерыв от следования этим статьям, отвлекитесь на эти примеры.

То, что примеры почти никогда не используют сторонние библиотеки - это одновременно и плюс, и минус. Это хорошо, чтобы получше понять базовый функционал движка. Но количество лишнего кода и некоторых не очень красивых решений может отпугнуть новых разработчиков.

Я перейду ко сторонним библиотекам почти сразу. Это уменьшит количество шагов назад с переписыванием кода.

## Создаём Проект

Начнём с создания директории где-нибудь в удобном для вас месте.

```bash
$ mkdir mygame && cd mygame
```

Игры на Go - это обычные приложения, поэтому вторым шагом будет инициализация модуля.

```bash
$ go mod init github.com/quasilyte/ebitengine-hello-world
```

Нам сразу же потребуется Ebitengine. Ставить нужно вторую версию.

```bash
$ go get github.com/hajimehoshi/ebiten/v2
```

Пакет main размещаем в `cmd/mygame`:

```bash
$ mkdir -p cmd/mygame
```

```go
package main

import (
	"github.com/hajimehoshi/ebiten/v2"
	"github.com/hajimehoshi/ebiten/v2/ebitenutil"
)

func main() {
	g := &myGame{
		windowWidth:  320,
		windowHeight: 240,
	}

	ebiten.SetWindowSize(g.windowWidth, g.windowHeight)
	ebiten.SetWindowTitle("Ebitengine Quest")

	// RunGame ожидает реализации трёх методов:
	// Update, Draw и Layout; они определены ниже.
	if err := ebiten.RunGame(g); err != nil {
		panic(err)
	}
}

type myGame struct {
	windowWidth  int
	windowHeight int
}

func (g *myGame) Update() error {
	return nil
}

func (g *myGame) Draw(screen *ebiten.Image) {
	ebitenutil.DebugPrint(screen, "Hello, World!")
}

func (g *myGame) Layout(w, h int) (w, h int) {
	// Layout - тема для продвинутых, поэтому нам пока
	// достаточно считать, что screen size = window size.
	return g.windowWidth, g.windowHeight
}
```

Игры в Ebitengine имеют разделённые логические тики и фреймы отрисовки. Количество кадров в секунду - FPS, количестко тиков в секунду - TPS.

Любая отрисовка графики на экран должна происходить в `Draw`. Игровая логика должна находиться в `Update`. После вызова [RunGame](https://pkg.go.dev/github.com/hajimehoshi/ebiten#RunGame) наша игра попадает в game loop, управляемый движком.

На вход в `Draw` мы получаем `ebiten.Image`, который по конвенции обычно называют screen. Ожидается, что на каждый вызов `Draw` мы будем заполнять этот image нужными пикселями. Каждый объект, который должен быть виден в игровом окне, должен быть отрисован на screen. Чаще всего это делается через метод [DrawImage](https://pkg.go.dev/github.com/hajimehoshi/ebiten#Image.DrawImage), который позволяет отрисовать одну текстуру на другой.

Если мы запустим эту игру, то получим чёрное окно с возмутительно уникальным текстом:

```bash
$ go run ./cmd/mygame
```

![](https://habrastorage.org/webt/1b/oj/bw/1bojbwbk2u_bhw5uoszdcgceeso.png)

## Загрузка Изображений

Многофункциональных спрайтов в движке нет, но тип [ebiten.Image](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image) весьма хорош как стартовая точка. Для тестового изображения возьмём [gopher.png](https://github.com/hajimehoshi/ebiten/blob/main/examples/resources/images/flappy/gopher.png) из `examples/resources`.

> Мы можем считать, что `ebiten.Image` - это абстракция над набором пикселей с методами отрисовки. Когда мы перейдём к абстракции спрайтов, image будет для нас чем-то вроде уровня текстур.

Изображение гофера мы разместим в пакете `assets`:

```
mygame/
  cmd/mygame/main.go
  internal/assets/
    _data/images/gopher.png
```

Часть важных ассетов можно хранить прямо в исполняемом файле игры с помощью `go:embed`. Пакет `assets` будет предоставлять доступ ко всем ресурсам игры.

```go
package assets

//go:embed all:_data
var gameAssets embed.FS

func OpenAsset(path string) io.ReadCloser {
	// Функция OpenAsset могла бы работать как с данными внутри бинарника,
	// так и с внешними. Для этого ей нужно распознавать ресурс по его пути.
	// Самым простым вариантом является использование префиксов в пути,
	// типа "$music/filename.ogg" вместо "filename.ogg", когда мы ищем
	// файл во внешнем каталоге (а не в бинарнике).
	//
	// Но на данном этапе у нас только один источник ассетов - бинарник.
	f, err := gameAssets.Open("_data/" + path)
	if err != nil {
		panic(err)
	}
	return f
}
```

Чтобы отрендерить изображение на экране, нужно большее, чем доступный на чтение ассет. Нужно декодировать PNG и создать объект `ebiten.Image` на основе этого. Аналогичные шаги нужно выполнять для остальных видов ресурсов - музыки (OGG), звуковых эффектов (WAV), шрифтов и так далее.

На помощь приходит библиотека [ebitengine-resource](https://github.com/quasilyte/ebitengine-resource). Она же будет ответственна за кеширование (мы не хотим декодировать одинаковые ресурсы несколько раз).

Все доступы к ресурсам будут проходить через числовые ключи (ID).

```go
package assets

import resource "github.com/quasilyte/ebitengine-resource"

const (
	ImageNone resource.ImageID = iota
	ImageGopher
)
```

Связка идентификаторов с метаданными ручная.

```go
package assets

import (
	_ "image/png"
)

func registerImageResources(loader *resource.Loader) {
	imageResources := map[resource.ImageID]resource.ImageInfo{
		ImageGopher: {Path: "images/gopher.png"},
	}

	for id, res := range imageResources {
		loader.ImageRegistry.Set(id, res)
	}
}
```

> `ebitengine-resource` требует импорта пакета `image/png` со стороны пользователя. Делать это нужно ровно один раз, в любом месте программы. Лучше всего для этого подходит файл, который описывает графические ресурсы.

Создаётся менеджер ресурсов на старте программы, а далее пробрасывается как часть контекста всей игры. Для текущего примера можно разместить `loader` внутри объекта `myGame`.

```go
package main

import (
	"github.com/quasilyte/ebitengine-hello-world/internal/assets"

	"github.com/hajimehoshi/ebiten/v2/audio"
	resource "github.com/quasilyte/ebitengine-resource"
)

func createLoader() *resource.Loader {
	sampleRate := 44100
	audioContext := audio.NewContext(sampleRate)
	loader := resource.NewLoader(audioContext)
	loader.OpenAssetFunc = assets.OpenAsset
	return loader
}
```

Теперь в любом месте программы мы можем использовать доступ по ID изображения, чтобы получить `*ebiten.Image`:

```go
img := loader.LoadImage(assets.ImageGopher)
```

Во время первого доступа по ключу, менеджер ресурсов загрузит ассет, декодирует его и закеширует. Все следующие обращения будут возвращать уже созданный для ресурса объект.

> Если выполнять для каждого ресурса `Load` где-нибудь на экране загрузки, то можно заранее прогреть все кеши.

## Отрисовка Изображения

Вот новый код метода `Draw` игры:

```go
func (g *myGame) Draw(screen *ebiten.Image) {
	gopher := g.loader.LoadImage(assets.ImageGopher).Data
	var options ebiten.DrawImageOptions
	screen.DrawImage(gopher, &options)
}
```

![](https://habrastorage.org/webt/2l/qz/si/2lqzsifzju6jv1hg2l30jrnpetu.png)

Гофер рисуется в позиции `{0,0}`. Мы можем поменять позицию, выполнив пару манипуляций с `options`. Но чтобы было интереснее, мы введём сущность player и закрепим изображение за ними.

Позиции в 2D играх чаще всего описываются как двумерные вектора. Настало время импортировать следующую библиотеку.

```go
package main

import "github.com/quasilyte/gmath"

type Player struct {
	pos gmath.Vec // {X, Y}
	img *ebiten.Image
}
```

> Пакет [gmath](https://github.com/quasilyte/gmath) содержит множество полезных в геймдеве математических функций. Большая часть API повторяет то, что можно найти в Godot.

Обработку инпутов мы рассмотрим в следующей статье, а сегодня игрок будет перемещаться автоматически. Так как перемещение - это логика, а не рендеринг, исполнять этот код мы будем внутри `Update`.

```go
// Так как теперь у нас есть объект, требующий инициализации,
// мы будем создавать его на старте игры.
// Метод init() нужно вызывать явно в main() до RunGame.
func (g *myGame) init() {
	gopher := g.loader.LoadImage(assets.ImageGopher).Data
	g.player = &Player{img: gopher}
}

func (g *myGame) Update() error {
	// В Ebitengine нет никаких time delta.
	// Подробнее почитать об этом можно тут:
    // https://github.com/tinne26/tps-vs-fps
    // По умолчанию, TPS=60, отсюда 1/60.
	g.player.pos.X += 16 * (1.0 / 60.0)

	return nil
}
```

Рендеринг остаётся внутри `Draw`:

```go
func (g *myGame) Draw(screen *ebiten.Image) {
	var options ebiten.DrawImageOptions
	options.GeoM.Translate(g.player.pos.X, g.player.pos.Y)
	screen.DrawImage(g.player.img, &options)
}
```

Такой способ отрисовки изображений слишком низкоуровневый, поэтому уже в следующей статье мы начнём использовать обёртки, реализующие более удобные спрайты.

## Закрепляем Изученное

* Традиционный способ изучать Ebitengine - исследовать [examples](https://github.com/hajimehoshi/ebiten/tree/main/examples)
* В играх на Ebitengine разделённые циклы для `Update` и `Draw`
* Поверхностно познакомились с [ebiten.Image](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image)
* Для загрузки и кширования ресурсов - [ebitengine-resource](https://github.com/quasilyte/ebitengine-resource)
* Для векторной двумерной арифметики - [gmath](https://github.com/quasilyte/gmath)
* В Ebitengine нет [time delta](https://ebitencookbook.vercel.app/blog)
* Запоминаем структуру проекта, которую я ввёл (дальше - больше)

Исходные коды этого небольшого проекта находятся в репозитории [ebitengine-hello-world](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0) (ссылка на тег `part0`).

В [следующий раз](https://habr.com/ru/articles/799497/) мы добавим в список используемых библиотек сцены, спрайты и кое-что для продвинутой обработки ввода игрока.

Причина, по которой мы не сразу начали использовать спрайты - время от времени вы всё равно будете работать с `ebiten.Image` как с полноценным объектом. Например, когда функционал спрайтов не покрывает ваши специфичные задачи. Тем более что менеджер ресурсов кеширует изображения именно как `ebiten.Image`.

Статей будет достаточно много, потому что впереди нас ждёт долгий путь.

Подключайтесь к нам в [телеграм-сообщество](https://t.me/go_gamedev), если тема геймдева на Go вам интересна.
