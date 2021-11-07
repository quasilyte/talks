# Парсеры Пратта для чайников

[Рекурсивный спуск](https://en.wikipedia.org/wiki/Recursive_descent) работает идеально, когда вы можете принимать решение относительно разбираемого куска кода с помощью текущего контекста и токена.

Картину портят [выражения](https://en.wikipedia.org/wiki/Expression_(computer_science)): постфиксные, инфиксные и прочие. Проблема: вы не можете понять, какого типа выражение вы обрабатываете до тех пор, пока не разберёте его первую половину. Зачастую для вас также важны [приоритет операции](https://en.wikipedia.org/wiki/Order_of_operations) и её [ассоциативность](https://en.wikipedia.org/wiki/Operator_associativity), чтобы построенное [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) имело правильную структуру.

В этой статье мы напишем парсер для диалекта Go, особенности которого мы рассмотрим чуть ниже. Как вы сможете убедиться, алгоритм Пратта решает большинство наших проблем.

![](https://habrastorage.org/webt/gi/eg/ji/giegjidcnqpsbkpiwlbrj9evce4.png)

<cut/>

<spoiler title="Терминология">
<hr>

Определения ниже даны в качестве краткой сводки. Другими словами, это не точные определения, а мои попытки описать какой-то термин в одно предложение.

**Оператор**: конструкция языка, производящая операцию (например, сложение).
**Операнд**: практически то же самое, что и аргумент.
**Выражение/expression**: вычисление, производящее значение. Пример: вызов функции.
**Statement**: конструкция языка, не производящяя значение. Пример: цикл.
**Приоритет операции/precedence**: очерёдность выполнения операций в выражении.
**Постфиксный оператор**: оператор, который следует за своим аргументом. Пример: `x++`.
**Префиксный оператор**: оператор, которые предшествует своему аргументу. Пример: `++x`.
**Инфиксный оператор**: оператор, который находится между аргументами. Пример: `x+y`.
**Левая ассоциативность/left-associative**: вычисление слева направо.
**Правая ассоциативность/right-associative**: вычисление справа налево.
**non-associative**: вычисление нельзя продолжить ни в одном направлении.
**Парсер**: компонент, превращающий поток токенов в AST или другое представление.
**AST**: программа в форме дерева, где элементами являются операторами и операндами.
**Лексер/токенизатор**: компонент, который превращает исходный текст в поток токенов.
**Токен**: классифицированный элемент исходного текста.
**Тег токена**: вольный термин для обозначения категории токена.
**Грамматика**: описание алфавита языка (синтаксис).
**Унарная операция**: операция, имеющая ровно один аргумент.
**Бинарная операция**: операция, имеющая ровно два аргумента.

<hr>
</spoiler>

## Go++ и ++Go

Чтобы не тратить время на [лексический анализ](https://en.wikipedia.org/wiki/Lexical_analysis), мы возьмём всё нужное из стандартной библиотеки:

* [go/scanner](https://golang.org/pkg/go/scanner/) позволяет получать из текста набор токенов
* [go/token](https://golang.org/pkg/go/token/) определяет те самые токены, которые нам даёт `scanner`

Для удобства, мы введём свой тип токена, чтобы не разделять теги и значения:

```go
type Token struct {
	kind  token.Token // Категория (тег) токена, INT/IDENT/ADD
	value string      // Текстовое значение токена
}
```

[scanner.Scanner](https://golang.org/pkg/go/scanner/#Scanner) мы обернём в свой тип `lexer`, в котором будет два основных метода:

* `Peek()` - возвращает следующий токен, но не извлекает его из потока
* `Consume()` - возвращает следующий токен, извлекая его

Мы не будем использовать пакет [go/ast](https://golang.org/pkg/go/ast/), потому что это уже напрямую относится к парсингу, который нам в рамках этой статьи интересен.

Поскольку в Go нет right-associative операторов, в своём диалекте мы определим оператор побитового сдвига влево как таковой.

В настоящем Go инкремент и декремент - это statement, а не expression. Но в нашем диалекте они будут выражениями. Более того, мы также добавим префиксные варианты этих операций.

> По ходу текста вам иногда будет не хватать некоторых определений типов и методов. Попробуйте не обращать на них внимания, весь код вы сможете посмотреть после того, как поймёте описываемый алгоритм.

## Начнём с простого

В этой статье мы будем изучать алгоритм на примерах. Если вам нужно более формальное описание алгоритма, рекомендую README этого репозитория: [github.com/richardjennings/prattparser](https://github.com/richardjennings/prattparser).

Первым делом попробуем успешно разобрать простейшие выражения, вроде переменных и префиксных операторов с их операндами.

```go
// exprNode описывает произвольное выражение в AST.
type exprNode interface {
	expr()
}

type nameExpr struct {
	Value string
}

type prefixExpr struct {
	Op  token.Token // Тип операции, например, '+' или '-'
	Arg exprNode    // Операнд унарной операции
}

func (e *nameExpr) expr()   {}
func (e *prefixExpr) expr() {}
```

Основной метод парсера выражений, `parseExpr()`, может выглядеть так:

```go
func (p *exprParser) parseExpr() exprNode {
	tok := p.lexer.Consume()
	switch tok.kind {
	case token.IDENT:
		return p.parseName(tok)
	case token.ADD, token.SUB:
		return p.parsePrefixExpr(tok)
	case token.LPAREN:
		return p.parseParenExpr(tok)
	// ... и так далее
	}
}

func (p *exprParser) parseName(tok Token) exprNode {
	return &nameExpr{Value: tok.value}
}

func (p *exprParser) parsePrefixExpr(tok Token) exprNode {
	arg := p.parseExpr()
	return &prefixExpr{Op: tok.kind, Arg: arg}
}
```

При такой структуре кода мы рискуем получить спагетти-код когда реализуем все нужные конструкции языка.

Если бы мы могли более декларативно описать отношения токенов и методов их обработки, код стал бы более наглядным. Сделать это можно, поместив методы-обработчики, типа `parsePrefixExpr()`, в `map` или прочую структуру, которая позволяет получить метод для обработки по токену. Для большей производительности стоит использовать массивы, поскольку часто разнообразие токенов не очень велико, но для простоты реализации мы возьмём `map`.

Поскольку у всех наших методов одинаковая сигнатура, заведём для них тип `prefixParselet`:

```go
type prefixParselet func(Token) exprNode
```

В сам парсер нужно добавить `map[token.Token]prefixParselet`. При создании парсера мы инициализируем эту таблицу:

```go
func newExprParser() *exprParser {
	p := &exprParser{
		prefixParselets: make(map[token.Token]prefixParselet),
	}

	prefixExpr := func(kinds ...token.Token) {
		for _, kind := range kinds {
			p.prefixParselets[kind] = p.parsePrefixExpr
		}
	}

	p.prefixParselets[token.IDENT] = p.parseName
	prefixExpr(token.ADD, token.SUB)

	return p
}
```

Мы ввели вспомогательную функцию `prefixExpr()` для упрощения добавления префиксных операторов. Можно пойти ещё дальше, тогда инициализирующий код будет ещё больше похож на грамматику (см. заключительные главы статьи).

```go
func (p *exprParser) parseExpr() exprNode {
  tok := p.lexer.consume()
  prefix, ok := p.prefixParselets[tok.kind]
  if !ok {
    // Parse error: unexpected token
  }
  return prefix(tok)
}
```

## Застряв посередине

Попробуем научить парсер работать с инфиксными выражениями, вроде `x+y`.

Текущая реализация вернёт нам `nameExpr{Value:"x"}`, остановившись на токене `+`. Этот токен как раз говорит нам о том, как парсить дальше.

Введём новый тип `infixParselet`, который отличается от `prefixParselet` тем, что принимает на вход `Expr`, идущий перед ним. В случае с `x+y` это будет `x`.

```go
type infixParselet func(left exprNode, tok Token) exprNode
```

Для всех инфиксных парслетов мы заведём отдельную `map`. Заполняется она аналогичным образом, в конструкторе парсера.

Бинарные операторы, типа `+` и `-` будем выражать через `binaryExpr`:

```go
func (p *exprParser) parseBinaryExpr(left exprNode, tok token) exprNode {
	right := p.parseExpr()
	return &binaryExpr{Op: tok.kind, Left: left, Right: right}
}
```

Добавим в метод `parseExpr()` использование инфиксных парслетов:

```go
func (p *exprParser) parseExpr() exprNode {
	tok := p.lexer.Consume()
	prefix, ok := p.prefixParselets[tok.kind]
	if !ok {
		// Parse error: unexpected token
	}
	left := prefix(tok)
	tok = p.lexer.Peek()
	infix, ok := p.infixParselets[tok.kind]
	if !ok {
		return left
	}
	p.lexer.Consume() // Делаем skip/discard для токена
	return infix(left, tok)
}
```

У этой реализации есть две проблемы:

1. Все выражения разбираются как право-ассоциативные: `x-y-z` => `x-(y-z)`
2. Все операции имеют одинаковый приоритет: `x*y+z` => `x*(y+z)`

## Приоритет и ассоциативность операций

Чтобы решить обе проблемы, нам нужно дать парсеру информацию о приоритетах операций.

Делать мы это будем через таблицы `{token.Token => precedence}`. Эти таблицы можно сделать глобальными и переиспользовать между парсерами, но нам проще разместить их прямо в конструкторе парсера, для локальности кода:

```go
p.prefixPrecedenceTab = map[token.Token]int{
	token.ADD: 4,
	token.SUB: 4,
}

p.infixPrecedenceTab = map[token.Token]int{
	token.ADD: 2,
	token.SUB: 2,
	token.MUL: 3,
	token.QUO: 3,
	// ... и так далее
}
```

Метод `parseExpr()` теперь будет принимать аргумент `precedence`:

```go
func (p *exprParser) parseExpr(precedence int) exprNode {
	tok := p.lexer.Consume()
	prefix, ok := p.prefixParselets[tok.kind]
	if !ok {
		// Parse error: unexpected token
	}
	left := prefix(tok)

	for precedence < p.infixPrecedenceTab[p.lexer.Peek().kind] {
		tok := p.lexer.Consume()
		infix := p.infixParselets[tok.kind]
		left = infix(left, tok)
	}

	return left
}
```

С помощью нового аргумента парсер знает, когда продолжать, а когда остановиться, что позволяет правильно связать итоговое AST.

Каждый парслет теперь должен передать свой `precedence` (приоритет) при вызове `parseExpr()`:

```go
func (p *exprParser) parseBinaryExpr(left exprNode, tok Token) exprNode {
	right := p.parseExpr(p.infixPrecedenceTab[tok.kind])
	return &binaryExpr{Op: tok.kind, Left: left, Right: right}
}
```

`parseBinaryExpr()` связывает выражения как лево-ассоциативные. Чтобы разобрать право-ассоциативные выражения, нужно вычесть 1 из приоритета операции:

```go
func (p *exprParser) rparseBinaryExpr(left exprNode, tok Token) exprNode {
	right := p.parseExpr(p.infixPrecedenceTab[tok.kind] - 1)
	return &binaryExpr{Op: tok.kind, Left: left, Right: right}
}
```

В начале статьи мы уточнили, что `<<` у нас будет право-ассоциативный, поэтому обрабатываться побитовый сдвиг будет с помощью `rparseBinaryExpr()`.

## Добавляем остальные операторы

Вызов функций - это `infixParselet`, который обрабатывает токен `'('` и собирает все аргументы через `parseExpr(0)` до тех пор, пока не найдёт `')'`.

Группирующий оператор (скобочки) - это `prefixParselet`, который обрабатывает токен `'('` и единственный аргумент через `parseExpr(0)`, а затем ожидает `')'`.

```go
func (p *exprParser) parseParenExpr(tok Token) exprNode {
	x := p.parseExpr(0)
	p.expect(token.RPAREN)
	return x
}
```

Постфиксные операции - это самый простой вид `infixParselet`:

```go
func (p *exprParser) parsePostfixExpr(left exprNode, tok Token) exprNode {
	return &postfixExpr{Op: tok.kind, Arg: left}
}
```

## Финальные штрихи

Если вы разобрались с описываемым паттерном написания парсеров, можно попробовать доработать код таким образом, чтобы он лучше удовлетворял вашей задаче.

Например, вместо того, чтобы отдельно заполнять таблицу приоритетов, мы можем добавить в каждую helper-функцию внутри конструктора аргумент `precedence`:

```go
prefixExpr := func(precedence int, kinds ...token.Token) {
	for _, kind := range kinds {
		p.prefixParselets[kind] = p.parsePrefixExpr
		p.prefixPrecedenceTab[kind] = precedence
	}
}
```

Это позволит нам сделать инициализацию парсера ещё более наглядной:

```go
addPrefixParselet(token.LPAREN, 0, p.parseParenExpr)
addPrefixParselet(token.IDENT, 0, p.parseNameExpr)
leftAssocBinaryExpr(1, token.LOR)  // ||
leftAssocBinaryExpr(2, token.LAND) // &&
leftAssocBinaryExpr(3,
	token.EQL, // ==
	token.NEQ, // !=
)
rightAssocBinaryExpr(4,
	token.SHL, // <<
)
leftAssocBinaryExpr(4,
	token.ADD, // +
	token.SUB, // -
	token.SHR, // >>
)
leftAssocBinaryExpr(5,
	token.MUL, // *
	token.QUO, // /
	token.REM, // %
)
prefixExpr(6,
	token.ADD, // +
	token.SUB, // -
	token.INC, // ++
	token.DEC, // --
)
postfixExpr(7,
	token.INC, // ++
	token.DEC, // --
)
addInfixParselet(token.LPAREN, 8, p.parseCallExpr)
```

Вместо магических констант можно ввести именованные группы, например, `PrecAdd=3`, `PrecMult=4` и так далее.

Для улучшения производительности стоит уйти от использования `map`.

Чтобы сделать создание парсера менее дорогой операцией, стоит инициализировать таблицы приоритетов и парслетов один раз, а потом передавать её в `newExprParser` входным аргументом. Вам придётся переделать сигнатуры `prefixParselet` и `infixParslet` так, чтобы они принимали один дополнительный аргумент - `*exprParser`.

Вместо отдельной таблицы под приоритеты, можно хранить и функцию, и приоритет в одном объекте. Вы также можете попробовать сделать оба парслета интерфейсами и завести отдельные типы под каждый вид парслета, но делать этого я не рекомендую: быстрее не станет, а заводить столько типов под простые операции не очень идиоматично - для этого у нас есть функции.

## Заключение

Изначально эта статья планировалась как перевод [Pratt Parsers: Expression Parsing Made Easy](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/) (Bob Nystrom). Но в ходе написания я в нескольких местах изменил структуру материала, переписал все примеры, расширил концовку и стало не очень очевидно, насколько это перевод, а не "материал, основанный на".

Я очень рекомендую почитать [оригинал](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/), если вы владеете английским.

Весь код можно найти в репозитории [github.com/quasilyte/pratt-parsers-go](https://github.com/quasilyte/pratt-parsers-go).

Альтернативная реализация: [github.com/richardjennings/prattparser](https://github.com/richardjennings/prattparser).

<hr>

Похожие статьи на хабре:
* [Нисходящий парсер с операторным предшествованием](https://habr.com/ru/post/227241/)
* [Как писать парсеры на JavaScript](https://habr.com/ru/post/224081/)
