# Lexer

The `lexer` is responsible for tokenizing the source code. It accepts as an input a source code, parses it and returns tokens distributed over a channel. Here are a few examples of inputs and outputs used in the `lexer_test.go` file:
```
tests := []struct {
	input  string
	tokens []token
}{
	{
		input:  ";; this is a comment",
		tokens: []token{{typ: lineStart}, {typ: eof}},
	},
	{
		input:  "0x12345678",
		tokens: []token{{typ: lineStart}, {typ: number, text: "0x12345678"}, {typ: eof}},
	},
	{
		input:  "0x123ggg",
		tokens: []token{{typ: lineStart}, {typ: number, text: "0x123"}, {typ: element, text: "ggg"}, {typ: eof}},
	},
	...
}
```

## Types

A `token` is defined as follow:
```
type token struct {
	typ    tokenType	// token type in: eof, lineStart, lineEnd, ...
	lineno int		// line on which the token belong
	text   string		// text of the token
}
```

The `typ tokenType` is defined as an int that finds its value in the following constants table:
```
type tokenType int

const (
	eof              tokenType = iota // end of file
	lineStart                         // emitted when a line starts
	lineEnd                           // emitted when a line ends
	invalidStatement                  // any invalid statement
	element                           // any element during element parsing
	label                             // label is emitted when a label is found
	labelDef                          // label definition is emitted when a new label is found
	number                            // number is emitted when a number is found
	stringValue                       // stringValue is emitted when a string has been found

	Numbers            = "1234567890"                                           // characters representing any decimal number
	HexadecimalNumbers = Numbers + "aAbBcCdDeEfF"                               // characters representing any hexadecimal
	Alpha              = "abcdefghijklmnopqrstuwvxyzABCDEFGHIJKLMNOPQRSTUWVXYZ" // characters representing alphanumeric
)
```

Corresponding labels are defined in the variable `stringtokenTypes`:
```
var stringtokenTypes = []string{
	eof:              "EOF",
	lineStart:        "new line",
	lineEnd:          "end of line",
	invalidStatement: "invalid statement",
	element:          "element",
	label:            "label",
	labelDef:         "label definition",
	number:           "number",
	stringValue:      "string",
}
```

A lexer is defined as follow:
```
type lexer struct {
	input string // input contains the source code of the program

	tokens chan token // tokens is used to deliver tokens to the listener
	state  stateFn    // the current state function

	lineno            int // current line number in the source file
	start, pos, width int // positions for lexing and returning value

	debug bool // flag for triggering debug output
}
```

## Lexer entry point: `Lex`

The entry point of the `lexer` is the function `Lex`:
```
func Lex(source []byte, debug bool) <-chan token {
	...
}
```

It first creates an `unbuffered token channel` that will be used by the lexer to retrieve to the listener the tokens, one by one:
```
ch := make(chan token)
```

It then initialises a `lexer`:
- `lexer.input`: string where the source code will be saved
- `lexer.tokens`: tokens channel used to get the tokenized source code
- `state`: state function that is used to parse the source code. Here it uses the function `lexLine`
- `debug`: debug flag that indicates whether the lexer should output the tokens during execution
```
l := &lexer{
	input:  string(source),
	tokens: ch,
	state:  lexLine,
	debug:  debug,
}
```
