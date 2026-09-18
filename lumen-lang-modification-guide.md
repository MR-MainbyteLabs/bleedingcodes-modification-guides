# lumen-lang — Modification Guide
**Version:** 0.1.1  
**Package:** `lumen/`  
**Entry points:** `lumen-run` (CLI), `lumen-repl` (REPL), `python -m lumen`  
**Dependencies:** stdlib only

---

## Who This Document Is For

Engineers and developers who want to add syntax, extend the runtime, add native functions, or embed the language in a larger application. This is not a usage guide. This document covers the full pipeline with actual class and function names.

---

## Pipeline Overview

```
source string
    ↓
Lexer.scan_tokens()       → list[Token]
    ↓
Parser.parse()            → list[Stmt]   (AST)
    ↓
Compiler.compile()        → FunctionProto  (bytecode)
    ↓
VM.run(proto)             → result value
```

The `runtime` module provides convenience wrappers:

```python
from lumen.runtime import compile_source, execute, disassemble_source

proto = compile_source("let x = 1 + 2;")
result = execute("print(42);", trace=False)
disassembly = disassemble_source("fn add(a, b) { return a + b; }")
```

---

## Module Map

| Module | File | Owns |
|---|---|---|
| `token.py` | `lumen/token.py` | `TokenType` enum, `KEYWORDS` dict, `Token` dataclass |
| `ast_nodes.py` | `lumen/ast_nodes.py` | All `Expr` and `Stmt` AST node classes |
| `lexer.py` | `lumen/lexer.py` | `Lexer` — source → tokens |
| `parser.py` | `lumen/parser.py` | `Parser` — tokens → AST (Pratt parser) |
| `bytecode.py` | `lumen/bytecode.py` | `Op` enum, `Instruction`, `Chunk`, `disassemble()` |
| `compiler.py` | `lumen/compiler.py` | `Compiler`, `FunctionProto`, `Local` — AST → bytecode |
| `vm.py` | `lumen/vm.py` | `VM`, `Closure`, `Cell`, `Native`, `Frame` — bytecode execution |
| `runtime.py` | `lumen/runtime.py` | `compile_source()`, `execute()`, `disassemble_source()` |
| `cli.py` | `lumen/cli.py` | `run` and `repl` entry points |

---

## Token System — `token.py`

### `TokenType` (Enum)

All token types. Current keywords:

```python
KEYWORDS = {
    "let": TokenType.LET,
    "fn": TokenType.FN,
    "return": TokenType.RETURN,
    "if": TokenType.IF,
    "else": TokenType.ELSE,
    "while": TokenType.WHILE,
    "true": TokenType.TRUE,
    "false": TokenType.FALSE,
    "null": TokenType.NULL,
}
```

**To add a new keyword** (e.g. `for`):

1. Add `FOR = auto()` to `TokenType`.
2. Add `"for": TokenType.FOR` to `KEYWORDS`.
3. Add the AST node in `ast_nodes.py`.
4. Add parsing in `parser.py`.
5. Add compilation in `compiler.py`.

### `Token` (frozen dataclass)

```python
@dataclass(frozen=True, slots=True)
class Token:
    type: TokenType
    lexeme: str       # raw source text
    literal: object   # parsed value (float for NUMBER, str for STRING, None otherwise)
    line: int
    column: int
```

---

## Lexer — `lexer.py`

`Lexer(source).scan_tokens() -> list[Token]`

Single-character tokens are dispatched via a `singles` dict in `_scan_token()`. Two-character tokens (`!=`, `==`, `<=`, `>=`, `&&`, `||`) are matched with `_match()`. `//` starts a line comment.

**To add a new operator** (e.g. `**` for exponentiation):

In `_scan_token()`, after the `/` case:
```python
elif char == "*":
    if self._match("*"):
        self._add(TokenType.STAR_STAR)
    else:
        self._add(TokenType.STAR)
```

Add `STAR_STAR = auto()` to `TokenType`, an AST node or binary operator string, and a compilation case.

**To add a new string escape** (e.g. `\t`):

In the `_string()` method:
```python
if self._peek() == "t":
    self._advance()
    value += "\t"
```

---

## AST Nodes — `ast_nodes.py`

Two base classes: `Expr` and `Stmt`. All nodes are `@dataclass(slots=True)`.

**Expressions:** `Literal`, `Variable`, `Assign`, `Unary`, `Binary`, `Logical`, `Call`, `ListLiteral`, `DictLiteral`, `Index`

