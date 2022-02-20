# Lexer

The `lexer` is responsible for tokenizing the source code. It receives of piece of code written in EVM op codes and returns tokens distributed over a channel.

A `token` is defined as follow:
```
type token struct {
	typ    tokenType
	lineno int
	text   string
}
```

