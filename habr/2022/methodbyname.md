# Ускоряем hugo на 20% простым изменением в пакете reflect

Найти значительное узкое место в производительности стандартной библиотеки или зрелого приложения - это редкость.

Я был удивлён, когда в top10 списке CPU-профиля [hugo](https://github.com/gohugoio/hugo) при сборке [digitalgov.gov](https://github.com/GSA/digitalgov.gov) на первой позиции находился метод `reflect.Type.MethodByName()`.

```
      flat  flat%   sum%        cum   cum%
     8.84s  6.28%  6.28%     57.85s 41.10%  reflect.(*rtype).MethodByName
     7.93s  5.63% 11.92%      8.50s  6.04%  reflect.name.readVarint
     7.56s  5.37% 17.29%    111.79s 79.43%  reflect.Value.call
     7.53s  5.35% 22.64%     23.33s 16.58%  runtime.mallocgc
     7.29s  5.18% 27.82%     16.10s 11.44%  reflect.name.name
```

В этой статье я расскажу вам о том, как так вышло и что с этим можно было бы сделать.

<cut/>

## Почему MethodByName такой медленный

Давайте откроем исходники MethodByName:

```go
func (t *rtype) MethodByName(name string) (m Method, ok bool) {
	// <лишний код вырезан>

	// TODO(mdempsky): Binary search.
	for i, p := range ut.exportedMethods() {
		if t.nameOff(p.name).name() == name {
			return t.Method(i), true
		}
	}
	return Method{}, false
}
```

MethodByName использует линейный поиск. Если экспортируемых методов у типа много, то работать это может очень долго, особенно если искомый метод будет находиться где-то в конце слайса.

## Как можно ускорить MethodByName

```go
func (t *rtype) MethodByName(name string) (m Method, ok bool) {
	// <лишний код вырезан>

	if len(methods) > 8 {
		i := sort.Search(len(methods), func(i int) bool {
			return t.nameOff(methods[i].name).name() >= name
		})
		if i < len(methods) && t.nameOff(methods[i].name).name() == name {
			return t.Method(i), true
		}
	} else {
		for i, p := range methods {
			if t.nameOff(p.name).name() == name {
				return t.Method(i), true
			}
		}
	}
	return Method{}, false
}
```

Мы оставили линейный поиск для малых слайсов, чтобы избежать замедления. На более крупных размерах бинарный поиск выигрывает почти в любой ситуации, кроме разве что случаев, когда искомый метод находится в самом начале слайса.

Если мы напишем бенчмарки, то получим примерно такие цифры:

```
name                           old time/op  new time/op  delta
MethodByName/4_first-8          426ns ± 2%   423ns ± 1%     ~   
MethodByName/4_last-8           482ns ± 1%   476ns ± 1%   -1.33%
MethodByName/4_nonexisting-8   57.8ns ± 0%  57.3ns ± 1%   -0.78%
MethodByName/16_first-8         427ns ± 1%   538ns ± 0%  +25.97%
MethodByName/16_last-8          643ns ± 1%   519ns ± 1%  -19.24%
MethodByName/16_nonexisting-8   194ns ± 0%   105ns ± 0%  -46.03%
MethodByName/32_first-8         429ns ± 1%   552ns ± 1%  +28.78%
MethodByName/32_last-8          831ns ± 0%   540ns ± 2%  -34.97%
MethodByName/32_nonexisting-8   396ns ± 1%   125ns ± 1%  -68.56%
MethodByName/64_first-8         431ns ± 1%   580ns ± 2%  +34.64%
MethodByName/64_last-8         1.36µs ± 1%  0.56µs ± 1%  -58.95%
MethodByName/64_nonexisting-8   759ns ± 0%   146ns ± 1%  -80.82%
[Geo mean]                      430ns        303ns       -29.58%
```

Как и ожидалось, новый метод работает хуже для случаев, когда искомый метод находится в начале слайса. Если учесть, что это вырожденный случаи и в среднем метод будет находиться где-то в середине, то выигрыш довольно очевиден.

## Проверяем влияние на hugo

Изначально, указанный выше сайт собирался за `20.4` секунды.

Соберём hugo с обновлённым пакетом `reflect` и запустим сборку сайта ещё раз.

Невероятно, сборка сайта теперь занимает `16.5` секунд!

А что же там с профилем?

```
(pprof) top 20
Showing nodes accounting for 50.85s, 44.61% of 114s total
Dropped 1174 nodes (cum <= 0.57s)
Showing top 20 nodes out of 194
      flat  flat%   sum%        cum   cum%
     8.64s  7.58%  7.58%     25.01s 21.94%  runtime.mallocgc
     7.69s  6.75% 14.32%     87.02s 76.33%  reflect.Value.call
     4.17s  3.66% 17.98%      5.39s  4.73%  runtime.heapBitsSetType
     2.88s  2.53% 20.51%         4s  3.51%  runtime.findObject
     2.83s  2.48% 22.99%      2.83s  2.48%  runtime.nextFreeFast (inline)
     2.79s  2.45% 25.44%      9.46s  8.30%  runtime.scanobject
     2.74s  2.40% 27.84%      2.74s  2.40%  runtime.duffcopy
     1.82s  1.60% 29.44%      1.88s  1.65%  runtime.pageIndexOf
     1.76s  1.54% 30.98%      1.76s  1.54%  runtime.memclrNoHeapPointers
     1.69s  1.48% 32.46%      3.24s  2.84%  runtime.mapaccess2
     1.66s  1.46% 33.92%      1.83s  1.61%  syscall.Syscall
     1.49s  1.31% 35.23%      1.61s  1.41%  reflect.name.readVarint
     1.43s  1.25% 36.48%      3.07s  2.69%  reflect.name.name
     1.42s  1.25% 37.73%     10.54s  9.25%  reflect.FuncOf
     1.37s  1.20% 38.93%      1.75s  1.54%  github.com/gohugoio/hugo/...
     1.34s  1.18% 40.11%      4.99s  4.38%  sync.(*Map).Load
     1.33s  1.17% 41.27%      1.33s  1.17%  reflect.(*rtype).Kind
     1.29s  1.13% 42.40%      1.29s  1.13%  cmpbody
     1.28s  1.12% 43.53%      1.28s  1.12%  runtime.resolveNameOff
     1.23s  1.08% 44.61%      1.30s  1.14%  runtime.heapBitsForAddr
```

MethodByName, который был на 1-ой позиции, теперь не входит даже в top20.

Но нам стоит копнуть глубже, чтобы делать какие-либо выводы. Давайте посчитаем сколько там было вызовов MethodByName и каково распределение размеров слайсов exportedMethods и позиций, на которых метод был найден.

Общее количество вызовов за сборку сайта: 17042238.

```
15920637 times | 120 methods
493669 times | 29 methods
227736 times | 16 methods
180098 times | 8 methods
143846 times | 39 methods
31371 times | 6 methods
14636 times | 1 methods
10933 times | 31 methods
10094 times | 9 methods
9026 times | 4 methods
102 times | 113 methods
85 times | 7 methods
4 times | 114 methods
1 times | 0 methods
```

Большая часть вызовов была над типом, у которого 120 экспортируемых методов.

```
12998952 times | 98 index pos
1203864 times | 70 index pos
515978 times | 102 index pos
354191 times | 110 index pos
262965 times | 6 index pos
200819 times | 11 index pos
178457 times | 19 index pos
170806 times | 97 index pos
147510 times | 74 index pos
136927 times | 3 index pos
123071 times | 20 index pos
106250 times | 9 index pos
87190 times | 7 index pos
62767 times | 31 index pos
62192 times | 5 index pos
50170 times | 51 index pos
49120 times | 69 index pos
45325 times | 72 index pos
42236 times | 115 index pos
38646 times | 0 index pos
37106 times | 28 index pos
22744 times | 108 index pos
21521 times | 14 index pos
18134 times | 92 index pos
17341 times | 4 index pos
11535 times | 103 index pos
10791 times | 63 index pos
10303 times | 2 index pos
9170 times | 10 index pos
7278 times | 68 index pos
5936 times | 15 index pos
5466 times | 65 index pos
5385 times | 16 index pos
5172 times | 47 index pos
4478 times | 43 index pos
3032 times | 40 index pos
2340 times | 17 index pos
2265 times | 25 index pos
1516 times | 44 index pos
1068 times | 88 index pos
843 times | 1 index pos
709 times | 71 index pos
358 times | 109 index pos
100 times | 67 index pos
50 times | 64 index pos
40 times | 42 index pos
33 times | 34 index pos
17 times | 86 index pos
15 times | 83 index pos
15 times | 41 index pos
15 times | 33 index pos
12 times | 118 index pos
5 times | 8 index pos
5 times | 27 index pos
4 times | 66 index pos
```

Распределение индексов искомых методов тоже подсказывает, что линейный поиск в данном случае - не лучший выбор. Большая часть методов находилась на позиции 98.

## Подводные камни

В коде выше я использовал `sort.Search`, но он не доступен для пакета `reflect`.

Дело в том, что пакет `sort` импортирует `reflect`, поэтому мы получаем ошибку циклических импортов.

Чтобы этого избежать, можно продублировать реализацию `sort.Search` в пакете `reflect` или создать новый internal пакет для Go, где будет некоторое подмножество пакета `sort`, не зависящее от `reflect`.

Этот фактор немного затрудняет внедрение предлагаемого патча, ведь самый красивый вариант реализации нам недоступен.

## Выводы

Я выслал [CL](https://go-review.googlesource.com/c/go/+/378634/) с изменениями в Go, но не знаю, примут его или нет.

В любом случае, было забавно увидеть как такая безобидная на первый взгляд операция замедляла сборку статического сайта на какое-то значительное время.

Немного другого я ожидал от hugo с заголовком "The world’s fastest framework for building websites."