**Statements:** `ExpressionStmt`, `LetStmt`, `BlockStmt`, `IfStmt`, `WhileStmt`, `FunctionStmt`, `ReturnStmt`

**To add a new expression node** (e.g. `ForRange` for `for x in range(n)`):

```python
@dataclass(slots=True)
class ForRange(Stmt):
    var: str
    iterable: Expr
    body: Stmt
```

---

## Parser — `parser.py`

Pratt (top-down operator precedence) parser. Key methods:

```python
Parser.parse() -> list[Stmt]          # entry point — returns all top-level statements
Parser._statement() -> Stmt            # dispatches on current token type
Parser._expression() -> Expr          # entry for expression parsing
Parser._parse_precedence(min_prec)     # Pratt core — handles binary operators
```

### Adding a New Statement

In `_statement()`:
```python
if self._check(TokenType.FOR):
    return self._for_statement()
```

```python
def _for_statement(self) -> ForRange:
    self._consume(TokenType.FOR, "Expected 'for'.")
    var = self._consume(TokenType.IDENTIFIER, "Expected variable name.").lexeme
    self._consume(TokenType.IN, "Expected 'in'.")
    iterable = self._expression()
    body = self._block()
    return ForRange(var, iterable, body)
```

### Adding a New Binary Operator

The Pratt table maps token types to `(left_bp, right_bp, operator_string)`. Find the existing binary operator table and add:

```python
TokenType.STAR_STAR: (60, 61, "**"),   # right-associative (right_bp > left_bp)
```

---

## Bytecode — `bytecode.py`

### `Op` (IntEnum)

All opcodes. Current set:

```
CONSTANT, NULL, TRUE, FALSE, POP
GET_LOCAL, SET_LOCAL, GET_GLOBAL, DEFINE_GLOBAL, SET_GLOBAL, GET_UPVALUE, SET_UPVALUE
EQUAL, GREATER, LESS
ADD, SUBTRACT, MULTIPLY, DIVIDE, MODULO
NOT, NEGATE
JUMP, JUMP_IF_FALSE, LOOP
CALL, CLOSURE, CLOSE_UPVALUE, RETURN
BUILD_LIST, BUILD_DICT, INDEX
```

**To add a new opcode** (e.g. `POWER`):

1. Add `POWER = auto()` to `Op`.
2. Emit it in `Compiler` for `**` expressions.
3. Handle it in `VM.run()`.

### `Chunk`

```python
class Chunk:
    code: list[Instruction]      # instruction stream
    constants: list[object]      # constant pool

    def add_constant(self, value) -> int   # returns constant index
    def emit(self, op, operand=None) -> int   # returns instruction index
    def patch(self, index, operand)   # back-patch a jump target
```

`patch()` is used for forward jumps: emit the jump with a placeholder operand, compile the jump target, then patch the operand with the actual offset.

### `disassemble(chunk, name) -> str`

Human-readable disassembly. Nested `FunctionProto` objects in the constant pool are disassembled recursively. Useful for debugging compilation output.

---

## Compiler — `compiler.py`

### `FunctionProto`

```python
@dataclass(slots=True)
class FunctionProto:
    name: str
    arity: int
    chunk: Chunk
    upvalue_count: int
```

The top-level script is compiled as a `FunctionProto` named `"<script>"` with `arity=0`.

### `Local`

```python
@dataclass(slots=True)
class Local:
    name: str
    depth: int         # scope depth (0 = global scope, 1+ = block scope)
    is_captured: bool  # True if referenced by an inner closure
```

### `Compiler`

Recursive-descent visitor over the AST. Each `Stmt` and `Expr` node type has a corresponding `_compile_*()` method.

**Scope handling:** `_begin_scope()` / `_end_scope()` push/pop local variable blocks. `_end_scope()` emits `CLOSE_UPVALUE` for any captured locals and `POP` for others.

**Upvalues (closures):** `_resolve_upvalue()` walks outer `Compiler` instances to find captured variables. Captured locals become `Cell` objects in the VM, allowing mutation across closures.

**To add compilation for a new statement** (e.g. `ForRange`):

In `_compile_statement()`:
```python
elif isinstance(stmt, ForRange):
    self._compile_for_range(stmt)
```

```python
def _compile_for_range(self, stmt: ForRange) -> None:
    # Compile iterable expression, store in local, loop with INDEX
    # ... emit GET_LOCAL, CONSTANT (index), INDEX, SET_LOCAL, body, LOOP
```

