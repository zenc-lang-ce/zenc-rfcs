# RFC: Every statement ends in `;`

- **Author(s):** @pherrymason
- **Status:** Draft
- **Created:** 2026-11-24
- **Target:** the language specification and the parser
- **Release:** 0.6 (breaking). The 0.5.x line keeps today's behavior and adds a warning.

---

## 1. Summary

A statement ends in `;`. Today the `;` can be left out and the end of the line ends the
statement instead, by rules that are not written down and that differ from one operator to
the next. This RFC makes the `;` mandatory, so a line break means nothing to the parser and
an expression can be split over several lines in any place.

## 2. Motivation

### The rule today

The reference does not say that `;` is optional, but the parser accepts this:

```zc
fn pick(a: int, p: int*) -> int {
    let x = a
    *p = 5
    return x
}
```

To read it as two statements and not as `let x = a * p = 5`, the expression parser stops at a
line break when the next line starts with `*`, `&`, `+` or `-`: the four binary operators that
are also prefix operators. Every other token continues the expression. So what a line break
does depends on the first token of the next line:

| Next line starts with | The expression | Example |
|---|---|---|
| `*`, `&`, `+`, `-` | ends | `a` ⏎ `- b` is `a` and then `-b` |
| `/`, `%`, `==`, `<`, `&&`, `\|\|`, `\|`, `^`... | continues | `a` ⏎ `/ b` is `a / b` |
| `.` | continues | `p` ⏎ `.get()` is `p.get()` |
| `[` | continues | `a` ⏎ `[0]` is `a[0]` |
| `(` | continues | `f` ⏎ `(g)()` is `f(g)()` |

The rule holds inside parentheses too, where no statement can start.

### What it costs

**A program that means one thing and compiles as another, with no warning.** Both of these
compile with 0 errors and 0 warnings:

```zc
let c = a
    - b;          // c is a. `-b;` is a second statement that does nothing

let d = a
    & b;          // d is a. With `&&` in place of `&`, d is a && b
```

**Code that cannot be written the usual way.** A long sum cannot put its operators at the
start of the lines, with or without parentheses:

```zc
let total = first
    + second;     // error: Unexpected token '+' while parsing expression

let total = (first
    + second);    // error: Expected )
```

**The opposite mistake.** A statement without `;` followed by a line that starts with `(` or
`[` is joined to it:

```zc
let f = one
(one)()           // error: Too many arguments: 'one' expects 0, got 1
```

**A rule that is already not uniform.** A top-level `let` requires its `;` ("Expected ';'
after the value of 'total'") and a local one does not; `extern fn` requires it and `def`,
`alias` and `import` do not. Two statements can share a line with nothing between them:
`let a = 1 let b = 2` compiles.

**Bugs in the compiler.** Each kind of statement decides on its own whether a `;` follows,
in about 30 places of the parser, and the cases without `;` are the ones that get less
testing. Issues #141 (a string statement), #148 (`defer`) and #379 (a string statement
again) of the original repository were all "it breaks when the semicolon is omitted".

### How much code leaves it out

A rough count over the `.zc` files of this organization, with a script that looks at lines
that are a `let`, an assignment, a call, a `return`, a `break`, a `continue` or a `defer`:

| Code | Statement lines | Without `;` |
|---|---|---|
| `std/` | about 2200 | 0 |
| `tests/` | about 5300 | about 150 (3%) |
| `awesome-zenc` | about 3500 | about 40 (1%) |

The standard library never leaves it out. The style the examples teach is the one with `;`.

## 3. Detailed Design

### The rule

1. A statement that is not a block ends in `;`: `let`, `def`, an assignment, an expression
   statement (a call, `x++`), `return`, `break`, `continue`, `defer`, `import`, `alias`,
   `opaque alias` and a function declaration without a body (`extern fn`).
2. A statement that ends in its own block takes no `;` after the `}`, as today: `if`, `for`,
   `while`, `loop`, `match`, `test`, `fn`, `struct`, `enum`, `trait`, `impl`.
3. A line break is white space. The check on `*`, `&`, `+` and `-` at the start of a line is
   removed.
4. An empty statement (a `;` on its own) stays valid, as today.

```zc
// Before: accepted                     // After
let a = 1                               let a = 1;
let b = add(a, 2)                       let b = add(a, 2);
defer close(file)                       defer close(file);
return b                                return b;
```

```zc
// After: every one of these is one expression
let total = first
    + second
    - third;

let mask = flags
    & 0x0F;

let ready = (count > 0
    && !paused);
```

### What does not change

- **Values of blocks.** The last expression of a block that is used as a value takes no `;`:
  `let c = if a > 0 { a } else { 0 };`. The `;` at the end is the one of the `let`.
