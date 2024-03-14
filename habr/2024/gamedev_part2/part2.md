# Делаем RPG на Go: часть 0.5

В [предыдущей статье](https://habr.com/ru/articles/791192/) мы начали знакомство с [Ebitengine](https://github.com/hajimehoshi/ebiten/).

В этой части структура игры будет доработана и переведена на сцены.

![](https://habrastorage.org/webt/m3/ur/0f/m3ur0fwi5ldndirq4578onhvwkq.jpeg)

<cut/>

## Часть 0.5?

Это вторая pre-1 часть, в которой разрабатывается отдельный демо-проект.

Начинать делать RPG с нулевой базы было бы слишком сложно: я хочу использовать все свои любимые библиотеки и практики как можно раньше, при этом у меня не получилось придумать способа достаточно плавно вводить все составляющие на менее искусственном проекте.

Возможно, уже следующая статья станет "настоящей" первой частью, а пока запасаемся терпением и осваиваем базовые приёмы разработки игр на Go.

![](https://habrastorage.org/webt/i5/sz/re/i5szre0ot7hawfyrywyvtu06r54.jpeg)

## Управляем Гофером

В Ebitengine из коробки есть простенькие функции для обработки ввода игрока. Работают они на уровне конкретных кнопок.

```go
// github.com/hajimehoshi/ebiten/v2
ebiten.IsKeyPressed(ebiten.KeyEnter)

// github.com/hajimehoshi/ebiten/v2/inpututil
inpututil.IsKeyJustPressed(ebiten.KeyEnter)
```

Через такое API нельзя абстрагироваться от конкретных кнопок и устройств ввода, поэтому для своих игр я использую пакет [ebitengine-input](https://github.com/quasilyte/ebitengine-input). Вдохновением для этой библиотеки были [Godot actions](https://docs.godotengine.org/en/stable/tutorials/inputs/inputevent.html#actions).

Вместо нажатий на кнопки, этой библиотекой проверяются активации действий. Возможные действия в нашей игре мы определяем через константы.

В демо-игре можно будет перемещаться в четырёх направлениях, поэтому действий будет как минимум четыре: `MoveRight`, `MoveDown`, `MoveLeft`, `MoveUp`.

Actions я предпочитаю размещать в отдельном пакете `controls`:

```
mygame/
  cmd/mygame/main.go
  internal/
    assets/
      _data/images/gopher.png
    controls/actions.go
```

```go
// internal/controls/actions.go

package controls

import (
	input "github.com/quasilyte/ebitengine-input"
)

const (
	ActionNone input.Action = iota

	ActionMoveRight
	ActionMoveDown
	ActionMoveLeft
	ActionMoveUp

	// Эти действия понадобятся позднее.
	ActionConfirm
	ActionRestart
)
```

Действиям нужно сопоставить триггеры активации. Триггером может быть нажатие на кнопку клавиатуры, контроллера, мыши, сенсорного экрана и так далее. Совокупность этих отображений я буду называть keymap.

Каждой игре нужен keymap по умолчанию. При желании, можно добавлять поддержку внешних конфигов или выполнять ремапинг конкретных действий внутри игры. Для нашей демо-игры достаточно статического keymap.

```go
// internal/controls/default_keymap.go

package controls

import (
	input "github.com/quasilyte/ebitengine-input"
)

var DefaultKeymap = input.Keymap{
	ActionMoveRight: {
		input.KeyRight,        // Кнопка [>] на клавиатуре
		input.KeyD,            // Кнопка [D] на клавиатуре
		input.KeyGamepadRight, // Кнопка [>] на крестовине контроллера
	},
	ActionMoveDown: {
		input.KeyDown,
		input.KeyS,
		input.KeyGamepadDown,
	},
	ActionMoveLeft: {
		input.KeyLeft,
		input.KeyA,
		input.KeyGamepadLeft,
	},
	ActionMoveUp: {
		input.KeyUp,
		input.KeyW,
		input.KeyGamepadUp,
	},

	ActionConfirm: {
		input.KeyEnter,
		input.KeyGamepadStart,
	},
	ActionRestart: {
		input.KeyWithModifier(input.KeyR, input.ModControl),
		input.KeyGamepadBack,
	},
}
```

Осталось создать объект считывания ввода, привязанный к заданному keymap. Этот объект создаётся через `input.System`, который нужен в единственном экземпляре на всю игру.

```diff
 type myGame struct {
 	windowWidth  int
 	windowHeight int
 
+	inputSystem input.System
 	loader      *resource.Loader
 
 	player *Player
 }
```

Системе ввода требуется разовая инициализация до запуска игры:

```go
g.inputSystem.Init(input.SystemConfig{
    DevicesEnabled: input.AnyDevice,
})
```

На каждый `Update` в игре нужно вызывать одноимённый метод в системе ввода:

```diff
 func (g *myGame) Update() error {
+	g.inputSystem.Update()
 	g.player.pos.X += 16 * (1.0 / 60.0)
 	return nil
 }
```

После интеграции системы, можно создавать те самые объекты считывания ввода. Эти объекты в библиотеке называются handlers. Каждый обработчик привязан к player ID, что особенно важно для игр с возможностью подключить несколько контроллеров одновременно.

Для демо-игры достаточно лишь одного обработчика с нулевым ID.

```diff
	inputSystem input.System
+	input       *input.Handler
 	loader      *resource.Loader
```

```go
g.input = g.inputSystem.NewHandler(0, controls.DefaultKeymap)
```

Теперь через `g.input` можно проверять состояние действий. Сцен у нас пока нет, поэтому вся логика будет сосредоточена в основном `Update`.

```go
func (g *myGame) Update() error {
	g.inputSystem.Update()

	speed := 64.0 * (1.0 / 60)
	var v gmath.Vec
	if g.input.ActionIsPressed(controls.ActionMoveRight) {
		v.X += speed
	}
	if g.input.ActionIsPressed(controls.ActionMoveDown) {
		v.Y += speed
	}
	if g.input.ActionIsPressed(controls.ActionMoveLeft) {
		v.X -= speed
	}
	if g.input.ActionIsPressed(controls.ActionMoveUp) {
		v.Y -= speed
	}
	g.player.pos = g.player.pos.Add(v)

	return nil
}
```

![](https://habrastorage.org/webt/ts/ix/-w/tsix-wlnuvunvr-jz-1siikcxbe.gif)

Это управление будет работать со всеми способами активации, которые мы задали в keymap: можно перемещаться на стрелочках, WASD, и даже через контроллер.

Тег [part0.5_controls](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0.5_controls) содержит состояние кода демо-игры после добавления управления.

## Рефакторинг

Перед тем, как внедрять сцены, стоит произвести рефакторинг.

Для начала, я вынесу контекст игры, существующий между сценами, в пакет `game`. То, что останется в объекте `myGame`, будет недоступно для сцен напрямую.

```diff
 type myGame struct {
-	windowWidth  int
-	windowHeight int
 
 	inputSystem input.System
-	input       *input.Handler
-	loader      *resource.Loader
 
 	player *Player // Это вынесем позже, в сцену
 }
```

```go
// internal/game/context.go

package game

import (
	input "github.com/quasilyte/ebitengine-input"
	resource "github.com/quasilyte/ebitengine-resource"
)

type Context struct {
	Input  *input.Handler
	Loader *resource.Loader

	WindowWidth  int
	WindowHeight int
}
```

> Что именно попадает в игровой контекст сильно зависит от игры и ваших предпочтений.

Тег [part0.5_game_context](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0.5_game_context) - это репозиторий после данного рефакторинга.

## Введение в Сцены

Чтобы разделить игру на отдельные части, удобно иметь понятие сцены.

Ранее в игре уже была неявная сцена - вся игра целиком. Игра запускается, `myGame` исполняет `Update+Draw` цикл для этой единственной сцены. Явные сцены меняют многое и требуют дополнительного кода, но их преимущества довольно быстро оправдывают эти инвестиции.

Переход на явные сцены выглядит примерно так:

```go
type myGame struct {
	ctx *game.Context
}

func (g *myGame) Update() error {
    g.ctx.InputSystem.Update()
	g.ctx.CurrentScene().Update()
	return nil
}

func (g *myGame) Draw(screen *ebiten.Image) {
	g.ctx.CurrentScene().Draw(screen)
}

type Scene struct {
	// ...
}

func (s *Scene) Update() {
	for _, o := range s.objects {
		o.Update()
	}
}

func (s *Scene) Draw(screen *ebiten.Image) {
	for _, g := range s.graphics {
		o.Draw(screen)
	}
}
```

> Обратите внимание, текущую сцену я храню в `game.Context`, а не в объекте `myGame`.

Переход из одной сцены в другую происходит через замену `myGame.ctx.currentScene`. Логика сцены заключена в её объектах (scene.objects), а вся графика реализована графическими объектами (scene.graphics).

Библиотека `gscene` реализует именно эту модель:

```bash
$ go get github.com/quasilyte/gscene
```

Объекты и графика (т.н. графические объекты) - это интерфейсы.

```go
type SceneObject interface {
	Init(*Scene)
	Update()
	IsDisposed() bool
}

type SceneGraphics interface {
	Draw(dst *ebiten.Image)
	IsDisposed() bool
}
```

Метод `IsDisposed` потребуется для удаления объектов со сцены. `Init` вызывается на объектах при их добавлении на сцену. Через аргумент-сцену эти объекты могут добавить на сцену дополнительные объекты или графику.

В моей интерпретации сцен очень полезно иметь один особенный вид объекта, по одному на сцену - контроллер. Сцена содержит объекты и является их контейнером, в то время как контроллер, закреплённый за сценой, является главным объектом этой сцены. Он же добавляет на сцену первый набор объектов.

Контроллер реализует интерфейс `SceneObject`, но без метода `IsDisposed`.

Объекты сцены могут получить доступ к объекту-контроллеру. Всё станет понятнее на примере.

## Создаём Сцены

У нас будет две сцены: экран-заставка и экран с геймплеем. Игра стартует на экране заставки, а после активации действия confirm переходит на сцену с геймплеем.

Переход сцены - это замена текущей сцены на новую. Реализация такой замены может выгдядеть так:

```go
// internal/game/context.go

// Реализуем как свободную функцию, потому что иначе не получится
// параметризовать функцию для разных T.
func ChangeScene[T any](ctx *Context, c gscene.Controller[T]) {
	s := gscene.NewRootScene[T](c)
	ctx.scene = s
}

// Заметим, что CurrentScene возвращает интерфейс GameRunner,
// а не сцену. Это позволяет унифицировать
// разные Scene[T] с точки зрения игрового цикла,
// ведь там достаточно иметь Update+Draw и ничего более.
func (ctx *Context) CurrentScene() gscene.GameRunner {
	return ctx.scene
}
```

Все сцены рекомендую хранить в пакете `scenes`. Для простейших сцен, типа сплеш-экрана, <abbr title="В этом случае вся сцена будет реализована одним лишь контроллером">достаточно одного файла</abbr> внутри `scenes`, а для более сложных случаев стоит создавать вложенные пакеты, по одному на каждую подобную сцену.

По желанию - давать этим вложенным сценам префикс `scene*`, чтобы не было конфликтов с другими пакетами (вполне обычная ситуация - иметь пакет `battle` для сцены и для каких-то общих геймплейных определений).

Для <abbr title="Типы, которые реализуют интерфейс gscene.SceneObject">логических объектов</abbr> сцены я предпочитаю добавлять суффикс `*node` (как в именах файлов, так и в именах типов).

```
mygame/
  cmd/mygame/main.go
  internal/
    assets/
      _data/images/gopher.png
    controls/actions.go
    scenes/
      splash_controller.go
      walkscene/
        walkscene_controller.go
        gopher_node.go
```

Контроллер для экрана заставки будет заглушкой, так как красиво показать текст на экране пока не выйдет (нужную библиотеку добавим позже). Всё, что он будет делать - это переключаться на основную сцену после обработки активации confirm.

```go
// internal/scenes/splash_controller.go

package scenes

import (
	"github.com/quasilyte/ebitengine-hello-world/internal/controls"
	"github.com/quasilyte/ebitengine-hello-world/internal/game"
	"github.com/quasilyte/ebitengine-hello-world/internal/scenes/walkscene"
	"github.com/quasilyte/gscene"
)

type SplashController struct {
	ctx *game.Context
}

func NewSplashController(ctx *game.Context) *SplashController {
	return &SplashController{ctx: ctx}
}

func (c *SplashController) Init(s *gscene.SimpleRootScene) {
	// В заглушке никакого текста вроде "press [Enter] to continue"
	// мы показывать не будем. Вернёмся к этому немного позднее.
}

func (c *SplashController) Update(delta float64) {
	if c.ctx.Input.ActionIsJustPressed(controls.ActionConfirm) {
		game.ChangeScene(c.ctx, walkscene.NewController(c.ctx))
	}
}
```

```go
// internal/scenes/walkscene/walkscene_controller.go

package walkscene

import (
	"os"

	"github.com/quasilyte/ebitengine-hello-world/internal/game"
	"github.com/quasilyte/gscene"
)

type Controller struct {
	ctx *game.Context
}

func NewController(ctx *game.Context) *Controller {
	return &Controller{ctx: ctx}
}

func (c *Controller) Init(s *gscene.SimpleRootScene) {
	os.Exit(0) // Пока что заглушка
}

func (c *Controller) Update(delta float64) {
}
```

При запуске игры у нас будет чёрный экран (сцена splash), а после обработки confirm игра сразу закроется, перейдя в сцену walkscene.

## Устанавливаем Graphics

```bash
$ go get github.com/quasilyte/ebitengine-graphics
```

Большая часть конструкторов из graphics требует передачи объекта `*graphics.Cache`, поэтому для удобства этот кеш следует спрятать внутри `game.Context`. Во многих играх будет полезно обернуть конструкторы графических объектов в методы контекста, чтобы сократить количество аргументов при вызове.

```go
// internal/game/context.go

func NewContext() *Context {
	return &Context{
		graphicsCache: graphics.NewCache(),
	}
}

func (ctx *Context) NewLabel(id resource.FontID) *graphics.Label {
	fnt := ctx.Loader.LoadFont(id)
	return graphics.NewLabel(ctx.graphicsCache, fnt.Face)
}

func (ctx *Context) NewSprite(id resource.ImageID) *graphics.Sprite {
	s := graphics.NewSprite(ctx.graphicsCache)
	if id == 0 {
		return s
	}
	img := ctx.Loader.LoadImage(id)
	s.SetImage(img.Data)
	return s
}
```

## Установка Шрифтов

Для отрисовки текста потребуется шрифт в формате ttf или otf. Скачиваем [DejavuSansMono.ttf](https://www.fontsquirrel.com/fonts/dejavu-sans-mono) и сохраняем его в `internal/assets/_data/fonts`.

По аналогии с <abbr title="Их мы разбирали в прошлой статье">графическими ресурсами</abbr>, ресурсы-шрифты нужно зарегистрировать.

```go
// internal/assets/fonts.go

package assets

import (
	resource "github.com/quasilyte/ebitengine-resource"
)

const (
	FontNone resource.FontID = iota
	FontNormal
	FontBig
)

func registerFontResources(loader *resource.Loader) {
	fontResources := map[resource.FontID]resource.FontInfo{
		FontNormal: {Path: "fonts/DejavuSansMono.ttf", Size: 10},
		FontBig:    {Path: "fonts/DejavuSansMono.ttf", Size: 14},
	}

	for id, res := range fontResources {
		loader.FontRegistry.Set(id, res)
		loader.LoadFont(id)
	}
}
```

В `RegisterResources` добавляется вызов `registerFontResources`:

```diff
 func RegisterResources(loader *resource.Loader) {
 	registerImageResources(loader)
+	registerFontResources(loader)
 }
```

## Объекты и Graphics

Пора добавить на сплэш-экран запрос на нажатие клавиши подтверждения.

Внутри метода `SplashController.Init` добавляется label с нужным текстом:

```go
func (c *SplashController) Init(s *gscene.SimpleRootScene) {
	l := c.ctx.NewLabel(assets.FontBig)
	l.SetAlignHorizontal(graphics.AlignHorizontalCenter)
	l.SetAlignVertical(graphics.AlignVerticalCenter)
	l.SetSize(c.ctx.WindowWidth, c.ctx.WindowHeight)
	l.SetText("Press [Enter] to continue")
	s.AddGraphics(l)
}
```

> Label - это графический объект, поэтому на сцену добавляется через `AddGraphics`.

![](https://habrastorage.org/webt/ch/dw/l6/chdwl6fwzbi4esc4ivhlpds-fra.png)

Есть два типа сцен - корневая и обычная. Корневая (root) передаётся в инициализатор контроллера. Всем остальным объектам сцены передаётся некорневая сцена.

Основная причина разделения на два типа сцен - улучшение API. Корневая сцена напрямую интегрируется в игровой цикл, у неё есть методы `Update` и `Draw`. Сцена объектов же этих методов не имеет.

Сцена всегда параметризируется типом контроллера (или <abbr title="Это может быть полезно, если требуется ограничить доступ к контроллеру; или когда контроллер и объекты находятся в разных пакетах, так как в Go запрещены циклические импорты">интерфейсом доступа к нему</abbr>). Если же объектам доступ к контроллеру не нужен, то параметром можно выставить <abbr title="Для корневой сцены в пакете gscene есть алиас SimpleRootScene, который мы уже использовали выше">any</abbr>.

Такое связывание помогает любому объекту получить доступ к контроллеру через объект сцены. Чтобы в рамках пакета не указывать генерик-тип, можно задать псевдоним:

```go
// internal/scenes/walkscene/walkscene_controller.go

package walkscene

import "github.com/quasilyte/gscene"

// Этот псевдоним типа упростит сигнатуры внутри пакета.
type scene = gscene.Scene[*Controller]
```

Гофер станет <abbr title="Методы Init, Update, IsDisposed, добавляется на сцену через AddObject">логическим объектом сцены</abbr>:

```go
// internal/scenes/walkscene/gopher_node.go

package walkscene

import (
	graphics "github.com/quasilyte/ebitengine-graphics"
	"github.com/quasilyte/ebitengine-hello-world/internal/assets"
	input "github.com/quasilyte/ebitengine-input"
	"github.com/quasilyte/gmath"
)

type gopherNode struct {
	input  *input.Handler
	pos    gmath.Vec
	sprite *graphics.Sprite
}

func newGopherNode(pos gmath.Vec) *gopherNode {
	return &gopherNode{pos: pos}
}

func (g *gopherNode) Init(s *scene) {
	// Controller() возвращает тип T, который связан со сценой.
	// В данном случае это walkscene.Controller.
	ctx := s.Controller().ctx

	g.input = ctx.Input

	g.sprite = ctx.NewSprite(assets.ImageGopher)
	g.sprite.Pos.Base = &g.pos
	s.AddGraphics(g.sprite)
}

func (g *gopherNode) IsDisposed() bool {
	return false
}

func (g *gopherNode) Update(delta float64) {
	// Здесь код, который раньше был в myGame Update.
}
```

Спрайт гофера - это его графический компонент. Пакет `graphics` использует тип [Pos](https://pkg.go.dev/github.com/quasilyte/gmath#Pos) для привязки позиции графического объекта к его владельцу. Позиция гофера - это часть логики, а спрайт лишь подглядывает на значение через указатель.

```go
type Pos struct {
	Base   *Vec
	Offset Vec
}
```

> Во время создания своих игр вам почти никогда не придётся создавать собственные графические типы вроде `Sprite`.

Гофер создаётся и добавляется на сцену внутри `Init` метода контроллера.

```go
func (c *Controller) Init(s *gscene.RootScene[*Controller]) {
	g := newGopherNode(gmath.Vec{X: 64, Y: 64})
	s.AddObject(g)
}
```

Тег [part0.5_scenes](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0.5_scenes) включает в себя описанные выше изменения.

Впереди ещё много кода, поэтому вот вам мем для разнообразия:

![](https://habrastorage.org/webt/ez/vw/jl/ezvwjl8gyue9po-c9gianzdkfco.jpeg)

## Добавляем Геймплея

В демо-проекте игровая механика будет очень простая - собирать квадраты, получать очки социального рейтинга.

Так как коллизии и физику разбирать в этой статье я не буду, квадраты будут проверять свою дистанцию до игрока и, если она ниже порога, будет происходить начисление баллов.

Здесь есть два варианта: или хранить объект гофера прямо в контроллере и доступаться к нему через сцену, или вынести это в явное разделяемое состояние. Я предпочитаю второй вариант.

```go
// internal/scenes/walkscene/scene_state.go

package walkscene

type sceneState struct {
	gopher *gopherNode
}
```

Объект `sceneState` хранится внутри контроллера и создаётся во время его инициализации.

```diff
 type Controller struct {
 	ctx *game.Context
 
+	state *sceneState
+	scene *gscene.RootScene[*Controller]
 }
```

```diff
 func (c *Controller) Init(s *gscene.RootScene[*Controller]) {
+	c.scene = s
 
 	g := newGopherNode(gmath.Vec{X: 64, Y: 64})
 	s.AddObject(g)
 
+	c.state = &sceneState{gopher: g}
 }
```

> Доступ к этому state-объекту можно выполнять как <abbr title="s.Controller().state">через сцену</abbr>, так и явно передавая state-объект в конструктор. Дело вкуса и вероисповеданий.

Без генератора случайных чисел будет скучно, поэтому добавляем его в контекст игры.

```diff
 type Context struct {
+	Rand gmath.Rand
 	...
```

Инициализировать рандом можно в `main`:

```go
ctx.Rand.SetSeed(time.Now().Unix())
```

Добавим объект-подбирашки:

```go
// internal/scenes/walkscene/pickup_node.go

package walkscene

import (
	graphics "github.com/quasilyte/ebitengine-graphics"
	"github.com/quasilyte/gmath"
	"github.com/quasilyte/gsignal"
)

type pickupNode struct {
	pos      gmath.Vec
	rect     *graphics.Rect
	scene    *scene
	score    int
	disposed bool

	EventDestroyed gsignal.Event[int]
}

func newPickupNode(pos gmath.Vec) *pickupNode {
	return &pickupNode{pos: pos}
}

func (n *pickupNode) Init(s *scene) {
	n.scene = s
	ctx := s.Controller().ctx

	// Количество очков-награды за подбор объекта
	// будет в случайном диапазоне от 5 до 10.
	n.score = ctx.Rand.IntRange(5, 10)

	n.rect = ctx.NewRect(16, 16)
	n.rect.Pos.Base = &n.pos
	n.rect.SetFillColorScale(graphics.ColorScaleFromRGBA(200, 200, 0, 255))
	s.AddGraphics(n.rect)
}

func (n *pickupNode) IsDisposed() bool {
	// Можно было бы использовать n.rect.IsDisposed(),
	// но я рекомендую не привязывать логику объектов
	// к состоянию графических компонентов.
	return n.disposed
}

func (n *pickupNode) Update(delta float64) {
	g := n.scene.Controller().state.gopher
	if g.pos.DistanceTo(n.pos) < 24 {
		n.pickUp()
	}
}

func (n *pickupNode) pickUp() {
	n.EventDestroyed.Emit(n.score)
	n.dispose()
}

func (n *pickupNode) dispose() {
	// Каждый объект должен вызывать методы Dispose
	// у своих компонентов в явном виде.
	n.rect.Dispose()
	n.disposed = true
}
```

> Здесь я добавил пакет [gsignal](https://github.com/quasilyte/gsignal), который реализует что-то вроде сигналов из Godot.

Осталось добавить код создания подбираемых объектов на сцене. Этот код добавляется в контроллер.

```diff
 type Controller struct {
+	scoreLabel *graphics.Label
+	score      int
 	...
```

```go
// internal/scenes/walkscene/walkscene_controller.go

func (c *Controller) createPickup() {
	p := newPickupNode(gmath.Vec{
		X: c.ctx.Rand.FloatRange(0, float64(c.ctx.WindowWidth)),
		Y: c.ctx.Rand.FloatRange(0, float64(c.ctx.WindowHeight)),
	})

	p.EventDestroyed.Connect(nil, func(score int) {
		c.addScore(score)
		c.createPickup()
	})

	c.scene.AddObject(p)
}

func (c *Controller) addScore(score int) {
	c.score += score
	c.scoreLabel.SetText(fmt.Sprintf("score: %d", c.score))
}
```

```diff
 func (c *Controller) Init(s *gscene.RootScene[*Controller]) {
 	...
 
+	c.scoreLabel = c.ctx.NewLabel(assets.FontNormal)
+	c.scoreLabel.Pos.Offset = gmath.Vec{X: 4, Y: 4}
+	s.AddGraphics(c.scoreLabel)
 
+	c.createPickup()
+	c.addScore(0) // Установит текст у scoreLabel
 }
```

![](https://habrastorage.org/webt/sr/dl/cw/srdlcw39vjpbp8btt8l3wkk4zt0.gif)

Весь код можно увидеть под тегом [part0.5_pickups](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0.5_pickups).

## Полировка

Напоследок добавим несколько менее значительных особенностей.

Начнём с разворота спрайта гофера при движении в левую сторону. Делается это в пару строк:

```diff
 func (g *gopherNode) Update(delta float64) {
 	...
 
+	if !v.IsZero() {
+		g.sprite.SetHorizontalFlip(v.X < 0)
+	}
 
 	g.pos = g.pos.Add(v)
 }
```

![](https://habrastorage.org/webt/cr/lr/dp/crlrdpfnnojuvxwov2a52mkqmty.gif)

Я хочу продемонстрировать, как легко реализовать рестарт сцены с этим фреймворком:

```go
// internal/scenes/walkscene/walkscene_controller.go

func (c *Controller) Update(delta float64) {
	if c.ctx.Input.ActionIsJustPressed(controls.ActionRestart) {
		game.ChangeScene(c.ctx, NewController(c.ctx))
	}
}
```

Всё, что нужно сделать - это заменить текущую сцену, используя новый экземпляр контроллера.

<spoiler title="Как бороться с цикличными импортами"><hr>

В Go запрещены циклические импорты пакетов.

В игре возможна ситуация, когда из сцены A есть переход в сцену Б, а из Б можно вернуться в А. Так как смена сцены требует создания контроллера, может возникнуть запрещённый импорт в случае, если контроллеры для А и Б определены в разных пакетах.

Универсального и лучшего ответа на эту ситуацию у меня пока нет. Начну с самого простого лайфхака, который позволит использовать указанную выше структуру проекта.

С помощью интерфейсов в Go можно стереть прямую зависимость. Используя тип [gscene.GameRunner](https://pkg.go.dev/github.com/quasilyte/gscene#GameRunner) можно принять любой объект контроллера как аргумент конструктора. Контроллер А сохранит в себе контроллер Б и, когда нужно будет сменить сцену, воспользуется заранее созданным объектом.

```go
func NewController(ctx *game.Context, back gscene.GameRunner) *Controller {
	return &Controller{
		ctx:  ctx,
		back: back,
	}
}
```

Из пакета scenes вызов будет выглядеть так:

```go
backController := NewSplashController(c.ctx)
controller := walkscene.NewController(c.ctx, backController)
game.ChangeScene(c.ctx, controller)
```

В момент, когда нужно "вернуться" в сцену splash, мы используем переданный ранее контроллер:

```go
game.ChangeScene(c.ctx, c.back)
```

Для более сложных случаев можно применить подход со scene registry.

```go
// internal/game/scene_registry.go

type SceneRegistry {
	NewSplashController(*Context) gscene.GameRunner
	NewWalksceneController(*Context) gscene.GameRunner
}
```

Реестр сцен добавляется как поле контекста. Связывание происходит в `main`:

```go
ctx.Scenes.NewSplashController = scenes.NewSplashController
ctx.Scenes.NewWalksceneController = walkscene.NewController
```

В момент смены сцены мы вызываем конструктор через реестр:

```go
game.ChangeScene(c.ctx, c.ctx.Scenes.NewSplashController(c.ctx))
```

Третьим вариантом является описание **всех** контроллеров в одном пакете `scenes`. Тогда из каждого контроллера будет возможность создать любой другой. Логика сложных сцен всё так же будет выноситься в отдельные пакеты, но переход между сценами будет обрабатываться внутри контроллера.

<hr></spoiler>

Финальную версию кода можно найти по тегу [part0.5_final2](https://github.com/quasilyte/ebitengine-hello-world/releases/tag/part0.5_final2).

## Закрепляем Изученное

Структура проекта на данный момент:

```
mygame/
  cmd/
    mygame/
    main.go
  internal/
    assets/
      _data/
        images/
          gopher.png
        fonts/
          DejavuSansMono.ttf
      assets.go
      fonts.go
      images.go
    controls/
      actions.go
      default_keymap.go
    game/
      context.go
    scenes/
      splash_controller.go
      walkscene/
        walkscene_controller.go
        scene_state.go
        gopher_node.go
        pickup_node.go
```

* Используем [ebitengine-input](https://github.com/quasilyte/ebitengine-input) для обработки пользовательского ввода
* Разделяемое между сценами состояние выносим в объект контекста
* Используем сцены [gscene](https://github.com/quasilyte/gscene) для разделения игры на разные "экраны"
* У каждой сцены есть контроллер (первый логический объект на сцене)
* Контроллер работает с `RootScene`, объекты - со `Scene`
* Сцену можно параметризировать как самим контроллером, так и интерфейсом
* Строго разделяем графические и логические объекты сцены
* Конвенция `Dispose`: объекты удаляют свои компоненты, вызывая их `Dispose` и так далее
* Для графических объектов используем пакет [ebitengine-graphics](https://github.com/quasilyte/ebitengine-graphics)
* Для связывания объектов пользуемся [gsignal](https://github.com/quasilyte/gsignal)

Подключайтесь к нам в [телеграм-сообщество](https://t.me/go_gamedev), если тема геймдева на Go вам интересна.
