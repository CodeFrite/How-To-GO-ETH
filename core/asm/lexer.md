# Lexer

The `lexer` is responsible for tokenizing the source code for the compiler which converts them to bytecodes. It accepts as an input a source code, parses it and returns tokens distributed over a channel. Here are a few examples of inputs and outputs used in the `lexer_test.go` file:
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

The principle of the `lexer` is pretty straight forward. It reads the source code, character by character and tries to form words by follwing a defined grammar. In the case of EVM opcode, a line of code can contain a:
- `comment` which starts with `;;`
- `element` in alpha characters
- `number` in digits
- `stringValue` in alpha characters
- `label` used for the `JUMP` instruction and beginning with the character `@`
- `endline` \n character
- `eof` character (end of file)

Given the following input, the lexer will slice the code in the following places and emit tokens:
```
input(source) => output(tokens):  

;; this is a comment \n		=> (startLine, 0, ''), (endLine, 0, '')
PUSH 666 \n			=> (startLine, 1, ''), (element, 1, 'PUSH'), (number, 1, '666'), (endLine, 1, '')
PUSH 111 \n			=> (startLine, 2, ''), (element, 2, 'PUSH'), (number, 1, '666'), (endLine, 2, '')
ADD \n				=> (startLine, 3, ''), (element, 3, 'ADD'), (eof, 3, '')
```

Let's dive into the code and learn more about the internal types and the helpers functions.

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
	input string 			// input contains the source code of the program

	tokens chan token 		// tokens is used to deliver tokens to the listener
	state  stateFn    		// the current state function

	lineno            int 		// current line number in the source file
	start, pos, width int 		// positions for lexing and returning value

	debug bool 			// flag for triggering debug output
}
```

## Helper functions

**+++ next()**

The helper function `next()` is used to ...:
```
func (l *lexer) next() (rune rune) {
	if l.pos >= len(l.input) {
		l.width = 0
		return 0
	}
	rune, l.width = utf8.DecodeRuneInString(l.input[l.pos:])
	l.pos += l.width
	return rune
}

![image](https://user-images.githubusercontent.com/34804976/154867481-2092ba6e-7f61-4f3a-a294-812a98d0e292.png)

**+++ backup()**

```
// backup backsup the last parsed element (multi-character)
func (l *lexer) backup() {
	l.pos -= l.width
}
```

**+++ peek()**

The function `lexer.peek` allows us to get the value of the next rune without changing the `lexer.pos` value:
```
// peek returns the next rune but does not advance the seeker
func (l *lexer) peek() rune {
	r := l.next()
	l.backup()
	return r
}
```

**+++ ignore()**

What actually does `lexer.ignore()`. It simply sets the lexer.start to the value lexer.pos which is equivalent as skipping the rune:
```
func (l *lexer) ignore() {
	l.start = l.pos
}
```

Now that we have a basic understanding of the data structure and helper functions, let's look the the lexer main entry point, the `Lex` function.

## Lexer entry point: `Lex()`

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

A Go routine is then launched in a separate thread to enable the lexer to process the source code and emit the tokens as they are processed on the `lexer.tokens` channel:
```
go func() {
	l.emit(lineStart)		// each line begins with a `lineStart` token
	for l.state != nil {		
		l.state = l.state(l)	// replace the state fct by the result of the call to the state function
	}
	l.emit(eof)			// if there are no more characters in the source code, emit an `eof` token
	close(l.tokens)			// close the `tokens` channel
}()
```

While the go routine is still running, we return the `tokens channel` to the caller:
```
return ch 	// which corresponds to the Lex function return value: func Lex(source []byte, debug bool) <-chan token {
```

### The state function: `lexLine`

The state function defines how the source should be converted to tokens. Any function with the following signature can be used as a state function:
```
type stateFn func(*lexer) stateFn
```

In the `lexer.go` file, it uses the function `lexLine`:
```
func lexLine(l *lexer) stateFn {
}
```

This function will parse the next `utf-8` character and will conditionaly decide to emit a token over the channel:
```
for {
	switch r := l.next(); {
	case r == '\n':
		...
	case r == ';' && l.peek() == ';':
		...	
	case isSpace(r):
		...		
	case isLetter(r) || r == '_':
		...			
	case isNumber(r):
		...			
	case r == '@':
		...			
	case r == '"':
		...	
	default:
		return nil
	}
}
```

**+++ lexLine: New line**

If the lexer encounters an endline `\n` character, it will emit a `lineEnd` token and ignore the current character. As we have a new line of code, the lexer will increment the line number and emit a `lineStart` token:
```
case r == '\n':
	l.emit(lineEnd)
	l.ignore()
	l.lineno++
	l.emit(lineStart)
```

**+++ lexLine: Comment line**

Comments in EVM bytecodes begin with ";;". Therefore, if the next rune is the character ";", we peek into the source []byte to see if the next character is also ";". If it is the case, we lex the current line as a `comment line`:
```
case r == ';' && l.peek() == ';':
	return lexComment
```

**+++ lexLine: New line**

```
```
