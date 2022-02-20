# Lexer

The `lexer` is responsible for tokenizing the source code. It accepts as an input a source code and returns tokens distributed over a channel. Here are a few examples of inputs and outputs used in the `lexer_test.go` file:
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

A `token` is defined as follow:
```
type token struct {
	typ    tokenType
	lineno int
	text   string
}
```

