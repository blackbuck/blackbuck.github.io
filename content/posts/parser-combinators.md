+++
title = "Parser Combinators"
description = "and the Chamber of Functional Programming"
date = 2025-01-13
author = "Anil Bishnoi"
+++

# The Call to Adventure

 **<mark style="background: #FFF3A3A6;">Skip this if you're here just for the parser generator part :)</mark>**
 
 If I had a list of disappointments for the year 2025, the first entry would be my college not offering the Compiler Design course in 6th semester. I mean, why shouldn't they? So what if they've clearly stated in the scheme that they'll be offering it in the 8th semester? Unable to digest the fact that I wasn't in 8th semester already, I decided to do something myself. And so, I started learning how to make my own compiler. To my surprise, it wasn't an easy task. Most "tutorials" on compiler design just sort of skip on the most basic phase: building a lexer and a parser. For the sake of "simplicity," they instead focus on more important tasks at hand. It was indeed disheartening. I mean, what's the point of learning something if I don't even know how it is supposed to work.

And so, I did. I got together a simple lexer and parser and made it to spit out a parse tree. But that was it. All that rage for nothing. A day or two later, I stumbled upon a Wikipedia article on parser generators (I don't know how I got there)

If I had a list for all the interesting things that happened to me in the year 2025, the first entry would be the discovery of parser combinators.

# The Fundamental Spells

According to Wikipedia,
> A parser combinator is a higher-order function that accepts several parsers as input and returns a new parser as its output.

Basically, a parser combinator is a function that takes one or more functions as parameters and returns a function combining these functions. In this context, these functions are supposed to be parsers. Parser combinators are a great tool for prototyping compilers for domain-specific tasks. More of it on later sections.

So, before moving to combining parsers, we need to define what a parser should be able to do. Mathematically, a parser is a function `f` that takes an `input` a recognizer `x` and an index `i` and produces an output such that:

$$
f(input, x, i) = \left\lbrace
	\begin{array}{ll}
		\brace  & if i \gt len(input) \\
		i + 1 & if input[i] = x \\
		\brace & otherwise
	\end{array}
\right.
$$

# The Dark Arts of Parsing

By functional programming standards, we should be able to return a [result type type](https://en.wikipedia.org/wiki/Result_type#:~:text=In%20functional%20programming%2C%20a%20result,value%20or%20an%20error%20code.) from the parser function. The naive approach might be to define result as follows:

```go
type Result[T any] struct {
	parsedResult   T
	remString      string
}

type Parser[T any] interface {
	parse(string) (result[T], error)
}
```

This is what I thought as well. But the problem with defining `Result` and `Parser` with generics is that dealing with generics became a whole lot complex and was causing issues with the testing (handling [`DeepEqual`](https://pkg.go.dev/reflect#DeepEqual) was a headache on its own.) There was also a different problem: I wasn't considering the input as a state machine. Although it wasn't that big of a performance bottleneck (I haven't benchmarked it against the second version) but I just didn't like passing a string around too much. I now had two problems to deal with:
1. Redefine `Parser` and `Result`
2. Implement a state machine.

A `State` could be defined as follows:

```Go
type State struct {
	input  string
	offset int
}
```

This implementation now had one major advantage: a separate type gave me greater freedom. I could now define helper functions maybe add a few more features for better error handling. 

```Go
// Check if there are characters available for parsing
func (s State) HasAvailableChars(n int) bool {}

// Consume n characters and return
func (s State) Consume(n int) (string, error) {}

// Peek one char -- consume without advancing
func (s State) PeekChar() (byte, error) {}

// Advance n places and return the next state
func (s State) Advance(n int) State {}
```

The `State` API was the last thing I added to this library, but for the purpose of this blog let's assume I defined it in the beginning. So, with the State in place, I was now left with the `Result` and `Parser` types.

```Go
type Result struct {
	parsedResult interface
	nextState    State
}

type Parser func(curState State) (result, error)
```

Having the `Parser` type as a function would allow the combinators to be "technically" called higher-order functions. Now, all that was left was to define some basic parsers.

```Go

// basic char parser
func CharParser(c byte) Parser {
    return func(curState State) (Result, error) {
        if curState.offset >= len(curState.input) {
            return NewResult(
                nil,
                curState,
            ), fmt.Errorf("reached the end of input string while parsing")
        }
        
        if curState.input[curState.offset] != c {
            return NewResult(
                nil,
                curState,
            ), fmt.Errorf("expected %c but received %c", c, curState.input[curState.offset])
        }
        return NewResult(
            string(c),
            curState.Advance(1),
        ), nil
    }
}

// Parses a string exactly and advances the current State.
func String(s string) Parser {
    return func(curState State) (Result, error) {
        if curState.input[curState.offset:] != s {
            return NewResult(
                nil,
                curState,
            ), fmt.Errorf("expected %s", s)
        }
        return NewResult(
            s,
            curState.Advance(len(s)),
        ), nil
    }
}
```


# Mastering the Dark Art of Parser Combinators

Now that I had defined two of the most basic parsers, I started writing combinators. Here are a few of them:

```Go
// Lazily perform OR between the left and right parsers
func Or(left Parser, right Parser) Parser {
	return func(curState State) (Result, error) {
        leftRes, err := left(curState)
        if err != nil {
            curState = leftRes.nextState
            return right(curState)
        }
        return leftRes, nil
    }
}

// Lazily perform AND between the left and right parsers
func And(left Parser, right Parser) Parser {}

// Parse 0 or more occurence of the parser in the input/state machine
func Many0(p Parser) Parser {}

// Parse 1 or more occurence of the parser in the input/state machine
func Many1(p Parser) Parser {}
```

You might've noticed that there's only two params in the `Or` and `And` combinators. It could benefit from multiple params, but as of now I haven't changed them :(

Apart from the basic ones, there were some special combinators that I found really interesting. First was the `Map` combinator. Similar to other general-purpose programming languages, a `Map` combinator maps the output of a parser to a different function. The reason it is interesting is that it helps to convert a parsed string to a data type of our choice, which comes in handy in a lot of different ways.

For instance, if I had to write a parser that takes an input and gives me an integer, I would want a combinator such that:

```Go
digits := Or(CharParser('0'), CharParser('1'), ..., CharParser('9'))
digitParser := Many0(digits)

// a function that takes an array of strings and outputs an int
func parseInt(d []string) (int, error) {
	res := ""
	for _, val := range d {
		res = res + val
	}
	return strconv.Atoi(res)
} 

intParser := Map(digitParser, parseInt) // we need this
```

Although we could've mapped it ourselves, but that would mean that we would have to repeat the same set of lines for every mapping we wanted, which violates the DRY principle. So, here's the implementation of the `Map` combinator:

```Go
func Map[A, B any](p Parser, mapping func(A) B) Parser {
    return func(curState State) (Result, error) {
        res, err := p(curState)
        
        if err != nil {
            return NewResult(
                nil,
                curState,
            ), err
        }
        
        return NewResult(
            mapping(res.parsedResult.(A)),
            res.nextState,
        ), nil
    }
}
```

I have used generics here, because this is a sort of a workaround because Go doesn't support [parameterized methods](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#No-parameterized-methods).

The next few were equally interesting:
1. `Between` combinator. This combinator matches a parser surrounded by two others. This would later help us in matching content within parentheses, braces, or quotation marks.
2. `Sequence` combinator. This combinator sequentially passes the state from one parser to another, and returns a `union` of the parsed results if **all** the parsers pass.
3. `Lazy` combinator. This combinator is perhaps the most important of all the combinators. It defers the creation of a parser unless required. It plays a pivotal role in parsing recursively defined parsers (more on that later).

```Go

// Lazy parsing with memoization
func Lazy(f func() Parser) Parser {
    var memo Parser
    return func(curState State) (Result, error) {
        if memo == nil {
            memo = f()
        }
        return memo(curState)
    }
}

// Parse the content between open and close parser
func Between(open, content, close Parser) Parser {
    return func(curState State) (Result, error) {
        openRes, err := open(curState)
        if err != nil {
            return NewResult(
                nil,
                curState,
            ), err
        }
        
        contentRes, err := content(openRes.nextState)
        if err != nil {
            return NewResult(
                nil,
                curState,
            ), err
        }
        closeRes, err := close(contentRes.nextState)
        
        if err != nil {
            return NewResult(
                nil,
                curState,
            ), err
        }
        
        return NewResult(
            contentRes.parsedResult,
            closeRes.nextState,
        ), nil
    }
}

// Sequentially parse parsers
func Seq(parsers ...Parser) Parser {
    return func(curState State) (Result, error) {
        var res []interface{}
        next := curState
        for _, parser := range parsers {
            x, err := parser(next)
            if err != nil {
                return NewResult(
                    nil,
                    curState, // fallback to the initial State
                ), err
            }
            res = append(res, x.parsedResult)
            next = x.nextState
        }

        return NewResult(
            res,
            next,
        ), nil
    }
}
```

Phew! That was indeed a LOT of work. But I was only just halfway there. The final boss was waiting for me.

# Practical Dark Arts Defense

Now that I had all this set up, it was time for me to make a small DSL. For me, it was going to be a small JSON parser. A JSON can appear in various shapes:
- Numbers
- Strings
- Arrays
The trickiest part - handling arrays within arrays - could now be handled easily with the help of our favorite `Lazy` combinator. Here's how I crafted it:

```Go

func jsonString() parser.Parser {
    stringChar := func() parser.Parser {
        return func(curState parser.State) (parser.Result, error) {
            c, err := curState.PeekChar()
            if err != nil {
                return parser.NewResult(nil, curState), fmt.Errorf("unexpected end of input")
            }
            
            if c != '"' && c != '\\' {
                return parser.NewResult(
                    string(c),
                    curState.Advance(1),
                ), nil
            }
            
            return parser.NewResult(nil, curState), fmt.Errorf("unexpected character %s", string(c))
        }
    }

    return parser.Map(
        parser.Between(
            parser.CharParser('"'),
            parser.Many0(stringChar()),
            parser.CharParser('"'),
        ),

        func(chars interface{}) string {
            var sb strings.Builder
            for _, c := range chars.([]interface{}) {
                sb.WriteString(c.(string))
            }
            return sb.String()
        },
    )
}

// parse a json array

func jsonArray() parser.Parser {
    return parser.Map(
        parser.Between(
            parser.Seq(parser.CharParser('['), whitespace()),
            parser.Many0(
                parser.Seq(
                    parser.Lazy(func() parser.Parser { return jsonValue }),
                    parser.Many0(
                        parser.Seq(
                            parser.CharParser(','),
                            whitespace(),
                            parser.Lazy(func() parser.Parser { return jsonValue }),
                        ),
                    ),
                ),
            ),
            parser.Seq(whitespace(), parser.CharParser(']')),
        ),

        func(val interface{}) []interface{} {
            if len(val.([]interface{})) == 0 {
                return []interface{}{}
            }
            result := make([]interface{}, 0)
            seqResults := val.([]interface{})
            result = append(result, seqResults[0].([]interface{})[0])

            // items of the second seq in the form [[',', ' ', jsonvalue], ...]
            restItems := seqResults[0].([]interface{})[1].([]interface{})
            for _, item := range restItems {
                itemSeq := item.([]interface{})
                result = append(result, itemSeq[2])
            }

            return result
        },
    )
}

  
//  parse the JSON
func ParseJSON(input string) (interface{}, error) {
    jsonValue = parser.Or(
        parser.Or(
            jsonString(),
            jsonNumber(),
        ),
        parser.Lazy(jsonArray),
    )

    res, err := jsonValue(parser.NewState(input, 0))
    fmt.Println(res)
    if err != nil {
        return nil, err
    }

    return res, nil
}


func main() {
    // Test cases
    inputs := []string{
        `123`,
        `"Hello World"`,
        `[ 1, 2, "Hello World" ]`,
        `[ 1, 2, [ 1, 3 ] ]`,
        `[ ]`,
    }
  
    fmt.Printf("Test cases: \n %s", inputs)
    for _, input := range inputs {
        result, err := ParseJSON(input)
        if err != nil {
            fmt.Printf("Error parsing %s: %v\n", input, err)
            continue
        }
        fmt.Printf("Successfully parsed %s: %v\n", input, result)
    }
}
```

# Beyond the Chamber

Parser combinators, although simple on their own, are really powerful tools. As Dumbledore might say, "It is not our parsing abilities that make us good programmers, but how we combine them."

But, as the saying goes, with great power comes great responsibility. Parser combinators are only used in domain-specific tasks that aren't too complex. This is because general-purpose languages are complex and they benefit from the separation of concerns provided by having a separate lexer and parser.

# Links
- [Parser Combinator Library](https://github.com/BlackBuck/pcom-go)
- [JSON Parser Implementation](https://github.com/BlackBuck/gopi)
- Bonus: [Lazy Evaluation- Wikipedia](https://en.wikipedia.org/wiki/Lazy_evaluation)