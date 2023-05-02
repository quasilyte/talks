Давненько я не писал никаких статей на Хабре.

Я планировал вести серию заметок о [разработке игр на Go](https://t.me/go_gamedev) и начал я с [рендеринга текста](https://habr.com/ru/articles/671556/), но меня не хватило даже на второй текст. Что же, настало время возвращаться, ведь с того момента я успел создать ещё несколько игрушек.

Сегодня я расскажу вам о [шейдерах](https://ru.wikipedia.org/wiki/%D0%A8%D0%B5%D0%B9%D0%B4%D0%B5%D1%80) в [Ebitengine](https://github.com/hajimehoshi/ebiten). Большая часть примеров будет взята из [Roboden](https://quasilyte.itch.io/roboden) и [Decipherism](https://quasilyte.itch.io/decipherism) (обе игры имеют открытые исходные коды и вы можете найти их на гитхабе).

<img title="На скриншоте видны сразу несколько шейдеров, мы рассмотрим их всех" src="https://habrastorage.org/webt/fq/jg/ed/fqjgedeltwlgpfj8h4s6v895r_8.png"/>

<cut/>

<p></p>

## Коротко о том, что такое шейдеры

Я буду говорить только о [фрагментных шейдерах](https://thebookofshaders.com/01/?lan=ru) (они же пиксельные), так как только такие поддерживаются в Ebitengine.

Фрагментный шейдер - это такой алгоритм, который описывает как преобразить пиксели изображения перед его отображением. Чаще всего этот алгоритм описан в виде кода, но существуют визуальные способы создавать шейдеры. На каком именно диалекте языка шейдеров описываются эти программы зависит от движка, который вы используете, так как они могут пытаться скрыть от вас детали того, под какую именно видеокарту шейдер вы пишите (но об этом позже).

Простейшим шейдером может быть программа, которая умножает alpha-канал каждого пикселя на 0.5, делая изображение полупрозрачным. Шейдеры могут быть очень комплексными и создавать эффекты, похожие на анимацию: искажения, волны, динамическое изменение цвета.

## Мотивация

А зачем нам вообще нужны шейдеры? Для изменения alpha-канала чаще всего есть способы, не требующие шейдеров. Создать анимацию волны можно и через несколько кадров.

И отчасти это даже правда: задачу, которую можно решить шейдерами, можно решить и без них. Однако, у шейдеров есть неоспоримые преимущества:

* Шейдеры могут упростить имплементацию
* Они почти всегда будут более эффективны, чем альтернативы
* Код получается более поддерживаемый, чем набор компонентов, реализующих эффект

Статья будет в формате задач и их решения. Мы ставить перед собой цель реализовать некоторый эффект, а затем будем воплощать это в жизнь через шейдеры.

## Раунд 1: отображение повреждений на объектах

Предположим, что у нас в игре есть здания. Вы можете захотеть графически отображать степень повреждённости здания. Как мы будем это делать?

Спрайты зданий, вид сверху, четыре разновидности:

<img title="Спрайты наших зданий" src="https://habrastorage.org/webt/qo/cd/65/qocd65hehbqrqncecv-fuace2is.png"/><p></p>

Как насчёт маски повреждений, которую мы будем накладывать поверх здания? Когда повреждений нет, у этой маски будет нулевая непрозрачность. По мере получения повреждений, альфа-канал увеличивается в своём значении и маска становится более заметной.

<img title="4 вариации маски повреждений" src="https://habrastorage.org/webt/xf/2d/wl/xf2dwlbrgjik6x7y-oez4jtaxs4.png"/>

> Я нарисовал только одну маску и покрутил её шагом в 90 градусов, чтобы получить 4 спрайта.

Теперь попробуем наложить их. Пусть количество урона равно ~100% и видимость маски близка к абсолютной.

<img title="Наложили маску повреждений на спрайты" src="https://habrastorage.org/webt/aj/c3/uu/ajc3uuhcrywvxeao_vjbmkwaaws.png"/><p></p>

Выглядит не очень аккуратно: такая маска подходит только для квадратных спрайтов.

Что только люди не придумают, чтобы не прибегать к шейдерам:

* Рисуют разные по форме маски, чтобы они идеально подходили под спрайт.
* Или, наоборот, везде используют круглые текстуры повреждений, которые подойдут везде.
* Применяют режимы отрисовки ([composite mode](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)) по пересечению текстур.

А давайте попробуем решить задачу через шейдер, не меняя картинку повреждений.

Сначала я напишу шейдер, а затем уже покажу как его подключать к изображениям.

```go
package main

var HP float // Значение уровня здоровья, от 0 до 1

func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	c := imageSrc0At(texCoord)    // Пиксель из спрайта здания
	mask := imageSrc1At(texCoord) // Пиксель из маски
	if c.a != 0.0 && mask.a != 0.0 {
		a := clamp(HP+(1.0-mask.a), 0.0, 1.0)
		// Создаём более тёмный пиксель при повреждениях.
		return vec4(c.r*a, c.g*a, c.b*a, c.a)
	}
	return c // Используем пиксель как есть
}
```

Шейдеру требуются:

* Текстура самого изображения (Src0)
* Текстура повреждений (Src1)
* Параметр HP для вычисления цветовых компонентов

<img src="https://habrastorage.org/webt/eq/ht/gi/eqhtgiutnpeqyx6igdilz0mqgri.png"/><p></p>

Src0 и Src1 должны быть идентичных размеров, поэтому каждому пикселю из Src0 есть какой-то соответствующий пиксель из Src1. Для каждого пересечения непрозрачных пикселей из Src0 и Src1 мы вычисляем новый цвет.

Шейдерный результат выглядит так:

<img title="Повреждения через шейдеры" src="https://habrastorage.org/webt/uo/kc/qn/uokcqnu6jgp6ocxk37fkog_qfik.png"/><p></p>

При желании, можно доработать шейдер так, чтобы текстура повреждений не накладывалась на контуры объекта. Проверяя не только на `c.a`, можно определить, нужно ли слияние текстур в этом пикселе или нет.

## О языке шейдеров Kage

Вы обратили внимание, что шейдер мы написали на Go-подобном языке?

В Ebitengine для написания шейдеров используется Kage, собственная разработка движка. Kage транслятор парсит Go-код, а затем генерирует из него сниппет на нужном диалекте. Например, на моей машине шейдер из прошлого примера преобразуется в следующий код:

```cpp
#if defined(GL_ES)
precision highp float;
#else
#define lowp
#define mediump
#define highp
#endif

int modInt(int x, int y) {
	return x - y*(x/y);
}

uniform vec2 U0;
uniform vec2 U1[4];
// ... много других uniform-деклараций.

varying vec2 V0;
varying vec4 V1;

vec4 F5(in vec2 l0);
vec4 F7(in vec2 l0);
vec4 F12(in vec4 l0, in vec2 l1, in vec4 l2);

vec4 F5(in vec2 l0) { /* ... */ }

vec4 F7(in vec2 l0) { /* ... */ }

vec4 F12(in vec4 l0, in vec2 l1, in vec4 l2) {
	vec4 l3 = vec4(0);
	vec4 l4 = vec4(0);
	l3 = F5(l1);
	l4 = F7(l1);
	if ((((l3).a) != (0.0)) && (((l4).a) != (0.0))) {
		float l5 = float(0);
		l5 = clamp((U8) + ((1.0) - ((l4).a)), 0.0, 1.0);
		return vec4(((l3).r)*(l5), ((l3).g)*(l5), ((l3).b)*(l5), (l3).a);
	}
	return l3;
}

void main(void) {
	gl_FragColor = F12(gl_FragCoord, V0, V1);
}
```

Знакомые для Go концепции тоже неплохо переводятся:

```cpp
// func F0() (int, int) { return 1, 2 }
void F0(out int l0, out int l1) {
	l0 = 1;
	l1 = 2;
	return;
}
```

Моё мнение насчёт Kage неоднозначное. С одной стороны, я понимаю, почему добавили этот слой абстракции. С другой стороны, Kage затрудняет работу с шейдерами как новичкам, там и опытным создателям шейдеров. Первым сложнее изучать нечто с минимальным количеством документации, а вторым сложнее применить уже существующие знания.

Плюсы Kage:

* Удобства редактирования как у Go: работает gofmt, автодополнение, go to definition
* Работают знакомые для Go концепции, типа multi-value return.
* Немного больше переносимости из-за возможности транслировать во что угодно.
* Kage-слой позволяет движку лучше контролировать шейдеры (завернуть их как требуется).
* Это прозвучит смешно, но Kage - это приятный для автора Ebitengine велосипед.

Минусы Kage:

* Сложнее портировать шейдеры; приходится переписывать на Kage.
* Меньше документации, примеров, туториалов
* Меньше шейдерного тулинга (жду визуального редактора шейдеров от [Артёма](https://github.com/sedyh/)).
* На практике отлаживать Kage сложнее.

Вот некоторые полезные сведения о Kage, которые нам вскоре пригодятся:

* Точка входа - функция `Fragment()`, её параметры мы будем разбирать отдельно
* Есть типы `int`, `float`, `vec2`, `vec4`
* Доступны встроенные функции, типа `distance()`, `clamp()` и `imageSrc0At()`
* Мы можем определять новые функции и константы (только числовые)
* Доступны фичи типа [свиззлинга](https://ebitengine.org/en/documents/shader.html#Swizzling)
* Арифметические операции типа `*` так же работают для векторов (`vec2`, `vec4`)

Координаты описываются через `vec2` (x, y), цвета через `vec4` (r, g, b, a).

## Подключение шейдера к изображению

Будем считать, что на игровой сцене у нас находятся объекты `Sprite`. Они содержат в себе [`*ebiten.Image`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image) и, опционально, скомпилированный шейдер.

```go
type Sprite struct {
	x, y float64

	img *ebiten.Image

	shader        *ebiten.Shader
	shaderTexture *ebiten.Image
	shaderParams  map[string]any
}
```

Отрисовка спрайтов без шейдеров может выглядеть так:

```go
func (s *Sprite) Draw(dst *ebiten.Image) {
	var options ebiten.DrawImageOptions
	options.GeoM.Translate(s.x, s.y)
	dst.DrawImage(s.img, &options)
}
```

Далее мы в своём корневом [`game.Draw`](https://pkg.go.dev/github.com/hajimehoshi/ebiten#Game) вызываем `Sprite.Draw()` и получаем отрисовку всех спрайтов на экране.

Теперь добавим рендеринг с шейдерами:

```go
func (s *Sprite) Draw(dst *ebiten.Image) {
	// Если шейдера нет, то делаем всё как раньше.
	if s.shader == nil {
		var options ebiten.DrawImageOptions
		options.GeoM.Translate(s.x, s.y)
		dst.DrawImage(s.img, &options)
		return
	}
	// Здесь нам нужен другой options-тип.
	var options ebiten.DrawRectShaderOptions
	options.GeoM.Translate(s.x, s.y)
	options.Images[0] = s.img           // Src0
	options.Images[1] = s.shaderTexture // Src1
	options.Uniforms = s.shaderParams
	b := s.img.Bounds()
	drawDest.DrawRectShader(b.Dx(), b.Dy(), s.shader, &options)
}
```

Кода стало больше, но ничего принципиально сложного там нет. Нам нужно правильно заполнить [`DrawRectShaderOptions`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#DrawRectShaderOptions) и вызвать [`DrawRectShader()`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image.DrawRectShader) вместо [`DrawImage()`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Image.DrawImage).

Откуда берутся `s.shaderParams` и `s.shaderTexture`? Я предлагаю закреплять их за спрайтом единожды при установке шейдера:

```go
type ShaderParams struct {
	Compiled *ebiten.Shader
	Uniforms map[string]any
	Src1     *ebiten.Image
	// ... при желании можно добавить поля Src2, Src3
}

func (s *Sprite) SetShader(params ShaderParams) {
	s.shader = params.Compiled
	s.shaderParams = params.Uniforms
	s.shaderTexture = params.Src1
}
```

[`*ebiten.Shader`](https://pkg.go.dev/github.com/hajimehoshi/ebiten/v2#Shader) можно переиспользовать для всех спрайтов, которым нужен эффект, реализуемый шейдером. Аналогично с `*ebiten.Image`, который будет использоваться как `Src1`. А вот "данные" (uniforms) для каждого спрайта будут свои.

Так как `map` - это обёртка над указателем, изменения снаружи будут видны внутри `Sprite`. Этим мы будем пользоваться для изменения параметров шейдера.

Код объекта, который использует спрайт с шейдером, будет похож на такой:

```go
func (b *Building) Init() {
	b.shaderData = map[string]any{"HP": 1.0}
	b.sprite = NewSprite()
	b.sprite.SetShader(damageShader, damageMask, b.shaderData)
}

func (b *Building) OnDamage(damage float64) {
	b.hp -= damage
	if b.hp <= 0 {
		b.destroy()
		return
	}
	// Обновляем параметр шейдера.
	// Обратите внимание: использовать нужно float32.
	// Поддерживаются типа int, float32 и []float32, но не float64.
	b.shaderData["HP"] = float32(b.hp / b.maxHP)
}
```

* `damageShader` - это `*ebiten.Shader`, созданный из нашего шейдер-сниппета
* `damageMask` - это `*ebiten.Image`, который содержит маску повреждений
* `b.shaderData` принадлежит объекту `Building`, а шейдер эти данные лишь читает

Наш скрипт шейдера - это обычный файл, данные. Хранить его можно или рядом с приложением, либо встраивать прямо в бинарник через `go:embed`. Чтобы скомпилировать шейдер, нам нужно байтики исходного кода шейдера передать функции [`ebiten.NewShader()`](https://pkg.go.dev/github.com/hajimehoshi/ebiten#NewShader).

## Раунд 2: pick-эффект

В интернете можно найти шрифты, которые выглядят как что-то рукописное. Однако каждая буква будет выглядеть идентично, что нереалистично. Нужна какая-то энтропия.

Достичь этой энтропии можно по-разному, но я в игре [Decipherism](https://quasilyte.itch.io/decipherism) просто рандомно перемешивал некоторые соседние пиксели при отрисовке текста:

<img title="Сравнение результата с шейдером и без" src="https://habrastorage.org/webt/n1/k-/yk/n1k-yk7m4ggcr1j4fzlhxkeoluc.png"/><p></p>

Давайте вспомним сигнатуру фрагментного шейдера (игнорируя неинтересные параметры):

```go
func Fragment(_ vec4, texCoord vec2, _ vec4) vec4
```

`texCoord` - это тексельная координата на текстуре, из которой мы читаем пиксели (source) для наложения на целевое изображение (destination).

О текселях нам достаточно знать то, что они имеют значение в диапазоне от 0 до 1. Условно, если изображение имеет размер 500 пикселей, то 0.5 текселей будут описывать размер в 250 пикселей в контексте этого изображения.

Функция `imageSrc0At()` принимает **тексельные** координаты. Но что, если мы хотим оперировать на уровне пикселей? Преобразования между текселями в пиксели и обратно возможны.

Ebitengine позволяет определять функции для шейдеров, чем мы и воспользуемся:

```go
// tex2pixCoord преобразует тексельную координату texCoord
// в пиксельную координату, учитывая смещение на атласе.
func tex2pixCoord(texCoord vec2) vec2 {
	pixSize := imageSrcTextureSize()
	originTexCoord, _ := imageSrcRegionOnTexture()
	actualTexCoord := texCoord - originTexCoord
	actualPixCoord := actualTexCoord * pixSize
	return actualPixCoord
}
```

Ebitengine объединяет несколько изображений в атласы, поэтому чаще всего наш source image находится на каком-то смещении от настоящей нулевой координаты. Из-за этого нам нужно вычитать origin для транслирования тексельной координаты в такую, которую мы затем можем интерпретировать как обычную пиксельную координату на изображении.

Алгоритм у нас будет такой:

1. Преобразуем тексели в пиксельные координаты;
2. Применяем логику над пикселями;
3. Конвертируем пиксели в тексели в самом финале.

Для последнего шага нужна будет обратная `tex2pixCoord()` операция:

```go
func pix2texCoord(actualPixCoord vec2) vec2 {
	pixSize := imageSrcTextureSize()
	actualTexCoord := actualPixCoord / pixSize
	originTexCoord, _ := imageSrcRegionOnTexture()
	texCoord := actualTexCoord + originTexCoord
	return texCoord
}
```

Далее нам нужно применить что-то вроде фильтра [pick](https://docs.gimp.org/2.10/nl/gimp-filter-noise-pick.html). Я могу предложить такую реализацию:

```go
func applyPixPick(pixCoord vec2, dist float, m, hash int) vec2 {
	// dist - на сколько пикселей сдвигаем;
	// dir - куда именно сдвигаем.
	// В Kage (язык шейдеров) пока нет switch,
	// поэтому используем if/else.
	dir := hash % m
	// Если явно не приводить литерал к int, то возникнет ошибка
	// "operands of `==' must have the same type",
	// потому что Ebitengine конвертирует литерал 0 в 0.0
	// и драйвер будет считать это типом float.
	if dir == int(0) {
		pixCoord.x += dist
	} else if dir == int(1) {
		pixCoord.x -= dist
	} else if dir == int(2) {
		pixCoord.y += dist
	} else if dir == int(3) {
		pixCoord.y -= dist
	}
	// А иначе никуда не сдвигаем.
	return pixCoord
}
```

> Чем выше параметр `m`, тем чаще пиксель не будет сдвигаться ни в одну из сторон.

Остаётся лишь один вопрос - а откуда взять `hash`? По идее, это некоторое псевдорандомное значение, которое определяет что делать с конкретным пикселем. Никакого [`rand()`](https://www.youtube.com/watch?v=dQw4w9WgXcQ) внутри шейдеров, конечно же, нет.

Напишем функцию генерации псевдослучайных чисел:

```go
func shaderRand(pixCoord vec2) int {
	return int(pixCoord.x+pixCoord.y) * int(pixCoord.y*5)
}
```

С помощью всех созданных выше функций выразим фрагментный процессор:

```go
func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	c := imageSrc0At(texCoord)
	actualPixCoord := tex2pixCoord(texCoord)
	if c.a != 0.0 {
		h := shaderRand(actualPixCoord)
		p := applyPixPick(actualPixCoord, 1.0, 15, h)
		return imageSrc0At(pix2texCoord(p))
	}
	return c
}
```

Этот шейдер будет производить желаемый нами pick-эффект.

## Раунд 3: эффект CRT-дисплея

В [Decipherism](https://quasilyte.itch.io/decipherism) мне нужно было реализовать терминальный экран, который выглядел бы в стиле ретро. На экране терминала выводились элементы схемы, реализующие некий кодирующий алгоритм.

Вот что из этого получилось:

<img title="Шейдеры включены" src="https://habrastorage.org/webt/zw/13/9r/zw139rxmwe6fvvlhxl861pdfje8.gif"/><p></p>

Если выключить шейдер:

<img title="Шейдеры выключены" src="https://habrastorage.org/webt/5g/m5/nv/5gm5nvpsn3ruarhtyvsxtl_aony.png"/><p></p>

Здесь нам потребуется более качественная генерация псевдорандомных чисел. Для этого мы введём два внешних параметра:

* `Tick` - некоторое скользящее со временем значение
* `Seed` - для каждого элемента будет создан свой сид для рандома

`shaderRand()` станет выглядеть следующим образом:

```go
func shaderRand(pixCoord vec2) (seedMod, randValue int) {
	pixSize := imageSrcTextureSize()
	pixelOffset := int(pixCoord.x) + int(pixCoord.y*pixSize.x)
	seedMod = pixelOffset % int(Seed)
	pixelOffset += seedMod
	return seedMod, pixelOffset + int(Seed)
}
```

> `seedMod` нам понадобится  как дополнительный источник энтропии.

Кроме этого, мы хотим создавать некие анимированные помехи. Я бы сказал, что это похоже на эффект [video degradation](https://docs.gimp.org/2.10/en/gimp-filter-video-degradation.html), но менее сильно выраженный.

```go
func applyVideoDegradation(y float, c vec4) vec4 {
	if c.a != 0.0 {
		// Каждый 4-ый пиксель по оси Y будет затенён.
		if int(y+Tick)%4 != int(0) {
			return c * 0.6
		}
	}
	return c
}
```

Финальный код фрагментного шейдера:

```go
func Fragment(pos vec4, texCoord vec2, _ vec4) vec4 {
	c := imageSrc0At(texCoord)

	actualPixCoord := tex2pixCoord(texCoord)
	if c.a != 0.0 {
		seedMod, h := shaderRand(actualPixCoord)
		dist := 1.0
		if seedMod == int(0) {
			dist = 2.0
		}
		p := applyPixPick(actualPixCoord, dist, 5, h)
		return applyVideoDegradation(pos.y, imageSrc0At(pix2texCoord(p)))
	}

	return c
}
```

Здесь я впервые использую параметр `pos`. Это позиция в целевом (destination) изображении в пикселях. Используя это значение я избегаю проблем при вращении source текстур. Таким образом, волны помех всегда идут сверху вниз, а не справа-налево, как в случае поворота на 90 градусов.

## Раунд 4: цикличная анимация текстуры

Возьмём текстуру энергетического луча:

<img title="Текстура для лазера" src="https://habrastorage.org/webt/ej/uq/d2/ejuqd2gdaviqspfyk1z4ler1cs8.png"/><p></p>

...и начнём циклично перемещать её по оси X:

<img title="Лазер с шейдером 1" src="https://habrastorage.org/webt/eu/wy/lh/euwylhgbj0jb5q5pe2gbtxvm_pw.gif"/><p></p>

Вот ещё пример:

<img title="Лазер с шейдером 2" src="https://habrastorage.org/webt/wt/cl/fm/wtclfmg0rico_9la1n0oqbfl_w0.gif"/><p></p>

Первая попытка решения:

```go
var Time float

func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	pixSize := imageSrcTextureSize()
	_, srcRegion := imageSrcRegionOnTexture()
	width := pixSize.x * srcRegion.x
	actualPixCoord := tex2pixCoord(texCoord)
	p := vec2(slide(actualPixCoord.x, width), actualPixCoord.y)
	return imageSrc0At(pix2texCoord(p))
}

func slide(v, size float) float {
	return mod(v-(100*Time), size)
}
```

Результат применения:

<img src="https://habrastorage.org/webt/se/qw/o2/seqwo2qrzblwbv5dafni5trc344.gif"/>

> Направление движения анимации зависит от того, уменьшается или увеличивается `Time`.

Это почти то, что нам нужно, но цикл получается резким из-за грубого перехода на обоих концах отрезка. Чтобы получить результат, как в примерах выше, нужно добавить немного кода в этот шейдер:

```go
func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	pixSize := imageSrcTextureSize()
	_, srcRegion := imageSrcRegionOnTexture()
	width := pixSize.x * srcRegion.x
	actualPixCoord := tex2pixCoord(texCoord)
	p := vec2(slide(actualPixCoord.x, width), actualPixCoord.y)

	c := imageSrc0At(pix2texCoord(p))
	const cutoffThreshold = 10.0
	if actualPixCoord.x <= cutoffThreshold {
		c *= actualPixCoord.x * 0.1
	} else if actualPixCoord.x >= (width - cutoffThreshold) {
		c *= (width - actualPixCoord.x) * 0.1
	}

	return c
}
```

Мы добавили градиент, уменьшающий непрозрачность изображения. Чем ближе к концам отрезка, тем выше прозрачность.

А знаете, что ещё можно реализовать через похожий шейдер? Планеты. Нам потребуется прямоугольная текстура.

<img src="https://habrastorage.org/webt/kx/y9/mo/kxy9moqosqov78akosruiifoql0.png"/><p></p>

Шейдер будет похож на предыдущие, но с добавлением тени и радиуса отрисовки:

```go
var Time float

func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	_, srcRegion := imageSrcRegionOnTexture()
	pixSize := imageSrcTextureSize()
	sizes := pixSize * srcRegion
	width := sizes.x
	height := sizes.y
	actualPixCoord := tex2pixCoord(texCoord)

	// То, что дальше радиуса окружности (32) мы рендерить не будем.
	// Так мы оставляем из всей текстуры только центральную часть.
	const planetSize = 64.0
	center := vec2(width, height) * 0.5
	if distance(center, actualPixCoord) > planetSize {
		return vec4(0)
	}

	// Свет будет падать чуть левее и выше от центра.
	lightPos := vec2(center.x*0.85, center.y*0.9)
	lightDist := distance(lightPos, actualPixCoord) / planetSize
	colorMultiplier := vec4(1, 1, 1, 1)
	// Чем больше дистанция от освещённой точки, тем темнее будет цвет.
	colorMultiplier.xyz *= clamp(1.8-lightDist*1.6, 0.0, 1.0)

	// А дальше применяем уже известную нам анимацию.
	p := vec2(slide(actualPixCoord.x, width), actualPixCoord.y)
	return imageSrc0At(pix2texCoord(p)) * colorMultiplier
}
```

Результат применения шейдера:

<img src="https://habrastorage.org/webt/n9/zi/qs/n9ziqsfxvnbx3zikgdjvtqpdto4.gif"/>

## Раунд 5: эффект построения здания

В [Roboden](https://quasilyte.itch.io/roboden) можно строить базы и турели. Анимация конструирования нового здания сделана через шейдеры.

В игре это выгдядит следующим образом:

<img title="Строительство колонии в Roboden" src="https://habrastorage.org/webt/-5/0h/ol/-50holsyyz4vg7in_qhx92hyuv8.gif"/><p></p>

Для удобства, вот фреймы из анимации выше, в изоляции:

<img title="Несколько стадий строительства" src="https://habrastorage.org/webt/if/eh/et/ifehetkbibgsa46_s3emuxu3kjk.png"/><p></p>

Параметр `t` (в шейдере назван `Time`) управляется логикой игры. Когда рабочие строят здание, `t` увеличивается. `t` - это нормализованное значение прогресса строительства (от 0 до 1).

Шейдер будет представлять из себя смесь того, что мы сегодня уже использовали:

* Отрисовка только той части текстуры, что находится за пределами окружности;
* Граница отрисовки затемняется, чтобы создать ощущение объёма;
* Пиксели близ контура перемещаются (эффект pick) и перекрашиваются.

Начнём с введения хелпер-функций:

```go
func shaderRand(p vec2) int {
	return int(p.x+p.y) * int(p.y*5)
}

func sourceSize() vec2 {
	pixSize := imageSrcTextureSize()
	_, srcRegion := imageSrcRegionOnTexture()
	return pixSize.x * srcRegion
}
```

Сам шейдер имеет много параметров, которые я вручную подбирал для желаемого результата. Специально для статьи я немного изменил его, чтобы он стал более универсальным.

```go
func Fragment(_ vec4, texCoord vec2, _ vec4) vec4 {
	// texCoord гарантированно в пределах Src0, поэтому можно
	// использовать unsafe версию, которая работает немного быстрее,
	// но out-of-bounds доступ будет вести к неопределённому поведению.
	c := imageSrc0UnsafeAt(texCoord)
	if c.a == 0 {
		return c
	}

	actualPixPos := tex2pixCoord(texCoord)

	// Вычисления будем завязывать на вычисляемый размер текстуры.
	// Это позволит использовать шейдер для изображений разного размера.
	sizes := sourceSize()
	width := sizes.x // Изображение квадратное, поэтому достаточно width

	// Задаём окружность прорисовки и её перемещение по dt.
	initialY := -2.0
	offsetY := width * 0.15 * Time
	circleCenter := vec2(width*0.5, initialY-offsetY)
	dist := distance(actualPixPos, circleCenter)

	progress := 1.4 - Time
	if dist > ((width * 0.95) * progress) {
		// То, что уже далеко от окружности, рисуем без искажений.
		return c
	}

	spread := 0
	colorMultiplier := vec4(0)

	// Определим несколько колец по диапазону дистанций.
	// Свича нет, поэтому идём через if/else.
	if dist > ((width * 0.85) * progress) {
		spread = 15
		colorMultiplier = vec4(1, 1.1, 1.3, 1.0)
	} else if dist > ((width * 0.75) * progress) {
		spread = 11
		colorMultiplier = vec4(0.9, 1.2, 1.6, 1.0)
	} else if dist > ((width * 0.65) * progress) {
		spread = 7
		colorMultiplier = vec4(0.8, 1.4, 2.0, 1.0)
	} else if dist > ((width * 0.62) * progress) {
		spread = 6
		colorMultiplier = vec4(0.25, 0.25, 0.25, 1.0)
	} else {
		// Слишком близко к окружности, эту область пропускаем.
		return vec4(0)
	}

	h := shaderRand(actualPixPos)
	p := applyPixPick(actualPixPos, 1, spread, h)
	if p == actualPixPos {
		// Если пиксель не переместился, рисуем его без изменения цвета.
		return c
	}
	return imageSrc0At(pix2texCoord(p)) * colorMultiplier
}
```

С увеличением `Time` мы смещаем абстрактную окружность вверх, что меняет распределение отображаемых пикселей из-за обновлённой дистанции от центра окружности.

Это был последний из шейдеров, который я хотел вам показать в рамках этой статьи.

Хочется ещё шейдеров? Откройте [examples/shader](https://github.com/hajimehoshi/ebiten/tree/main/examples/shader) из репозитория Ebitengine, там можно найти:

* Расстворение (dissolve)
* Радиальное размытие
* Эффект отражений в воде
* Хроматическую аберрацию

Напоследок поделюсь с вами несколькими рекомендациями по работе с шейдерами в Ebitengine:

* Храните исходники шейдеров внутри бинарника, через `go:embed`.
* Компилируйте каждый шейдер только один раз, переиспользуйте `*ebiten.Shader`.
* Когда шейдер не нужен*, рисуйте через `DrawImage`, а не `DrawRectShader`.
* Пишите больше хелпер-функций в шейдерах, они значительно улучшают читабельность.
* Работайте на уровне пикселей, если это делает алгоритм понятнее.

> (*) Пример: маска повреждения при `HP=1.0` не будет менять отображение, поэтому можно рисовать спрайт через `DrawImage()`, а не `DrawRectShader()`.

## Чек-лист для заинтересованных

Хотите попробовать писать игрушки на Go, но не знаете, с чего начать?

1. Вступайте в [русскоязычное сообщество разработки игр на Go](https://t.me/go_gamedev).
2. Если знаете английский, [подключайтесь к официальному дискорду](https://discord.gg/3tVdM5H8cC).
3. Пройдите [мини-тур](https://ebitengine.org/en/tour/) по Ebitengine.
4. Придумайте идею для своей первой игры на Go, что-то не очень сложное.

Параллельно с этим:

* По мере создания игры, подглядывайте в [examples](https://github.com/hajimehoshi/ebiten/tree/main/examples).
* Если ищете 3rd-party библиотеки, загляните в [awesome-ebitengine](https://github.com/sedyh/awesome-ebitengine).
* Если не справляетесь, пишите в сообществах, вам точно помогут.

Понравилась эта статья и вы хотите сказать автору спасибо? Ставьте плюсик. Мне ещё есть, о чём рассказать, а ваша поддержка может увеличить шанс появления следующих текстов из серии.

Если хочется порадовать меня ещё сильнее, то можете посмотреть на мои игры. Они все с открытыми исходными кодами, но только две из них имеют отдельный репозиторий. Ссылочки в конце статьи.

## Ссылки и полезные ресурсы

* [Ebitengine](https://github.com/hajimehoshi/ebiten)
* [Русскоязычное сообщество разработки игр на Go](https://t.me/go_gamedev)
* [Kage desk](https://github.com/tinne26/kage-desk) - англоязычные материалы по языку шейдеров
* [Страница документации шейдеров из Ebitengine](https://ebitengine.org/en/documents/shader.html)
* [Репозиторий игры Roboden](https://github.com/quasilyte/roboden-game)
* [Репозиторий игры Decipherism](https://github.com/quasilyte/decipherism-game/)
* [awesome-ebitengine](https://github.com/sedyh/awesome-ebitengine)