---

## VM — `vm.py`

### Runtime Value Types

Lumen values are Python objects on the `VM.stack`:

| Lumen type | Python type |
|---|---|
| number | `float` |
| string | `str` |
| bool | `bool` |
| null | `None` |
| list | `list` |
| dict | `dict` |
| function | `Closure` |
| native | `Native` |

### `VM` Class

```python
class VM:
    stack: list[object]
    frames: list[Frame]
    globals: dict[str, object]
    open_upvalues: dict[int, Cell]   # stack index → Cell for open (non-closed) upvalues
```

`trace=True` prints every instruction as it executes — useful for debugging.

### `Frame`

```python
@dataclass(slots=True)
class Frame:
    closure: Closure
    ip: int       # instruction pointer into closure.proto.chunk.code
    slots: int    # base index in stack where this frame's locals start
```

### Upvalue Mechanism

When a closure is created (`Op.CLOSURE`), upvalues are captured:
- **Local in enclosing scope:** `Cell` is created and stored in `open_upvalues[stack_index]`; shared between the closure and the stack slot.
- **Non-local (from outer closure):** The upvalue is copied from the enclosing closure's upvalue list.

When a local goes out of scope (`Op.CLOSE_UPVALUE`), open upvalues at that stack index are "closed" — their value is moved from the stack into the `Cell` so the closure can still access it.

### Adding a Native Function

```python
vm.globals["sqrt"] = Native("sqrt", 1, lambda x: x ** 0.5)
```

Or at VM initialization:
```python
self.globals["sqrt"] = Native("sqrt", 1, lambda x: x ** 0.5)
```

`Native.arity = None` means any argument count is accepted. The function receives arguments as positional Python args.

**To add native list methods** (e.g. `push`, `pop`):

The current `INDEX` opcode handles `list[i]` and `dict[key]` access. Method calls require either:
1. A `GetProperty` opcode that dispatches on value type, or
2. Wrapping lists as custom objects with a method dispatch table.

The simpler path: add native functions that take the list as the first argument:
```python
vm.globals["push"] = Native("push", 2, lambda lst, val: lst.append(val) or None)
vm.globals["pop"] = Native("pop", 1, lambda lst: lst.pop())
```

### Adding a New Opcode Handler

In `VM.run()`, the dispatch is a long `if/elif` chain. Add a new branch:

```python
elif op is Op.POWER:
    b = self.stack.pop()
    a = self.stack.pop()
    if not isinstance(a, float) or not isinstance(b, float):
        raise VMError("Operands to ** must be numbers.")
    self.stack.append(float(a ** b))
```

---

## REPL — `cli.py`

The REPL shares a single `VM` instance across inputs, so globals persist between lines:

```python
vm = VM()
while True:
    source = input("lumen> ")
    result = execute(source, vm=vm)
```

The `execute()` function in `runtime.py` accepts an optional `vm` argument. Passing the same instance preserves state.

**To add REPL commands** (e.g. `:dis` to disassemble):

```python
if line.startswith(":dis "):
    source = line[5:]
    print(disassemble_source(source))
    continue
```

---

## Common Modifications — Quick Reference

| What you want to do | Where to start |
|---|---|
| Add a new keyword | `token.py` `KEYWORDS` dict + `TokenType` enum |
| Add a new operator | `lexer.py` `_scan_token()` + `TokenType` + parser Pratt table + `compiler.py` + `vm.py` |
| Add a new string escape | `lexer.py` `_string()` method |
| Add a new AST node | `ast_nodes.py` — dataclass subclassing `Expr` or `Stmt` |
| Add a new statement (parsing) | `parser.py` `_statement()` + new `_parse_*()` method |
| Add a new opcode | `bytecode.py` `Op` enum + `compiler.py` emit + `vm.py` handler |
| Add a native function | `VM.globals[name] = Native(name, arity, fn)` |
| Add variadic native | `Native(name, arity=None, function=fn)` |
| Enable VM execution tracing | `VM(trace=True)` or `execute(source, trace=True)` |
| Disassemble bytecode | `disassemble_source(source)` or `disassemble(chunk, name)` |
| Share VM state across calls | Pass same `vm=` instance to `execute()` |
| Embed the language | `from lumen.runtime import execute; execute(source)` |

---

*Built by MainbyteLabs — [github.com/MR-MainbyteLabs](https://github.com/MR-MainbyteLabs)*
