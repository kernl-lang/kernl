# Getting Started with kernl

This guide walks you through the kernl language from first principles. By the end, you'll understand every construct in the language and how they fit together.

## Your first program

Create a file called `add_one.knl`:

```
fn add_one
  in x: int
  out result: int
  do add x 1
```

This declares a function `add_one` that takes an integer `x` and returns `x + 1`.

Compile it:

```bash
kernlc hello.knl --target debug   # see the parsed structure
kernlc hello.knl --target llvm    # emit LLVM IR
kernlc hello.knl --target wasm    # emit WebAssembly
```

To **run** a program with a **`main`** entrypoint from the compiler build directory, use the **`kernl`** driver (implies **`--run`**; pipe stdin when **`main`** takes one **`str`**):

```bash
./target/debug/kernl my_program.knl
```

## Anatomy of a function

Every kernl function follows the same structure:

```
fn <name>
  in  <params>
  out <return>
  inv <invariant>
  do  <body>
```

| Clause | Required | Purpose |
|--------|----------|---------|
| `fn` | Yes | Declares the function and its name |
| `in` | No | Input parameters with types |
| `out` | No | Return binding and type |
| `inv` | No | Invariants that must hold (can have multiple) |
| `do` | Yes | The implementation |

Each clause goes on its own line. There are no braces, no semicolons, no indentation rules. The parser reads clause keywords to determine structure.

## Types

### Primitives

| Type | Description | Example |
|------|-------------|---------|
| `int` | 64-bit signed integer | `42`, `-7` |
| `uint` | Unsigned integer | `0`, `255` |
| `float` | 64-bit floating point | `3.14` |
| `bool` | Boolean | `true`, `false` |
| `str` | UTF-8 string | `"hello"` |
| `void` | No value | |

### Compound types

```
[int]           # list of integers
{str: int}      # map from string to integer
(int, str)      # tuple of integer and string
int?            # optional integer (int or nothing)
```

## Operators

kernl uses **named operators** instead of symbols. This is a deliberate design choice — LLMs confuse `>=`, `=>`, `->`, and `>>` constantly. Named operators are unambiguous.

| Operator | Meaning | Equivalent |
|----------|---------|------------|
| `add` | addition | `+` |
| `sub` | subtraction | `-` |
| `mul` | multiplication | `*` |
| `div` | division | `/` |
| `modulo` | modulo | `%` |
| `eq` | equal | `==` |
| `neq` | not equal | `!=` |
| `gt` | greater than | `>` |
| `lt` | less than | `<` |
| `gte` | greater or equal | `>=` |
| `lte` | less or equal | `<=` |
| `and` | logical and | `&&` |
| `or` | logical or | `\|\|` |
| `not` | logical not | `!` |

Operators are **prefix** with **fixed arity** — binary operators take exactly 2 arguments, `not` takes 1:

```
add x 1          # x + 1
gt score 100     # score > 100
not is_empty     # !is_empty
```

## Pipes

The pipe `|` is kernl's composition operator. It takes the result of the left expression and feeds it as the first argument to the right expression.

```
filter nums gt 0 | reduce add
```

This reads left to right: "filter nums where greater than 0, then reduce with add." Without pipes, this would require nested calls: `reduce(add, filter(nums, gt, 0))`.

Pipes keep code **flat** — a critical property for LLM generation, because nested parentheses require the model to track long-range dependencies.

## Invariants

Invariants are contracts that the compiler checks. They declare what must be true about the function's behavior.

```
fn clamp
  in  val: int lo: int hi: int
  out result: int
  inv gte result lo
  inv lte result hi
  do  max lo min hi val
```

The two `inv` clauses state: "the result is always between `lo` and `hi`." The type checker verifies these are boolean expressions. Future versions will statically prove them.

You can have zero or many invariants per function.

## Bindings

```
let x: int = 42
let name: str = "kernl"
```

Bindings are **immutable by default**. Use `mut` for mutable bindings:

```
mut counter: int = 0
```

## Control flow

### Conditionals

```
if gt x 0
  "positive"
elif eq x 0
  "zero"
else
  "negative"
end
```

Blocks are terminated with `end` — no braces to match.

### Iteration

```
each item in collection
  process item
end

while gt remaining 0
  step remaining
end
```

## Structs

```
struct Account
  id: uint
  balance: int
  owner: str
end
```

Access fields with dot notation: `account.balance`

Temporal references use `@`: `balance@pre` refers to the value of `balance` before the current operation (useful in invariants).

## Modules

```
mod math
use io.print
```

`mod` declares the current module. `use` imports from another module.

## Fluid mode

Fluid mode is kernl's unique feature for AI-era programming. It lets functions declare **intent** instead of (or in addition to) implementation:

```
fn recommend
  mode fluid
  in  user: User context: Context
  intent "surface items user would engage with"
  confidence 0.85
  fallback popular_items context
```

| Clause | Purpose |
|--------|---------|
| `mode fluid` | Marks this function for runtime resolution |
| `intent` | Natural language description of desired behavior |
| `confidence` | Minimum confidence threshold (0.0 to 1.0) |
| `fallback` | Expression to use when confidence is not met |

Fluid functions compile to a runtime that can invoke external resolvers (including LLMs) for underspecified regions. If the resolver's confidence is below the threshold, the fallback executes instead.

## Complete example

Here's a realistic kernl program combining structs, invariants, pipes, and guarantees:

```
struct Account
  id: uint
  balance: int
  owner: str
end

fn transfer
  in  amount: uint from: Account to: Account
  inv gte from.balance amount
  guarantee atomic
  do  debit from amount | credit to amount
```

This declares:
1. An `Account` struct with three fields
2. A `transfer` function that takes an amount and two accounts
3. An invariant: the sender must have sufficient balance
4. A guarantee: the operation is atomic
5. The implementation: debit the sender, pipe to credit the receiver

In 12 lines, with zero syntactic noise.

## Next steps

- **Agents / AI assistants:** [docs/agents/README.md](agents/README.md) — parser quirks, pipes, builtins vs executor, checklists
- **Full specification:** [spec/LANGUAGE.md](../spec/LANGUAGE.md) — every keyword, type, and operator
- **Annotated examples:** [docs/examples.md](examples.md) — real programs with explanations
- **Compiler architecture:** [docs/architecture.md](architecture.md) — how the compiler works internally
- **Contributing:** [CONTRIBUTING.md](../CONTRIBUTING.md) — how to get involved