- **Match arms.** `4 => 1,` is an arm with an expression, separated by commas.
- **Lambdas with an expression as body:** `(x: int) => x + 1`.
- **Fields of a struct**, separated by `;` or `,`, with the last separator optional. They
  are declarations inside a list, not statements.

### Diagnostics

A missing `;` is reported at the end of the statement that lacks it, not at the token of the
next line where the parser found out:

```
error: Expected ';' after this statement
  --> main.zc:4:14
   |
4  |     let x = a
   |              ^ here
```

The parser then goes on as if the `;` were there, so one missing `;` is one error.

### Transition

- **0.5.x:** a statement that ends at a line break without `;` is a warning, with a note that
  it will be an error from 0.6. This is the form 0.5.2 already uses for what 0.6 rejects.
  Nothing changes in how programs are read.
- **0.6:** the rule above.

No program changes its meaning without notice. The two readings only differ for a line that
starts with `*`, `&`, `+` or `-`: in 0.6 it continues the statement before it. If that
statement had no `;`, 0.5.x has already warned about it. If it had one, nothing is joined.
The programs that do change are the ones of section 2 that compile today as something
their author did not write (`let c = a` ⏎ `- b;`), and those become what they say.

### In the compiler

The parser has about 30 places of the form "if the next token is `;`, consume it". They
become one function that expects the `;` and reports it missing, called by every statement
of rule 1. The line check in the expression parser is deleted. The lexer and the generated
C do not change.

## 4. Drawbacks

- **It breaks programs.** About 3% of the statement lines of the test suite, and whatever
  users have written in that style, need a `;` added. The change is mechanical and the
  compiler says where.
- **It takes away a style some users like.** A program without `;` reads lighter, closer to
  Go, Swift or Kotlin.
- **The REPL.** A line typed at the prompt would need its `;`. The REPL can add it at the
  end of the input on its own (see "Unresolved questions").
- **A second breaking change to explain** in the 0.6 release, next to the others.

## 5. Alternatives Considered

**Do nothing.** The silent cases stay, and so does the limit on how an expression can be
split. Every new kind of statement keeps having to get the case without `;` right.

**Fix only the worst cases, in 0.5.x.** Two changes that break no valid program: do not
apply the line check inside parentheses and brackets, and warn about an expression statement
that does nothing (`-b;`). They are worth doing whatever is decided here, but outside
parentheses `first` ⏎ `+ second` remains an error, and the rule remains a table to memorize.

**Keep `;` optional and write a precise rule, as Go does.** A line break ends the statement
when the line ends in a token that can end one (a name, a literal, `)`, `]`, `}`, `return`,
`break`, `continue`, `++`). The rule is simple and has no silent cases, but it forbids an
operator at the start of a line for good, and `f` ⏎ `(g)()` or a method chain split before
the `.` need their own exceptions. It is a larger design than this one and it keeps two
ways of writing every program.

**Keep `;` optional and continue on every binary operator.** Then `let x = a` ⏎ `*p = 5`
becomes a multiplication: a pointer write at the start of a line, which is common, would
need a `;` on the line before it. It moves the silent case to another place.

## 6. Prior Art

- **C, C++, Rust, Zig:** `;` is mandatory. In Rust a block's last expression without `;` is
  its value, the same exception this RFC keeps.
- **Go:** the lexer inserts a `;` at a line break after a token that can end a statement.
  This is why Go code cannot start a line with a binary operator or put `{` on its own
  line, and why `gofmt` is not optional in practice.
- **JavaScript:** automatic semicolon insertion works the other way round: the line break
  ends the statement only if going on would be a syntax error. Its known traps are the
  mirror image of the ones in section 2: `return` ⏎ `value` returns nothing, and a line
  that starts with `(` or `[` joins the one before it.
- **Swift, Kotlin:** no `;`, with rules of their own for where an expression can be split.
  Kotlin has the same silent case as Zen C today: `val c = a` ⏎ `- b` is two statements.

Zen C generates C, its syntax is close to C's and Rust's, and its standard library is
already written with `;` everywhere. The mandatory `;` is the smaller rule.

## 7. Unresolved Questions

1. **The REPL:** should a line at the prompt need its `;`, or does the REPL add one at the
   end of each input?
2. **A tool for the migration:** is the 0.5.x warning enough, or should `zc-format` (or a
   `zc fix`) add the missing `;`? The warning already knows every position.
3. **A `;` after the last expression of a block that is a value.** Today
   `if a > 0 { a; } else { 0; }` gives the same value as without the `;`. Whether a `;`
   there should discard the value, as in Rust, is a separate question, left out of this RFC.
4. **Top-level declarations that end in a block** are covered by rule 2. Is there any
   declaration where a `;` after the `}` should be required, as C requires after a
   `struct`? This RFC says no.
