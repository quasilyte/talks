# Путешествие gocritic'а в прошлое

![](https://habrastorage.org/webt/de/ui/zg/deuizg9w3z_5h9mc9him0m50a3e.png)

Хочу поделиться результатами работы последних нескольких дней, которые включали в себя анализ [git](https://git-scm.com/) истории некоторых крупных [Go](https://golang.org/) проектов с целью нахождения коммитов, которые исправляли ошибки c последующей их формализацией для детектирования в случае их появления в новом коде.

Во второй части статьи рассмотрим некоторые новые диагностики в [go-critic](https://github.com/go-critic/go-critic/), которые позволяют находить код, который с высокой степенью вероятности содержит ошибку.

<cut/>

# Поиск идей в прошлом

Чтобы находить идеи для новых инспекций кода, можно подглядывать в статические анализаторы для других языков программирования, можно вручную исследовать open source код с попыткой найти в нём ошибочные шаблоны кода, а можно заглянуть в историю.

Предположим, вы проверяете проект `Go-Files`. После запуска линтера (статического анализатора) проблемы не находятся, аудит исходных текстов тоже не принёс больших результатов. Стоит ли торопиться откладывать `Go-Files` в сторону? Не совсем.

За время существования `Go-Files` имел какое-то количество легко детектируемых дефектов, но сейчас они уже исправлены. То, что нам нужно сделать - это начать анализировать git историю репозитория.

![](https://habrastorage.org/webt/jh/mn/nb/jhmnnbbrjp-yrbpqsklp25md5ju.png)

Баги где-то рядом.

# Сбор интересующих нас коммитов

**Проблемы:**

* Коммитов может быть очень много
* Иногда в одной и той же ревизии выполнены не связанные друг с другом вещи
* Иногда комментарий к коммиту не совсем соответствует фактическим изменениям
* Некоторые недооценивают пользу хороших commit message (не будем показывать пальцем)

В первую очередь мы хотим снизить количество информации, которую придётся обрабатывать без автоматизации. Проблему false-negative срабатываний можно решить увеличением объёма выборки (скачать больше Go пакетов в контрольную группу).

Анализировать можно по-разному, мне было достаточно "`git log --grep`" по набору шаблонов, с завышением и занижением веса (score) коммита в зависимости от содержимого его commit message. Если коммит слишком огромный, его сразу можно отбросить, потому что с большой вероятностью разобраться что там происходит будет нетривиально.

Хочется ещё добавить, что рядом с исправлениями или нахождением ошибок довольно часто встречаются не очень оптимистичные и иногда не вполне культурные описания патчей:

```
- "We check length later, but we assumed it was always 1 bytes long. Not always the case. I'm a little depressed that this bug was there"

- "Really fixed the bug now"

- "WTF, the changes that actually fixed the bug for the accesspattern wasn't actually committed"

- "how do i pronounce this damn project" (проект traefic)

- "damnit travis"

- "one f*cking dot" (./.. => ./... в .travis.yml)
```

Самые бесполезные "исправления ошибок" - это правка синтаксических ошибок или прочих проблем, которые не дают проекту даже собраться через `go build`. Не во всех проектах используется [CI](https://en.wikipedia.org/wiki/Continuous_integration) или [pre-commit hook](https://xakep.ru/2016/02/11/git-hook-magic/), поэтому такие сломанные ревизии иногда попадают в master ветку.

# Исторические ошибки

Пройдёмся по некоторым наиболее интересным ошибкам, которые были найдены под одеялом git логов.

## Возврат nil вместо реального результата

[gin-gonic/gin: fix bug, return err when failed binding bool](https://github.com/gin-gonic/gin/commit/5636afe02d00308bc5ade9dd80acfa7646b608df):

```diff
	if err == nil {
		field.SetBool(boolVal)
	}
-	return nil
+	return err
}
```

[go-pg/pg: AppendParam bug fix](https://github.com/go-pg/pg/commit/c10d76d6a679e5f4b61c17ebdaf710ffa5921e8f):

```diff
		return method.AppendValue(b, m.strct.Addr(), 1), true
	}
-	return nil, false
+	return b, false
}
```

## Отсутствие return в fast path

[gonum/gonum: fixed bug in dswap fast path](https://github.com/gonum/gonum/commit/f142409d829436e33dc8ec4463928537e4e627a4):

```diff
		for i := 0; i < n; i++ {
			x[i], y[i] = y[i], x[i]
		}
+		return
	}
```

## Запуск горутины без wg.Add(1)

[btcsuite/btcd: Fix several bugs in the RPC server shutdown](https://github.com/btcsuite/btcd/commit/bd98836a2b7525c93e7b3b60d9732e13c28e36eb):

```diff
+s.wg.Add(1)
go s.walletListenerDuplicator()
```

## Некорректное логическое условие

[gorgonia/gorgonia: fixed a few bugs as well](https://github.com/gorgonia/gorgonia/commit/eb7da5862b1ba302bbb54d0ac580a752c59a308d):

```diff
-for i := 0; i > start; i++ {
+for i := 0; i < start; i++ {
	retVal[i] = 0
}
```

[src-d/go-git: plumbing/idxfile: fix bug searching in MemoryIndex](https://github.com/src-d/go-git/commit/7418b411660aaa3d8d54eb602fda8accaed2833f):

```diff
-if low < high {
+if low > high {
	break
}
```

[go-xorm/xorm: fix bug on sum](https://github.com/go-xorm/xorm/commit/6b6721cccc0015ea5275fdb3ae233872e1f654df):

```diff
-if !strings.Contains(colName, " ") && strings.Contains(colName, "(") {
+if !strings.Contains(colName, " ") && !strings.Contains(colName, "(") {
```

[btcsuite/btcd: server: Fix bug disconnecting peer on filteradd](https://github.com/btcsuite/btcd/commit/0734b553631c3e59d438c6acc1a8d00b22d385fd):

```diff
-if sp.filter.IsLoaded() {
+if !sp.filter.IsLoaded() {
```

[btcsuite/btcd: Fix reversed test bug with the max operations handling](https://github.com/btcsuite/btcd/commit/bac455cdd2326897b3e56539081cdd19212504e9):

```diff
-if pop.opcode.value < OP_16 {
+if pop.opcode.value > OP_16 {
	s.numOps++
```

## Обновление receiver'а (this/self), передаваемого по значению

Классика. Внутри метода модифицируется копия объекта, из-за чего вызывающая сторона не увидит ожидаемых результатов.

[Fixed a stack/queue/deque bug (pointer receiver)](https://github.com/gonum/gonum/commit/a342f2e9ed5637e064c03741ed13ea5e1f7015ff):

```diff
-func (s Stack) Push(x interface{}) {
-	s = append(s, x)
+func (s *Stack) Push(x interface{}) {
+	*s = append(*s, x)
}
```

## Обновление копий объектов внутри цикла

[containous/traefik: Assign filtered tasks to apps contained in slice](https://github.com/containous/traefik/commit/3174fb88613c1b7841f89995e5cf3ee2a85b3011):

```diff
-for _, app := range filteredApps {
-	app.Tasks = fun.Filter(func(task *marathon.Task) bool {
+for i, app := range filteredApps {
+	filteredApps[i].Tasks = fun.Filter(func(task *marathon.Task) bool {
		return p.taskFilter(*task, app)
	}, app.Tasks).([]*marathon.Task)
```

[containous/traefik: Add unit tests for package safe](https://github.com/containous/traefik/commit/2f06f339eca65568a4bfe50262090cef953e8064):

```diff
-for _, routine := range p.routines {
+for i := range p.routines {
	p.waitGroup.Add(1)
-	routine.stop = make(chan bool, 1)
+	p.routines[i].stop = make(chan bool, 1)
	Go(func() {
-		routine.goroutine(routine.stop)
+		p.routines[i].goroutine(p.routines[i].stop)
		p.waitGroup.Done()
	})
}
```

## Результат оборачивающей функции не используется

[gorgonia/gorgonia: Fixed a tiny nonconsequential bug in Grad()](https://github.com/gorgonia/gorgonia/commit/7b264e55887f4c1dc0871a5e8db8aca7bb32441a):

```diff
if !n.isInput() {
-	errors.Wrapf(err, ...)
-	// return
+	err = errors.Wrapf(err, ...)
+	return nil, err
}
```

## Неправильная работа с make

Иногда молодые гоферы делают "`make([]T, count)`" с последующим `append` внутри цикла. Правильным вариантом здесь является "`make([]T, 0, count)`".

[HouzuoGuo/tiedot: fix a collection scan bug](https://github.com/HouzuoGuo/tiedot/commit/3af35e191cd3a944ceb0a1abdc47923d11111144):

```diff
-ids := make([]uint64, benchSize)
+ids := make([]uint64, 0)
```

Исправление выше правит исходную ошибку, но имеет один изъян, который поправили в одном из следующих коммитов:

```diff
-ids := make([]uint64, 0)
+ids := make([]uint64, 0, benchSize)
```

Но некоторые операции, например `copy` или `ScanSlice` могут требовать "правильной" длины.

[go-xorm/xorm: bug fix for SumsInt return empty slice](https://github.com/go-xorm/xorm/commit/2cadda5fe5e472658c47d23d2e3c07672b873284):

```diff
-var res = make([]int64, 0, len(columnNames))
+var res = make([]int64, len(columnNames), len(columnNames))
if session.IsAutoCommit {
	err = session.DB().QueryRow(sqlStr, args...).ScanSlice(&res)
} else {
```

Скоро [go-critic](https://github.com/go-critic/go-critic) будет помогать в нахождении ошибок этого класса.

## Остальные ошибки

Перечисление получается довольно громоздким, оставшуюся часть уношу под спойлер. Эта часть опциональна, так что можете оставить её на потом или смело двигаться дальше.

<spoiler title="Хочу ещё!"><hr>

Следующий фрагмент интересен тем, что несмотря на то, что он правит ошибку, итерируемый ключ назван тем же именем, которое использовалось для итерируемого значения. Новый код не выглядит корректным и может запутать следующего программиста.

[gonum/gonum: Fixed bug in subset functions](https://github.com/gonum/gonum/commit/62ea9c8bfdc284d53170a37303e9f0c0e359673f):

```diff
-for _, el := range *s1 {
+for el := range *s1 {
	if _, ok := (*s2)[el]; !ok {
		return false
	}
```

Именованные результаты - практически такие же обычные переменные, поэтому инициализируются они по тем же правилам. Указатели будут иметь значение `nil` и если требуется работать с этим объектом, нужно явно инициализировать этот указатель (например, через `new`).

[dgraph-io/dgraph: fixing nil pointer reference bug in server/main.go](https://github.com/dgraph-io/dgraph/commit/33f936dd18cee4b2359e2966d6f8c1acf34cd8c7):

```diff
func (s *server) Query(ctx context.Context,
-	req *graph.Request) (resp *graph.Response, err error) {
+	req *graph.Request) (*graph.Response, error) {
+	resp := new(graph.Response)
```

Чаще и сильнее всего [shadowing](https://www.google.com/search?q=golang+shadowing) кусается при работе с ошибками.

[btcsuite/btcd: blockchain/indexers: fix bug in indexer re-org catch up](https://github.com/btcsuite/btcd/commit/be191ca111b494bb1ccdbeb7295723ee56949d07):

```diff
-block, err := btcutil.NewBlockFromBytes(blockBytes)
+block, err = btcutil.NewBlockFromBytes(blockBytes)
if err != nil {
	return err
}
```

[go-xorm/xorm: fix insert err bug](https://github.com/go-xorm/xorm/commit/a9eb28a00e4b93817906eac5c8af2a566e8c73af):

```diff
-err = session.Rollback()
+err1 := session.Rollback()
+if err1 == nil {
+	return lastId, err
+}
+err = err1
```

[gin-gonic/gin: Fixed newline problem with debugPrint in Run* functions](https://github.com/gin-gonic/gin/commit/44f024a413c9afc57970740a5fe3b6b6597bfe75):

```diff
-debugPrint("Listening and serving HTTP on %s", addr)
+debugPrint("Listening and serving HTTP on %s\n", addr)
```

Внедрение мьютекса в примере ниже навело меня на мысль целесообразности написания проверки, которая докладывает об использованиях `*sync.Mutex` в качестве члена структуры. Dave Cheney рекомендует [всегда использовать значение мьютекса](https://twitter.com/davecheney/status/743289114980548612), а не указатель на него.

[tealeg/xlsx: fix bug when loading spreadsheet](https://github.com/tealeg/xlsx/commit/dacfeea7a608cbaa21da22bd608441c207b53d70):

```diff
	styleCache map[int]*Style `-`
+	lock       *sync.RWMutex
}
```

<hr></spoiler>

# Крестовый поход против подозрительного кода

## badCond

Проверка `badCond` находит потенциально некорректные логические выражения.

Примеры подозрительных логических выражений:
* Результат всегда `true` или `false`
* Выражение написано в той форме, которая практически неотличима от ошибочной

Эта проверка также срабатывает на операциях вроде "`x == nil && x == y`". Одним из простых решений является переписывание в "`x == nil && y == nil`". Читабельность кода не страдает, зато линтер не будет видеть в этом коде что-то подозрительное.

Вот пример исправления бага, которая может быть найдена `badCond`:

```diff
-if err1 := f(); err != nil && err == nil {
+if err1 := f(); err1 != nil && err == nil {
	err = err1
}
```

**Трофеи:**
* [moby/buildkit: snapshot: fix impossible condition](https://github.com/moby/buildkit/pull/747)
* [tdewolff/parse: EqualFold from util.go has weird condition](https://github.com/tdewolff/parse/issues/46)
* [HouzuoGuo/tiedot: httpapi: suspicious condition in srv_test.go](https://github.com/HouzuoGuo/tiedot/issues/164)

## weakCond

`weakCond` похож на `badCond`, но уровень достоверности предупреждений несколько ниже.

Слабое условие - это такое условие, которое не покрывает входные данные достаточно полно.

Хороший пример слабого (недостаточного) условия - это проверка слайса на `nil` с последующим использованием первого элемента. Условия "`s != nil`" здесь недостаточно. Корректным условием будет "`len(s) != 0`" или её эквивалент:

```diff
func addRequiredHeadersToRedirectedRequests(
		req *http.Request, 
		via []*http.Request) error {
-	if via != nil && via[0] != nil {
+	if len(via) != 0 && via[0] != nil {
```

**Трофеи:**
* [etcd-io/etcd: etcdserver/api/v2v3: indexing after nil check without len check](https://github.com/etcd-io/etcd/issues/10315)
* [moby/moby: registry: indexing after nil check without len check](https://github.com/moby/moby/issues/38347)
* [inkyblackness/hacked: ss1/content/text: indexing after nil check without len check](https://github.com/inkyblackness/hacked/issues/52)

## dupArg

Для некоторых функций передача одного и того же значения (или переменной) в качестве нескольких аргументов не имеет большого смысла. Довольно часто это сигнализирует о [copy/paste](https://www.viva64.com/en/t/0068/) ошибке.

Примером такой функции является `copy`. Выражение `copy(xs, xs)` не имеет смысла.

```diff
-if !bytes.Equal(i2.Value, i2.Value) {
+if !bytes.Equal(i1.Value, i2.Value) {
	return fmt.Errorf("tries differ at key %x", i1.Key)
}
```

**Трофеи:**
* [ethereum/go-ethereum: light: fix duplicated argument in bytes.Equal call](https://github.com/ethereum/go-ethereum/pull/18269)
* [openshift/origin: fix reflect.DeepEqual(x, x) bug](https://github.com/openshift/origin/pull/21640)

## dupCase

Наверное вы знаете, что в Go нельзя использовать повторяющиеся `case` значения внутри `switch`.

Пример ниже [не компилируется](https://play.golang.org/p/imRt9SsnZ7P):

```go
switch x := 0; x {
case 1:
	fmt.Println("first")
case 1:
	fmt.Println("second")
}
```

> Ошибка компиляции: duplicate case 1 in switch.

Но что если взять [пример поинтереснее](https://play.golang.org/p/8hUHft5Ac8-):

```go
one := 1
switch x := 1; x {
case one:
	fmt.Println("first")
case one:
	fmt.Println("second")
}
```

Через переменную использовать дублирующиеся значения разрешены. Помимо этого, `switch true` даёт ещё больше простора для ошибок:

```go
switch x := 0; true {
case x == 1:
	fmt.Println("first")
case x == 1:
	fmt.Println("second")
}
```

А вот и пример исправления реальной ошибки:

```diff
switch {
case m < n: // Upper triangular matrix.
	// Вырезано для краткости.
case m == n: // Diagonal matrix.
	// Вырезано для краткости.
-case m < n: // Lower triangular matrix.
+case m > n: // Lower triangular matrix.
	// Вырезано для краткости.
}
```

**Трофеи:**
* [mellium/sasl: remove duplicated switch case](https://github.com/mellium/sasl/pull/2)
* [gonum/gonum: fix duplicated switch case](https://github.com/gonum/gonum/pull/758)
* [pact-foundation/pact-go: remove duplicated switch case](https://github.com/pact-foundation/pact-go/pull/104)
* [minio/minio: suspicious duplicated switch case clause in cmd/object-api-error.go](https://github.com/minio/minio/issues/6948)
* [henrylee2cn/pholcus: common/bytes: remove duplicated switch case](https://github.com/henrylee2cn/pholcus/pull/101)
* [go-ble/ble: linux/att: fix duplicated switch case error](https://github.com/go-ble/ble/pull/39)
* [docker/distribution: registry/storage: fix duplicated switch case](https://github.com/docker/distribution/pull/2785)

## dupBranchBody

В условных операторах, таких как `if/else` и `switch` есть тела для исполнения. Когда условие выполнено, управление передаётся ассоциированному блоку операций. В то время как другие диагностики проверяют что условия этих блоков различны, `dupBranchBody` проверяет что сами исполняемые блоки не являются полностью идентичными.

Наличие дублирующихся тел внутри условных операторов, если только это не `type switch`, как минимум подозрительно.

```diff
-if s, err := r.ReceivePack(context.Background(), req); err != nil { 
-	return s, err 
-} else { 
-	return s, err 
-}
+return r.ReceivePack(context.Background(), req)
```

Суждения о степени корректности кода ниже оставляю на откуп читателям:

```go
if field.IsMessage() || p.IsGroup(field) {
		// Вырезано для краткости.
	} else if field.IsString() {
		if nullable && !proto3 {
			p.generateNullableField(fieldname, verbose)
		} else {
			p.P(`if this.`, fieldname, ` != that1.`, fieldname, `{`)
		}
	} else {
		if nullable && !proto3 {
			p.generateNullableField(fieldname, verbose)
		} else {
			p.P(`if this.`, fieldname, ` != that1.`, fieldname, `{`)
		}
	}
}
```

Тела внутри "`if field.IsString()`" и связанного с ним "`else`" идентичны.

**Трофеи:**
* [src-d/go-git: Suspicious duplicated branch bodies](https://github.com/src-d/go-git/issues/1035)

## caseOrder

Все типы внутри `type switch` перебираются последовательно, до первого совместимого. Если tag-значение имеет тип `*T`, и реализует интерфейс `I`, то ставить метку с "`case *T`" нужно *до* метки с "`case I`", иначе выполняться будет только "`case I`", так как `*T` совместим с `I` (но не наоборот).

```diff
case string:
	res = append(res, NewTargetExpr(v).toExpr().(*expr))
-case Expr:
-	res = append(res, v.toExpr().(*expr))
case *expr:
	res = append(res, v)
+case Expr:
+	res = append(res, v.toExpr().(*expr))
```

**Трофеи:**
* [cmd/guru: fix incorrect case order in describe.go](https://go-review.googlesource.com/c/tools/+/153397)
* [message/pipeline: fix type switch case order](https://go-review.googlesource.com/c/text/+/153399)
* [coyove/goflyway: proxy: fix case order in a type switch](https://github.com/coyove/goflyway/pull/129)
* [go-graphite/carbonapi: pkg/parser: fix type switch case order](https://github.com/go-graphite/carbonapi/pull/381)

## offBy1

Ошибка на единицу так популярна, что у неё даже есть [своя страница на википедии](https://en.wikipedia.org/wiki/Off-by-one_error).

```diff
if len(optArgs) > 1 {
	return nil, ErrTooManyOptArgs
}
-node = optArgs[1]
+node = optArgs[0]
```

```diff
-last := xs[len(xs)]
+last := xs[len(xs) - 1]
```

В `go-critic` есть ограниченные возможности нахождения таких ошибок, но пока ни одного исправления в публичный репозиторий отправлено не было. Вот некоторые исправления, которые были найдены при просмотре истории:

* [btcsuite/btcd: test and fix bugs in getaddednodeinfo](https://github.com/btcsuite/btcd/commit/b3b8b37855011495127b1949e78f1ea315e3b580)
* [btcsuite/btcd: Improve test coverage and fix some bugs found by tests](https://github.com/btcsuite/btcd/commit/e1dd773e7c8d964ced5bf8b6acc64adacab4b845)

# Немного ужасов напоследок

В `go vet` есть хорошая проверка для выражений типа "`x != a || x != b`". Казалось бы, люди могут не знать про `gometalinter`, но `go vet` запускает почти каждый, да?

С помощью утилиты [gogrep](https://github.com/mvdan/gogrep) я собрал небольшой список похожих выражений, которые до сих пор находятся в master ветках некоторых проектов.

<spoiler title="Для самых храбрых котиков"><hr>

Предлагаю рассмотреть этот список и отправить pull request'ы.

```
cloud.google.com/go/trace/trace_test.go:943:7:
  expectTraceOption != ((o2&1) != 0) || expectTraceOption != ((o3&1) != 0)
github.com/Shopify/sarama/mocks/sync_producer_test.go:36:5:
  offset != 1 || offset != msg.Offset
github.com/Shopify/sarama/mocks/sync_producer_test.go:44:5:
  offset != 2 || offset != msg.Offset
github.com/docker/libnetwork/api/api_test.go:376:5:
  id1 != i2e(i2).ID || id1 != i2e(i3).ID
github.com/docker/libnetwork/api/api_test.go:408:5:
  "sh" != epList[0].Network || "sh" != epList[1].Network
github.com/docker/libnetwork/api/api_test.go:1196:5:
  ep0.ID() != ep1.ID() || ep0.ID() != ep2.ID()
github.com/docker/libnetwork/api/api_test.go:1467:5:
  ep0.ID() != ep1.ID() || ep0.ID() != ep2.ID()
github.com/docker/libnetwork/ipam/allocator_test.go:1261:5:
  len(indices) != len(allocated) || len(indices) != num
github.com/esimov/caire/grayscale_test.go:27:7:
  r != g || r != b
github.com/gogo/protobuf/test/bug_test.go:99:5:
  protoSize != mSize || protoSize != lenData
github.com/gogo/protobuf/test/combos/both/bug_test.go:99:5:
  protoSize != mSize || protoSize != lenData
github.com/gogo/protobuf/test/combos/marshaler/bug_test.go:99:5:
  protoSize != mSize || protoSize != lenData
github.com/gogo/protobuf/test/combos/unmarshaler/bug_test.go:99:5:
  protoSize != mSize || protoSize != lenData
github.com/gonum/floats/floats.go:65:5:
  len(dst) != len(s) || len(dst) != len(y)
github.com/gonum/lapack/testlapack/dhseqr.go:139:7:
  wr[i] != h.Data[i*h.Stride+i] || wr[i] != h.Data[(i+1)*h.Stride+i+1]
github.com/gonum/stat/stat.go:1053:5:
  len(x) != len(labels) || len(x) != len(weights)
github.com/hashicorp/go-sockaddr/ipv4addr_test.go:659:27:
  sockaddr.IPPort(p) != test.z16_portInt || sockaddr.IPPort(p) != test.z16_portInt
github.com/hashicorp/go-sockaddr/ipv6addr_test.go:430:27:
  sockaddr.IPPort(p) != test.z16_portInt || sockaddr.IPPort(p) != test.z16_portInt
github.com/nats-io/gnatsd/server/monitor_test.go:1863:6:
  v.ID != c.ID || v.ID != r.ID
github.com/nbutton23/zxcvbn-go/adjacency/adjcmartix.go:85:7:
  char != "" || char != " "
github.com/openshift/origin/pkg/oc/cli/admin/migrate/migrator_test.go:85:7:
  expectedInfos != writes || expectedInfos != saves
github.com/openshift/origin/test/integration/project_request_test.go:120:62:
  added != deleted || added != 1
github.com/openshift/origin/test/integration/project_request_test.go:126:64:
  added != deleted || added != 4
github.com/openshift/origin/test/integration/project_request_test.go:132:62:
  added != deleted || added != 1
gonum.org/v1/gonum/floats/floats.go:60:5:
  len(dst) != len(s) || len(dst) != len(y)
gonum.org/v1/gonum/lapack/testlapack/dhseqr.go:139:7:
  wr[i] != h.Data[i*h.Stride+i] || wr[i] != h.Data[(i+1)*h.Stride+i+1]
gonum.org/v1/gonum/stat/stat.go:1146:5:
  len(x) != len(labels) || len(x) != len(weights)
```

<hr></spoiler>

# Учимся на чужих ошибках

![](https://habrastorage.org/webt/ys/9n/qo/ys9nqo2yu_gvopttkccqvn3cd4c.png)

Нет, даже в последней части не будет ничего про [машинное обучение](http://lmgtfy.com/?q=%D0%B8%D0%B2%D0%B0%D0%BD+%D0%B8%D0%B2%D0%B0%D0%BD%D0%BE%D0%B2%D0%B8%D1%87).

Зато о чём здесь будет написано, так это о [golangci-lint](https://github.com/golangci/golangci-lint), который в одном из последних релизов интегрировал [go-critic](https://github.com/go-critic/go-critic). `golangci-lint` - это аналог [gometalinter](https://github.com/alecthomas/gometalinter), почитать про достоинства которого можно, например, в статье ["Статический анализ в Go: как мы экономим время при проверке кода"](https://habr.com/company/roistat/blog/413175).

Сам `go-critic` недавно был переписан с помощью [lintpack](https://github.com/go-lintpack/lintpack). За подробностями можно проследовать в ["Go lintpack: менеджер компонуемых линтеров"](https://habr.com/post/430196/).

Если вы ещё не начали активно использовать инструменты для анализа ваших программ на наличие потенциальных ошибок, как стилистических, так и логических, рекомендую сегодня ещё раз подумать о своём поведении за чашечкой чая с коллегами. У вас всё получится.

Всем добра.
