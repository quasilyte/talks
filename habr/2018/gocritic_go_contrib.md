# Контрибьютим в Go с помощью статического анализатора go-critic

![](https://habrastorage.org/webt/fp/k0/rn/fpk0rncwtlo5ozovjev4yi_gok8.png)

Вы, возможно, помните недавний [анонс нового статического анализатора для Go](https://habr.com/post/414739/) под названием [go-critic](https://github.com/go-critic/go-critic).

Я проверил с его помощью проект [golang/go](https://github.com/golang/go) и отправил несколько патчей, которые исправляют некоторые найденные там проблемы.

В этой статье мы разберём исправленный код, а также будем мотивироваться отправлять ещё больше подобных изменений в Go.

Для самых нетерпеливых: [обновляемый список трофеев](https://go-critic.github.io/trophies.html#golanggo).

<cut/>

<spoiler title="Список разбираемых патчей в Go">
1. [net: combine append calls in reverseaddr](https://golang.org/cl/117615) `appendCombine`
1. [cmd/link/internal/ld: avoid Reloc copies in range loops](https://golang.org/cl/113636) `rangeValCopy`
1. [cmd/compile/internal/ssa: fix partsByVarOffset.Less method](https://golang.org/cl/122776) `dupSubExpr`
1. [runtime: remove redundant explicit deref in trace.go](https://golang.org/cl/122895) `underef`
1. [cmd/link/internal/sym: uncomment code for ELF cases in RelocName](https://golang.org/cl/122896) `commentedOutCode`
1. [runtime: simplify slice expression to sliced value itself](https://go-review.googlesource.com/c/go/+/123375) `unslice`
1. [html/template: use named consts instead of their values](https://go-review.googlesource.com/c/go/+/123376) `namedConst`
1. [cmd/internal/obj/arm64: simplify some bool expressions](https://go-review.googlesource.com/c/go/+/123377) `boolExprSimplify`
1. [math,net: omit explicit true tag expr in switch](https://go-review.googlesource.com/c/go/+/123378) `switchTrue`
1. [archive/tar: remore redundant parens in type expressions](https://go-review.googlesource.com/c/go/+/123379) `typeUnparen`
</spoiler>

# dupSubExpr

Все мы допускаем ошибки, и, довольно часто, по невнимательности. Go, будучи языком, в котором временами приходится писать скучный и шаблонный код, порой способствует опечаткам и/или ошибкам категории copy/paste.

[CL122776](https://go-review.googlesource.com/c/go/+/122776) содержит исправление бага, найденного проверкой [dupSubExpr](https://go-critic.github.io/overview#dupSubExpr-ref):

```diff
func (a partsByVarOffset) Less(i, j int) bool {
-	return varOffset(a.slots[a.slotIDs[i]]) < varOffset(a.slots[a.slotIDs[i]])
+	return varOffset(a.slots[a.slotIDs[i]]) < varOffset(a.slots[a.slotIDs[j]])
//                                     ^__________________________________^
}
```

Обратите внимание на индекс слева и справа. До исправления, LHS и RHS оператора `<` были идентичны, на что и сработал `dupSubExpr`.

# commentedOutCode

Если ваш проект под эгидой [системой контроля версий](https://en.wikipedia.org/wiki/Version_control_system), то вместо отключения кода путём оборачивания его в комментарий, стоит удалять его полностью. Есть исключения, но чаще такой "мёртвый" код мешает, путает и может скрывать ошибки.

[commentedOutCode](https://go-critic.github.io/overview#commentedOutCode-ref) смог найти вот такой занимательный фрагмент ([CL122896](https://go-review.googlesource.com/c/go/+/122896)):

```go
switch arch.Family {
// ... другие case clause.
case sys.I386:
	return elf.R_386(nr).String()
case sys.MIPS, sys.MIPS64:
	// return elf.R_MIPS(nr).String()   // <- 1
case sys.PPC64:
	// return elf.R_PPC64(nr).String()  // <- 2
case sys.S390X:
	// return elf.R_390(nr).String()    // <- 3
default:
	panic("unreachable")
}
```

Немного выше есть комментарий:

```go
// We didn't have some relocation types at Go1.4.
// Uncomment code when we include those in bootstrap code.
```

Если переключиться на ветку `go1.4` и убрать эти 3 строки из-под комментария, код не будет компилироваться, однако если раскомментировать их на мастере, всё будет работать.

Обычно код, спрятанный в комментарий, требует либо удаления, либо обратной активации.

Полезно, время от времени, навещать такие отголоски прошлого в вашем коде.

<spoiler title="О сложностях детектирования">
Эта одна из самых моих любимых проверок, но она же является одной из самых "шумных".

Очень много ложных срабатываний для пакетов, которые используют `math/big` и внутри компилятора. В первом случае обычно это поясняющие комментарии к выполняемым операциям, а во втором - описание кода, который описывает фрагмент AST. Различить такие комментарии от реального "мёртвого" кода, без внесения false negatives - нетривиально.

Отсюда возникает идея: а что если нам договориться как-то по-особому оформлять код внутри комментариев, который является пояснительным? Тогда статический анализ упростится. Это может быть любая мелочь, которая или позволит легко определить такой пояснительный комментарий, или сделает его невалидным Go кодом (например, если добавлять в начало строки знак решётки, `#`).

Другая категория - это комментарии с явным `TODO`. Если код убран под комментарий, но при этом есть чёткое описание почему это сделано и когда планируется починить этот кусок кода, то лучше не выдавать предупреждения. Это уже реализовано, но могло бы работать более надёжно.
</spoiler>

# boolExprSimplify

Иногда люди пишут странный код. Возможно мне кажется, но особо странно иногда выглядят логические ([булевы](https://en.wikipedia.org/wiki/Boolean_algebra)) выражения.

У Go отличный бэкенд ассемблера x86 (тут упала слеза), а вот ARM провинился по-настоящему:
```go
if !(o1 != 0) {
	break
}
```

"Если не o1 не равно 0"... Двойное отрицание - это классика. Если вам понравилось, приглашаю ознакомиться с [CL123377](https://go-review.googlesource.com/c/go/+/123377). Там вы сможете увидеть исправленный вариант.

<spoiler title="Исправленный вариант (для тех, кого не заманить на go-review)">
```diff
- if !(o1 != 0) {
+ if o1 == 0 {
```
</spoiler>

[boolExprSimplify](https://go-critic.github.io/overview#boolExprSimplify-ref) нацелен на упрощения, которые повышают читабельность (а с вопросом производительности оптимизатор Go справился бы и без этого).

# underef

Если вы используете Go с его ранних версий, то можете помнить обязательные точки с запятыми, отсутсвие автоматического разыменования указателей, и прочие особенности, которые сегодня увидеть в новом коде практически невозможно.

В старом же коде до сих пор можно встретить нечто подобное:

```go
// Когда-то явное разыменование было обязательным:
buf := (*bufp).ptr()
// ...теперь же можно упростить выражение до:
buf := bufp.ptr()
```

Несколько срабатываний [underef](https://go-critic.github.io/overview#underef-ref) анализатора исправлены в [CL122895](https://go-review.googlesource.com/c/go/+/122895).

# appendCombine

Возможно, вы знаете, что `append` может принимать несколько аргументов в качестве элементов, добавляемых в целевой слайс. В некоторых ситуациях это позволяет немного улучшить читабельность кода, но, что может быть интереснее, это также может ускорить вашу программу, поскольку компилятор не выполняет схлапывание совместимых вызовов `append` ([cmd/compile: combine append calls](https://github.com/golang/go/issues/25828)).

В Go проверка [appendCombine](https://go-critic.github.io/overview#appendCombine-ref) нашла следующий участок:

```diff
- for i := len(ip) - 1; i >= 0; i-- {
- 	v := ip[i]
- 	buf = append(buf, hexDigit[v&0xF])
- 	buf = append(buf, '.')
- 	buf = append(buf, hexDigit[v>>4])
- 	buf = append(buf, '.')
- }
+ for i := len(ip) - 1; i >= 0; i-- {
+ 	v := ip[i]
+ 	buf = append(buf, hexDigit[v&0xF],
+ 		'.',
+ 		hexDigit[v>>4],
+ 		'.')
+ }
```

```
name              old time/op  new time/op  delta
ReverseAddress-8  4.10µs ± 3%  3.94µs ± 1%  -3.81%  (p=0.000 n=10+9)
```

Подробности в [CL117615](https://go-review.googlesource.com/c/go/+/117615).

# rangeValCopy

Ни для кого не секрет, что значения, перебираемые в `range` цикле, копируются. Для объектов малого размера, скажем, меньше 64 байт, вы можете этого даже не заметить. Однако если подобный цикл лежит на "горячем" пути, или , над которым вы итерируетесь, содержит очень большое количество элементов, накладные расходы могут быть осязаемы.

В Go довольно медленный [линковщик](https://en.wikipedia.org/wiki/Linker_(computing)) (cmd/link), и без значительных изменений в его архитектуре, сильного прироста производительности добиться нельзя. Но зато можно немного уменьшать его неэффективность с помощью микрооптимизаций. Каждый процент-два на счету.

Проверка [rangeValCopy](https://go-critic.github.io/overview#rangeValCopy-ref) нашла сразу несколько циклов с нежелательным копированием данных. Вот наиболее интересный из них:

```diff
- for _, r := range exports.R {
- 	d.mark(r.Sym, nil)
- }
+ for i := range exports.R {
+ 	d.mark(exports.R[i].Sym, nil)
+ }
```

Вместо того, чтобы копировать сам `R[i]` на каждой итерации, мы лишь обращаемся к единственному интересующему нас члену, `Sym`.

```
name      old time/op  new time/op  delta
Linker-4   530ms ± 2%   521ms ± 3%  -1.80%  (p=0.000 n=17+20)
```

Полная версия патча доступена по ссылке: [CL113636](https://go-review.googlesource.com/c/go/+/113636).

# namedConst

В Go, к сожалению, именованные константы, даже собранные в группы, не связаны между собой и не образуют перечисление ([proposal: spec: add typed enum support](https://github.com/golang/go/issues/19814)).

Одной из проблем является приведение [untyped констант](https://golang.org/ref/spec#Constants) к типу, который вы хотели бы использовать как enum.

Допустим, вы определили тип `Color`, у него есть значение `const ColDefault Color = 0`.<br>
Какой из этих двух фрагментов кода вам нравится больше?

```go
// (A)
if color == 0 {
    return colorBlack
}
// (B)
if color == colorDefault {
    return colorBlack
}
```

Если `(B)` вам кажется более уместным, проверка [namedConst](https://go-critic.github.io/overview#namedConst-ref) поможет вам отследить использование значений именованных констант в обход самой именованной константе.

Вот так преобразился метод [`context.mangle`](https://github.com/golang/go/blob/631402f142e52f535b66864ad1957ef39c78c704/src/html/template/context.go#L45) из пакета `html/template`:

```diff
 	s := templateName + "$htmltemplate_" + c.state.String()
-	if c.delim != 0 {
+	if c.delim != delimNone {
 		s += "_" + c.delim.String()
 	}
-	if c.urlPart != 0 {
+	if c.urlPart != urlPartNone {
 		s += "_" + c.urlPart.String()
 	}
-	if c.jsCtx != 0 {
+	if c.jsCtx != jsCtxRegexp {
 		s += "_" + c.jsCtx.String()
 	}
-	if c.attr != 0 {
+	if c.attr != attrNone {
 		s += "_" + c.attr.String()
 	}
-	if c.element != 0 {
+	if c.element != elementNone {
 		s += "_" + c.element.String()
 	}
 	return s
```

Кстати, иногда по ссылкам на патчи можно найти интересные обсуждения...<br>
[CL123376](https://go-review.googlesource.com/c/go/+/123376) - один из таких случаев.

# unslice

Особенность [slice expression](https://golang.org/ref/spec#Slice_expressions) в том, что `x[:]` всегда идентично `x`, если тип `x` - это слайс или строка. В случае слайсов это работает для любого типа элемента `[]T`.

Всё в списке ниже - одно и то же (`x` - слайс):
* `x`
* `x[:]`
* `x[:][:]`
* ...

[unslice](https://go-critic.github.io/overview#unslice-ref) находит подобные избыточные выражения срезов. Вредны эти выражения прежде всего лишней когнитивной нагрузкой. `x[:]` имеет вполне весомую семантику в случае взятия среза у массива. Срез среза с диапазонами по умолчанию не вносит ничего, кроме шума.

За патчем прошу в [CL123375](https://go-review.googlesource.com/c/go/+/123375).

# switchTrue

В [CL123378](https://go-review.googlesource.com/c/go/+/123378) выполнена замена "`switch true {...}`" на "`switch {...}`".<br>
Обе формы эквивалентны, но вторая является более идиоматичной.<br>
Найдено с помощью проверки [switchTrue](https://go-critic.github.io/overview#switchtrue).

Большинство стилистических проверок выявляют именно подобные случаи, где допустимы оба варианта, но один из них является более распространённым и привычным большему количеству Go программистов. Следующая проверка из этой же серии.

# typeUnparen

Go, как и многие другие языки программирования, любит скобочки. Настолько, что готов принять любое их количество:

```go
type (
    t0 int
    t1 (int)
    t2 ((int))
    // ... ну, вы поняли.
)
```

Но что произойдёт, если запустить `gofmt`?

```go
type (
	t0 int
	t1 (int) // <- Эй! Ничего не изменилось.
    t2 (int) // <- Выполнена только половина работы...
)
```

Вот поэтому и существует [typeUnparen](https://go-critic.github.io/overview#typeunparen). Он находит в программе все выражения типов, в которых можно уменьшить количество скобочек. Я попробовал выслать [CL123379](https://go-review.googlesource.com/c/go/+/123379), посмотрим, как его примет сообщество.

<spoiler title="Лиспы не любят скобочки">
В отличие от C-подобных языков, в Лиспах не так просто вставить бесполезные скобочки в произвольное место. Так что в языках, у которых синтаксис основан на [S-выражениях](https://en.wikipedia.org/wiki/S-expression), написать ничего не делающую программу, но имеющую огромное количество скобочек, сложнее, чем на некоторых других языках.
</spoiler>

# go-critic на службе Go

<a href="https://habrastorage.org/webt/x-/ol/cb/x-olcbpvhqsf1vs0crntwfec83y.jpeg" title="Нажмите для увеличения"><img src="https://habrastorage.org/webt/bq/0a/ig/bq0aigrftmrocldewkgzuhiycdo.jpeg"></a>


Мы рассмотрели только малую часть реализованных проверок. При этом их количество и качество будет только эволюционировать со временем, в том числе благодаря тем людям, которые [присоединились к разработке](https://github.com/go-critic/go-critic/graphs/contributors).

[go-critic](https://github.com/go-critic/go-critic) абсолютно бесплатен для любого использования (лицензия [MIT](https://github.com/go-critic/go-critic/blob/master/LICENSE)), а также открыт для вашего участия в развитии проекта. Присылайте нам идеи для проверок, можно сразу с реализацией, докладывайте о найденных ошибках и недоработках, делитесь впечатлениями. Можете также предлагать проекты для аудита или докладывать о проведённых вами ревью Go кода, этот опыт для нас бесценен.

## Тема контрибьютинга в Go

Видели статью [Go contribution workshop в России](https://habr.com/post/413815/)? Этой осенью будет второй раунд. И на этот раз, кроме более удачного формата и спонсоров, у нас будет секретное оружие - замечательный статический анализатор. Контрибьюций хватит на всех!

Но на самом деле, начинать вы можете прямо сейчас (хотя лучше - немного позже, после [code freeze](https://github.com/golang/go/wiki/Go-Release-Cycle)). Если получится освоиться до следующего воркшопа, будет очень здорово, потому что нам сильно нехватает менторов в России.
