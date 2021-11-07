# ELFийские трюки в Go

![](https://habrastorage.org/webt/hd/mj/7h/hdmj7hkb2kyjkyyybccynazhd7s.png)

В этой заметке мы научимся получать машинный код Go функции прямо в рантайме, распечатаем его с помощью дизассемблера и по пути узнаем несколько фокусов вроде получения адреса функции без её вызова.

**Предупреждение**: ничему полезному эта мини-статья вас не научит.

<cut/>

# Function value в Go

Для начала определимся, что такое Go функция и зачем нам нужно понятие **function value**.

Лучше всего это объясняет документ [Go 1.1 Function Calls](https://golang.org/s/go11func). Документ не новый, но большая часть информации в нём всё ещё актуальна.

На самом низком уровне это всегда указатель на исполняемый код, но когда мы используем анонимные функции/замыкания или передаём функцию как `interface{}`, этот указатель скрывается внутри некоторой структуры.

Само по себе имя функции не является выражением, поэтому, такой код не работает:

```go
// https://play.golang.org/p/wXeVLU7nLPs
package main
func add1(x int) int { return 1 }
func main() {
	addr := &add1
	println(addr)
}
```

> `compile: cannot take the address of add1`

Но при этом мы можем получить `function value` через то же имя функции:

```go
// https://play.golang.org/p/oWqv_FQq4hy
package main
func add1(x int) int { return 1 }
func main() {
	f := add1 // <--------
	addr := &f
	println(addr)
}
```

Этот код запускается, но печатать он будет адрес локальной переменной на стеке, что не совсем то, чего мы хотели. Но, как было сказано выше, адрес функции всё ещё там, просто нужно знать, как к нему доступиться.

Пакет [`reflect`](https://golang.org/pkg/reflect/) зависит от этой детали реализации, чтобы успешно выполнять [`reflect.Value.Call()`](https://golang.org/pkg/reflect/#Value.Call). [Там же (reflect/makefunc.go)](https://github.com/golang/go/blob/dcd3b2c173b77d93be1c391e3b5f932e0779fb1f/src/reflect/makefunc.go#L56-L60) можно подсмотреть следующий шаг для получения адреса функции:

```go
dummy := makeFuncStub
code := **(**uintptr)(unsafe.Pointer(&dummy))
```

Код выше демонстрирует базовую идею, которую можно доработать до функции:

```go
// funcAddr returns function value fn executable code address.
func funcAddr(fn interface{}) uintptr {
	// emptyInterface is the header for an interface{} value.
	type emptyInterface struct {
		typ   uintptr
		value *uintptr
	}
	e := (*emptyInterface)(unsafe.Pointer(&fn))
	return *e.value
}
```

Получить адрес функции `add1` можно с помощью вызова `funcAddr(add1)`.

# Получение блока машинного кода функции

Теперь, когда у нас есть адрес начала машинного кода функции, хотелось бы получить весь машинный код функции. Здесь требуется уметь определять, где же код текущей функции заканчивается.

Если бы архитектура x86 имела инструкции фиксированной длины, было бы не так сложно и нам могли бы помочь несколько эвристик, среди которых:

* Как правило, в конце кода функций есть отбивка из [`INT3`](https://en.wikipedia.org/wiki/INT_(x86_instruction)#INT3) инструкций. Это хороший маркер конца кода функции, но он может отсутствовать.
* У функций с ненулевым фреймом для стека есть пролог, который проверяет, нужно ли расширять этот стек. Если да, то выполняется прыжок на код сразу за кодом функции, а затем прыжок на старт функции. Интересующий нас код будет посередине.

Но вам потребуется честно декодировать инструкции, потому что побайтовый проход может найти байт `INT3` внутри другой инструкции. Сделать вычисление длины инструкции для её пропуска тоже не так легко, [потому что это x86, детка](https://stackoverflow.com/questions/45801447/x86-assembly-how-to-calculate-instruction-opcodes-length-in-bytes).

Адрес функции в контексте пакета [`runtime`](https://golang.org/pkg/runtime) иногда называют `PC`, чтобы подчеркнуть возможность использовать адрес где-то внутри функции, а не только точку входа функции. Результат `funcAddr` можно использовать как аргумент функции [`runtime.FuncForPC()`](https://golang.org/pkg/runtime/#FuncForPC), для получения [`runtime.Func`](https://golang.org/pkg/runtime/#Func) без вызова самой функции. Через небезопасные новогодние преобразования мы можем получить доступ к [`runtime._func`](https://play.golang.org/p/lQxxK36ZXru), что познавательно, но не очень полезно: там нет информации о размере блока кода функции.

Похоже, без помощи [ELFов](https://golang.org/pkg/debug/elf/) мы не справимся.

> Для платформ, где исполняемые файлы имеют иной формат, большая часть статьи останется релевантна, но вам нужно будет использовать не [`debug/elf`](https://golang.org/pkg/debug/elf/), а другой пакет из [`debug`](https://golang.org/pkg/debug/).

# ELF, который прячется в вашей программе

Информация, которая нам нужна, уже содержится в метаданных [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#File_header) файла.

Через [`os.Args[0]`](https://golang.org/pkg/os/#pkg-variables) мы можем получить доступ к самому исполняемому файлу, а уже из него получить таблицу символов.

```go
func readELF() (*elf.File, error) {
	f, err := os.Open(os.Args[0])
	if err != nil {
		return nil, fmt.Errorf("open argv[0]: %w", err)
	}
	return elf.NewFile(f)
}
```

# Поиск символа внутри [`elf.File`](https://golang.org/pkg/debug/elf/#File)

Получить все символы можно с помощью метода [`File.Symbols()`](https://golang.org/pkg/debug/elf/#File.Symbols). Этот метод возвращает [`[]elf.Symbol`](https://golang.org/pkg/debug/elf/#Symbol), которые содержат поле `Symbol.Size` - это и есть искомый нами "размер функции". Поле `Symbol.Value` должно совпадать со значением, возвращаемым `funcAddr`.

Искать нужный символ можно либо по адресу (`Symbol.Value`), либо по имени (`Symbol.Name`). Если бы символы были сортированы по имени, можно было бы применить [`sort.Search()`](https://golang.org/pkg/sort/#Search), но это не так:

> The symbols will be listed in the order they appear in file.

Если нужно часто находить символы в таблице, то следует построить дополнительный индекс, например, через `map[string]*elf.Symbol` или `map[uintptr]*elf.Symbol`.

Поскольку мы уже умеем получать адрес функции по её значению, будем делать поиск по нему:

```go
func elfLookup(f *elf.File, value uint64) *elf.Symbol {
	symbols, err := f.Symbols()
	if err != nil {
		return nil
	}
	for _, sym := range symbols {
		if sym.Value == value {
			return &sym
		}
	}
	return nil
}
```

> **Примечание**: для работы этого подхода, нам нужна таблица символов. Если бинарник собран с '`-ldflags "-s"`', то `elfLookup()` всегда будет возвращать `nil`. Если вы будете запускать программу через `go run` вы можете столкнуться с той же проблемой. Для примеров из статьи рекомендуется делать '`go build`' или '`go install`' для получения исполняемых файлов.

# Получение машинного кода функции

Зная диапазон адресов, в котором находится исполняемый код, остаётся только вытащить его в виде `[]byte` для удобной обработки.

```go
func funcCode(addr uintptr) ([]byte, error) {
	elffile, err := readELF()
	if err != nil {
		return nil, fmt.Errorf("read elf: %w", err)
	}
	sym := elfLookup(elffile, uint64(addr))
	if sym == nil {
		return nil, fmt.Errorf("can't lookup symbol for %x", addr)
	}
	code := *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: addr,
		Len:  int(sym.Size),
		Cap:  int(sym.Size),
	}))
	return code, nil
}
```

Этот код намеренно упрощен для демонстрации. Не стоит каждый раз читать `ELF` и выполнять линейный поиск по его таблице.

Результатом работы функции `funcCode()` является слайс с байтами машинного кода функции. На вход ей стоит подавать результат вызова `funcAddr()`.

```go
code, err := funcCode(funcAddr(add1))
if err != nil {
	log.Panicf("can't get function code: %v", err)
}
fmt.Printf("% x\n", code)
// => 48 8b 44 24 08 48 ff c0 48 89 44 24 10 c3
```

# Дизассемблирование машинного кода

Для того, чтобы проще читать машинный код, мы воспользуемся дизассемблером.

Я больше всего знаком с проектами [zydis](https://github.com/zyantific/zydis) и [Intel XED](https://github.com/intelxed/xed), поэтому в первую очередь мой выбор падает на них.

Для Go можно взять биндинг [go-zydis](https://github.com/jpap/go-zydis), который достаточно хорош и прост в установке для нашей задачи.

Опишем абстракцию "обхода машинных инструкций", с помощью которой потом можно реализовать остальные операции:

```go
func walkDisasm(code []byte, visit func(*zydis.DecodedInstruction) error) error {
	dec := zydis.NewDecoder(zydis.MachineMode64, zydis.AddressWidth64)

	buf := code
	for len(buf) > 0 {
		instr, err := dec.Decode(buf)
		if err != nil {
			return err
		}
		if err := visit(instr); err != nil {
			return err
		}
		buf = buf[int(instr.Length):]
	}

	return nil
}
```

Эта функция принимает на вход слайс машинного кода и вызывает callback-функцию для каждой декодируемой инструкции.

На основе неё мы можем написать нужную нам `printDisasm`:

```go
func printDisasm(code []byte) error {
	const ZYDIS_RUNTIME_ADDRESS_NONE = math.MaxUint64
	formatter, err := zydis.NewFormatter(zydis.FormatterStyleIntel)
	if err != nil {
		return err
	}
	return walkDisasm(code, func(instr *zydis.DecodedInstruction) error {
		s, err := formatter.FormatInstruction(instr, ZYDIS_RUNTIME_ADDRESS_NONE)
		if err != nil {
			return err
		}
		fmt.Println(s)
		return nil
	})
}
```

Если мы запустим `printDisasm` на коде функции `add1`, то получим долгожданный результат:

```asm
mov rax, [rsp+0x08]
inc rax
mov [rsp+0x10], rax
ret
```

## Валидация результата

Теперь мы попробуем убедиться в том, что ассемблерный код, полученный в предыдущей секции, является корректным.

Поскольку у нас уже есть собранный бинарник, можно использовать поставляемый с Go `objdump`:

```bash
$ go tool objdump -s 'add1' exe
TEXT main.add1(SB) example.go
  example.go:15    0x4bb760    488b442408    MOVQ 0x8(SP), AX
  example.go:15    0x4bb765    48ffc0        INCQ AX
  example.go:15    0x4bb768    4889442410    MOVQ AX, 0x10(SP)
  example.go:15    0x4bb76d    c3            RET
```

Всё сходится, только синтаксис немного разный, что ожидаемо.

# Method expressions

Если нам нужно совершить аналогичное с методами, то вместо имени функции мы будем использовать [method expression](https://golang.org/ref/spec#Method_expressions).

Допустим, наш `add1` - это на самом деле не функция, а метод типа `adder`:

```go
type adder struct{}

func (adder) add1(x int) int { return x + 2 }
```

Тогда вызов получения адреса функции будет выглядеть как `funcAddr(adder.add1)`.

# Заключение

Пришёл я к этим вещам не случайно и, возможно, в одной из следующих статей расскажу, как планировалось использовать все эти механизмы. А пока предлагаю относиться к этой заметке как к поверхностному описанию того, как `runtime` и `reflect` смотрят на наши Go функции через function value.

Список использованных ресурсов:

* [Go 1.1 Function Calls](https://golang.org/s/go11func)
* [Analyzing Golang Executables](https://www.pnfsoftware.com/blog/analyzing-golang-executables/)
* [Go "internal ABI" design](https://go.googlesource.com/proposal/+/master/design/27539-internal-abi.md)
