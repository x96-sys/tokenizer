# Tokenizer

Respects [Basic Latin (ASCII)](https://www.unicode.org/charts/PDF/U0000.pdf)

Implements [Basic table](Kind.md)

**Patterns**: `Visitor`; `Chain Of Responsibility`; `Strategy`

## Flow

#### Receives/Starts

- **string**
  - auto-adds `SOI` (`0x2`) sentinel at the start
  - converts string to an integer array
  - auto-adds `EOI` (`0x3`) sentinel at the end
- **integer array**
  - no sentinel is included

#### Concludes/Returns

- array of `Token` with `Kind`, `Lexeme`, and `Span`

## Example

#### Empty String

```java
Cortex cortex = new Cortex();
for (Token token : cortex.tokenStream("")) {
    System.out.println(token);
}
```

Output:

```
Token { kind[SOI] Lexeme[0x2] Span[{0:0 0}:{1:1 1}] }
Token { kind[EOI] Lexeme[0x3] Span[{1:1 1}:{1:2 2}] }
```

#### Example with a simple rule

```java
Cortex cortex = new Cortex();
for (Token token : cortex.tokenStream("c=s;")) {
    System.out.println(token);
}
```

Output:

```
Token { kind[SOI]       Lexeme[0x2]  Span[{0:0 0}:{1:1 1}] }
Token { kind[LATIN]     Lexeme[0x63] Span[{1:1 1}:{1:2 2}] }
Token { kind[EQUALS]    Lexeme[0x3D] Span[{1:2 2}:{1:3 3}] }
Token { kind[LATIN]     Lexeme[0x73] Span[{1:3 3}:{1:4 4}] }
Token { kind[SEMICOLON] Lexeme[0x3B] Span[{1:4 4}:{1:5 5}] }
Token { kind[EOI]       Lexeme[0x3]  Span[{1:5 5}:{1:6 6}] }
```

👀 In this layer, the tokenizer responds to the [pattern defined](Kind.md) by
`Kind`, and further on, the `Visitors` can return re-marked tokens, signaling
elements for the `Parser`.

Each character of this sentence you are reading corresponds to a hexadecimal
value, where a previous, discreet, and silent tokenizer has already been
present.

Let's consider the phrase:

```
"Sofi Ceci"
```

There is a binary for each char, which in turn can be represented in octal,
decimal, and as adopted in the project, can also be expressed in hexadecimal:

| Char |     BIN     |  OCT  |  DEC  |  Hex   |
| :--: | :---------: | :---: | :---: | :----: |
| `S`  | `0101 0011` | `123` | `83`  | `0x53` |
| `o`  | `0110 1111` | `157` | `111` | `0x6F` |
| `f`  | `0110 0110` | `146` | `102` | `0x66` |
| `i`  | `0110 1001` | `151` | `105` | `0x69` |
| ` `  | `0010 0000` | `40`  | `32`  | `0x20` |
| `C`  | `0100 0011` | `103` | `67`  | `0x43` |
| `e`  | `0110 0101` | `145` | `101` | `0x65` |
| `c`  | `0110 0011` | `143` | `99`  | `0x63` |
| `i`  | `0110 1001` | `151` | `105` | `0x69` |

There are several ways to represent the same thing. The same number can
represent different values depending on the context it fits into.

Between `CHAR` and `BIN`, there is already a `Tokenizer`; between `BIN` and
`OCT`, there is another.

A new table respects a previous one to represent its value.

This **Tokenizer** [respects the basic table, fully described here](Kind.md).

This contract is static and is the core of the Tokenizer.

What follows is preceded by these basic rules.

## Implementation Example

The basic table responds to:

```
Kind.is(hex)
```

and returns the corresponding tabulated `Token`.

The visitor implements/calls `Kind.is(hex)` and builds its response according to
its rules.

In the [Example with a simple rule](#example-with-a-simple-rule), we check that
`c` was **tokenized** as `LATIN`, and this respects the [basic table](Kind.md).

Now in this layer, we implement the `Visitor` that recognizes a grammar we are
defining.

The `IDENTITY` rule does not appear in the [basic table](Kind.md).

Designing the `IdentityVisitor` ensures that:

- any `LATIN` can start an `IDENTITY`
- any `LATIN` and `LOW_LINE` can follow the `IDENTITY`

🦇 By implementing a `Visitor`, we can tabulate a new rule, generating changes:

- `c` was `LATIN`
- `x` was `LATIN_SMALL_LETTER_X`

Now, by including the `IdentityVisitor` in the `VisitorChain`, it is able to
respond, to **tokenize** `c` and `x` as `IDENTITY`.

The `Tokenizer` has the `tokenize` method that responds with the tabulated
`Kind`, and also has `tokenize(kind)` which receives a `Kind` for programmed
registration in the `Visitor`.

The `IdentityVisitor` calls `tokenizer.tokenize(Kind.IDENTITY)`.

🦇 Implementing the `WordVisitor`:

The word syntax is `'anything between apostrophes'`.

The `WordVisitor` starts visits with `APOSTROPHE` without needing to
re-tabulate; it calls `tokenize()` with the already default `Kind`.

Next, anything will be identified as `ANY` by calling `tokenize(Kind.ANY)`,
until it finds another `APOSTROPHE` and calls `tokenize()` again without
re-tabulation.

All this for...

For example, a `Parser` ahead, which will read the token stream and needs a
validation of `meaning`/`Kind`, and also the guarantee of the order that each
`Visitor` delivers.

`x` in the middle of an identifier is `IDENTITY`. `x` after a `0` is a
hexadecimal marker.

This is the magic of the Tokenizer.

🧙‍♂️
