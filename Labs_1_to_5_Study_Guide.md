# CSE 4102 — Labs 1–5 Study Guide

Repository: `https://github.com/DewanNafis/CSE4102.git`

The ZIP archives have been extracted into `CSE4102/Lab1` through `CSE4102/Lab5`.

## 0. The big picture

These labs build a tiny compiler in stages:

```text
source text
   ↓
preprocessor / lexical analyzer (Flex)
   ↓ tokens
syntax analyzer (Bison parser)
   ↓
semantic analysis + symbol table
   ↓
intermediate representation (IR)
   ↓
assembly code generation
   ↓
machine code
```

| Lab | Main idea | Important files |
|---|---|---|
| 1 | GCC compilation phases; tokenization with Flex | `hello.c`, `hello.i`, `hello.s`, `hello.o`, `hello.dump`, `cal.l` |
| 2 | Context-free grammar and parsing with Bison | `cal.l/.y`, `prog.l/.y` |
| 3 | Semantic analysis, symbol table, and type checking | `lexer.l`, `parser.y`, `symtab.c/.h` |
| 4 | Stack-style intermediate code generation | Lab 3 files plus `codeGen.c/.h` |
| 5 | x86/MASM basics and IR-to-assembly translation | `.asm` examples and `IR_and_CG` |

Generated files such as `lex.yy.c`, `parser.tab.c`, and `*.tab.h` are tool output. Read the `.l`, `.y`, and handwritten `.c/.h` files first.

---

# Lab 1 — Compilation phases and lexical analysis

## Task 1: Observe GCC’s stages

`hello.c` declares `a`, `b`, and `c`, computes `c = a + b`, and prints `a` (10), not `c` (30).

The Makefile runs:

```make
# 1. Preprocess
gcc -E hello.c > hello.i

# 2. Compile C into assembly
gcc -S -masm=intel hello.i

# 3. Assemble into relocatable machine code
as -o hello.o hello.s

# 4. Disassemble the object file
objdump -M intel -d hello.o > hello.dump
```

### What each artifact means

- `hello.c`: original C source.
- `hello.i`: preprocessed C. `#include <stdio.h>` has been expanded and macros handled, so it is much larger.
- `hello.s`: human-readable assembly. The submitted file is 64-bit Intel-syntax assembly produced in a MinGW/Windows environment.
- `hello.o`: relocatable object code. It contains machine instructions and unresolved relocation/symbol information; it is not yet a complete executable.
- `hello.dump`: `objdump`’s readable disassembly of the bytes in `hello.o`.
- Missing final stage: linking. Normally `gcc hello.o -o hello` combines the object with startup code and libraries.

### Relate C to assembly

In `main`:

```asm
mov DWORD PTR -4[rbp], 10     ; a = 10
mov DWORD PTR -8[rbp], 20     ; b = 20
mov edx, DWORD PTR -4[rbp]    ; load a
mov eax, DWORD PTR -8[rbp]    ; load b
add eax, edx                  ; a + b
mov DWORD PTR -12[rbp], eax   ; c = result
```

`rbp` is being used as a stack-frame reference. Negative offsets hold local variables. `eax` and `edx` are 32-bit register portions, suitable for C `int` values.

Assembly text and disassembly differ in purpose:

- Assembly (`hello.s`) is input to an assembler and includes directives, labels, and symbolic references.
- Disassembly (`hello.dump`) reconstructs instructions from object-code bytes and shows addresses/opcodes such as `c7 45 fc ...`.

## Task 2: Flex lexical analyzer

A lexer groups characters into **lexemes** and assigns **token classes**.

Flex file layout:

```lex
%option noyywrap
%{
/* C declarations */
%}
definitions
%%
rules
%%
/* C functions */
```

Definitions in `cal.l`:

```lex
delim  [ \t\n]
ws     {delim}+
digit  [0-9]
number {digit}+
letter [A-Za-z]
id     {letter}+
```

Important rules:

```lex
{ws}      { }                              /* ignore whitespace */
"if"      { printf("%s IF\n", yytext); }
{number}  { printf("%s NUM\n", yytext); }
{id}      { printf("%s ID\n", yytext); }
```

- `yytext` points to the matched lexeme.
- `yylex()` repeatedly finds matches.
- `%option noyywrap` means no separate `yywrap()` is required at end-of-input.
- Flex chooses the **longest match**; if lengths tie, the earlier rule wins. Therefore `if` becomes `IF`, while `iffy` becomes one longer `ID` token.

Input `if (a == 5)` becomes approximately:

```text
IF LP ID EQ NUM RP
```

### Practice

1. Add identifiers containing digits/underscores after the first character:
   ```lex
   id [A-Za-z_][A-Za-z0-9_]*
   ```
2. Add tokens for `-`, `*`, `/`, `<=`, and `>=`.
3. Add a final error rule:
   ```lex
   . { printf("Unknown: %s\n", yytext); }
   ```
4. Predict the tokens for `ifx + if == 12` before running it.

---

# Lab 2 — Parsing with Bison

A lexer answers, “What token is this?” A parser answers, “Does this token sequence have a valid grammatical structure?”

## Flex–Bison connection

1. `bison -d cal.y` creates `cal.tab.c` and `cal.tab.h`.
2. `cal.l` includes `cal.tab.h`, so lexer actions know token constants such as `NUM` and `ADD`.
3. `flex cal.l` creates `lex.yy.c`.
4. `yyparse()` requests tokens by calling `yylex()`.
5. On an invalid sequence, Bison calls `yyerror()`.

## Calculator grammar

```bison
cal: exp;

exp: exp ADD NUM
   | exp ADD ID
   | exp SUB NUM
   | exp SUB ID
   | ID
   | NUM
   ;
```

This accepts a starting `ID` or `NUM`, followed by any number of `+ ID`, `+ NUM`, `- ID`, or `- NUM` pieces. Thus:

```text
var + var + 5 - 7
```

is accepted. A derivation can be seen as:

```text
exp
⇒ exp SUB NUM
⇒ exp ADD NUM SUB NUM
⇒ exp ADD ID ADD NUM SUB NUM
⇒ ID ADD ID ADD NUM SUB NUM
```

This grammar validates form but does not calculate a numeric result because it has no semantic values/actions for arithmetic.

## Tiny programming-language grammar

`prog.l` recognizes keywords, identifiers, numbers, operators, braces, parentheses, assignment, and semicolon. `prog.y` recognizes:

- declaration with initialization: `int a = 6;`
- assignment: `a = a + 1;`
- `if (...) { ... }` with optional `else`
- `while (...) { ... }`
- nested statements

The recursive rule

```bison
statements: statements statement
          | /* empty */
          ;
```

means “zero or more statements.” The empty alternative is the recursion’s base case.

`tail: LB statements RB` groups a block. Because `statements` can contain `if_statement` or `while_statement`, nesting works naturally.

### Syntax versus meaning

Lab 2 may accept `b = b - 7;` even if `b` was never declared. That is grammatically valid but semantically questionable. Lab 3 adds that missing check.

### Practice

1. Decide whether these are accepted and explain why:
   - `a + 5 - b`
   - `+ a`
   - `int x = 3` (no semicolon)
   - `while (x > 0) { x = x - 1; }`
2. Add multiplication and division tokens/rules.
3. Extend declarations to permit `int x;` without initialization.
4. Add a `print(x);` statement.
5. Write a grammar for parentheses and normal arithmetic precedence:
   ```bison
   exp: exp ADD term | term;
   term: term MUL factor | factor;
   factor: NUM | ID | LP exp RP;
   ```

---

# Lab 3 — Semantic analysis and symbol tables

Parsing checks structure. Semantic analysis checks contextual rules such as:

- Was an identifier declared?
- Was it declared twice?
- Are operand and assignment types compatible?

## Passing values between lexer and parser

The Bison union is:

```bison
%union {
    char str_val[100];
    int int_val;
}
```

`ID` carries a string:

```bison
%token<str_val> ID
```

The lexer copies the lexeme into its semantic value:

```lex
{ID} { strcpy(yylval.str_val, yytext); return ID; }
```

Nonterminals such as `type`, `constant`, and `exp` carry integer type codes:

```bison
%type<int_val> type constant exp
```

In an action, `$1`, `$2`, etc. refer to right-hand-side values; `$$` is the left-hand-side result.

## Symbol table

Each linked-list node stores:

```c
char st_name[40];
int st_type;
struct list_t *next;
```

Type codes are `INT_TYPE`, `REAL_TYPE`, and `CHAR_TYPE`.

Core operations:

- `search(name)`: walk the list and return a matching node or `NULL`.
- `insert(name, type)`: reject duplicate declarations; otherwise add a node at the head.
- `idcheck(name)`: report use of undeclared identifiers.
- `gettype(name)`: retrieve the declared type.
- `typecheck(t1, t2)`: accept only equal types in this simplified implementation.

Insertion at the head is constant time, but search is O(n). A production compiler commonly uses a hash table and nested scope structures.

## Type propagation

```bison
constant: ICONST { $$ = INT_TYPE; }
        | FCONST { $$ = REAL_TYPE; }
        | CCONST { $$ = CHAR_TYPE; };
```

For `b + 6.9`, `b` produces `INT_TYPE` and `6.9` produces `REAL_TYPE`. The addition action invokes `typecheck(INT_TYPE, REAL_TYPE)`, which fails. The provided input therefore reports a type mismatch on line 4.

Assignment checks the expression type against the declared left side:

```bison
ID ASSIGN exp SEMI {
    if (idcheck($1) == 0) yyerror();
    else if (typecheck(gettype($1), $3) == 0) yyerror();
}
```

## Operator precedence

The declarations intend multiplication to bind more tightly than addition, and addition more tightly than comparison:

```bison
%left LT GT
%left ADDOP
%left MULOP
```

Later precedence declarations have higher precedence. However, the submitted grammar does not fully handle semantic values for multiplication/comparisons, and `SUBOP`, `DIVOP`, and `EQUOP` are declared but not properly integrated into expression productions/precedence.

### Practice

1. Test duplicate declaration: `int a; int a;`.
2. Test undeclared use: `a = 4;`.
3. Test valid types: `double x; double y; x = y + 2.5;`.
4. Fix initialized declarations so they both insert the ID and check initializer type.
5. Implement semantic actions for `-`, `*`, `/`, `<`, `>`, and `==`.
6. Advanced: add block scopes so an inner declaration can shadow an outer declaration.

---

# Lab 4 — Intermediate representation (IR)

Lab 4 changes the symbol table by adding an integer `address`. The first variable gets address 0, the next address 1, and so on. These are logical slots, not literal hardware addresses.

## Instruction representation

```c
enum code_ops {
    START, HALT, LD_INT, LD_VAR, STORE,
    SCAN_INT_VALUE, PRINT_INT_VALUE, ADD
};

struct instruction {
    enum code_ops op;
    int arg;
};
```

`gen_code(op, arg)` appends an instruction to the `code` array. `code_offset` is the next free position.

## Syntax-directed translation

Actions are attached directly to grammar reductions:

```bison
program: { gen_code(START, -1); }
         code
         { gen_code(HALT, -1); };

exp: ICONST { gen_code(LD_INT, $1); }
   | ID     { gen_code(LD_VAR, idcheck($1)); }
   | exp ADDOP exp { gen_code(ADD, -1); };
```

For:

```c
int a;
int b;
scan(a);
b = a + 10;
print(b);
```

addresses are `a → 0`, `b → 1`, and IR is:

```text
start -1
scan_int_value 0
ld_var 0
ld_int 10
add -1
store 1
print_int_value 1
halt -1
```

`-1` means “this opcode has no meaningful explicit argument.”

## Why this is stack-machine code

For `a + 10`:

1. `LD_VAR 0` pushes the value of `a`.
2. `LD_INT 10` pushes 10.
3. `ADD` pops two values and pushes their sum.
4. `STORE 1` stores the result into `b`.

Postfix order makes tree-shaped expressions easy to evaluate. For `a + b * 2`, expected IR is:

```text
LD_VAR a
LD_VAR b
LD_INT 2
MUL
ADD
```

### Practice

1. Hand-generate IR for `c = a + b + 5;`.
2. Add opcodes `SUB`, `MUL`, and `DIV`.
3. Add grammar actions for those opcodes.
4. Add `print(25);` or `print(a + 2);`, not just `print(ID)`.
5. Replace the fixed `code[999]` array with bounds checking or a dynamic array.

---

# Lab 5 — Assembly and IR-to-code generation

This lab contains two parts:

- `Assembly_Code`: handwritten 32-bit MASM examples.
- `IR_and_CG`: the Lab 4 front end plus automatic assembly emission.

The assembly is Windows-specific: `.686`, MASM syntax, `INVOKE`, and `C:\masm32` include/library paths.

## Assembly examples

### `prog1.asm`: input/output

Declares format strings and a 32-bit signed variable:

```asm
input_integer_format byte "%d",0
output_integer_msg_format byte "%d", 0Ah, 0
number sdword ?
```

Then invokes C runtime functions:

```asm
INVOKE scanf, ADDR input_integer_format, ADDR number
INVOKE printf, ADDR output_integer_msg_format, number
```

`ADDR number` passes an address to `scanf`; `number` passes the stored value to `printf`.

### `prog2.asm`: conditional branch

```asm
mov eax, number
cmp eax, 0
jge POSITIVE
jl NEGATIVE
```

`cmp` sets CPU flags as if subtraction occurred. `jge` and `jl` are signed comparisons. Zero is classified as non-negative/“positive” by this program.

### `prog3.asm`: loop

```asm
mov ecx, number
LOOP_:
    cmp ecx, 0
    jle EXIT_
    ; print ecx
    sub ecx, 1
    jmp LOOP_
```

This counts down from the input to 1.

### `prog4.asm`: manual translation of tiny source

It corresponds to declarations, scan, addition by 10, store, and print. Variables use frame-relative slots, while `ebx` tracks a temporary evaluation stack.

## Automatic code generator

Lab 5 retains the IR but `print_assembly()` maps each opcode to MASM instructions.

### Variable slots

For logical address `k`, the generator uses an offset based on `4*k`:

```asm
mov eax, [ebp-...]
mov dword ptr [ebp-...], eax
```

Each integer occupies four bytes.

### Evaluation stack

`ebx` points to the next temporary slot. Loading a value performs:

```asm
mov eax, value
mov [ebx], eax
add ebx, 4
```

Addition performs:

```asm
sub ebx, 4
mov eax, [ebx]      ; right operand
sub ebx, 4
mov edx, [ebx]      ; left operand
add eax, edx
mov [ebx], eax      ; push result into left slot
add ebx, 4
```

Net stack effect: two operands become one result.

### Function prologue/epilogue

The intended structure is:

```asm
push ebp
mov ebp, esp
; reserve local storage
...
mov esp, ebp
pop ebp
ret
```

The submitted generator uses `sub ebp, 100`, which is unconventional and unsafe because stack allocation normally changes `esp`, not `ebp`. A conventional version would use `sub esp, 100` while keeping `ebp` stable.

### Practice

1. Trace `ebx`, `eax`, and stack values for `a + 10`.
2. Implement assembly templates for `SUB`, `MUL` (`imul`), and signed division (`cdq`, `idiv`).
3. Generate labels for `if` and `while`, e.g. `L0`, `L1`.
4. Add IR branch operations such as `JZ`, `JMP`, `LT`, and `LABEL`.
5. Port the generated code to Linux/NASM or GNU assembler syntax.

---

# Important problems and limitations in the submitted material

These files are useful teaching examples, but they are incomplete and environment-dependent.

1. **Windows command convention:** Makefiles run `a`, which is common for `a.exe` on Windows. Linux normally needs `./a.out` and often `gcc ... -o parser`.
2. **Windows assembly:** Lab 5 requires 32-bit MASM32 paths and will not directly assemble on a normal Linux setup.
3. **Generated files are stale/environment-specific:** regenerate `lex.yy.c` and `*.tab.c/.h` from `.l/.y` rather than editing generated code.
4. **Newline rule likely wrong in Labs 3–5:** `"\n"` in a Flex file matches a literal backslash+n rather than an actual newline. Prefer `\n { lineno++; }` without quotes.
5. **Missing default return from lexer:** unknown characters call `yyerror`, but error handling and function prototypes are old-style/inconsistent.
6. **Incomplete operator support:** several tokens are declared but have no full grammar, semantic, IR, or assembly implementation.
7. **Lab 3 initialized declaration is incomplete:** `type ID ASSIGN exp SEMI` has no insertion/type-checking action.
8. **Relational expression types are not properly assigned:** comparison expressions should normally yield a boolean/integer type.
9. **No scopes:** one global linked list means declarations inside blocks are not scoped.
10. **Fixed buffers/arrays:** IDs use fixed 100/40-byte arrays and IR has a fixed 999-instruction capacity.
11. **Including `.c` files:** `parser.y` includes `symtab.c`/`codeGen.c` directly. Better practice is including headers and compiling/linking separate `.c` files.
12. **Assembly frame bugs/oddities:** offsets such as `[ebp-0]`, use of `[ebp+4]`, and modifying `ebp` for allocation are suspicious. Treat the generator as a demonstration, not production-quality code.
13. **`printf("\% %d", ...)` style in generator:** this relies on awkward formatting to print `%d`; using `%%d` in the C format string is clearer and correct.
14. **No optimizer:** IR is translated directly, so redundant loads/stores remain.

---

# How to build and practice on Linux/WSL

Install tools:

```bash
sudo apt update
sudo apt install build-essential flex bison
```

For a Flex-only lab:

```bash
flex cal.l
gcc -Wall -Wextra lex.yy.c -o lexer
./lexer < input.txt
```

For Flex + Bison:

```bash
bison -Wall -d parser.y
flex lexer.l
gcc -Wall -Wextra parser.tab.c lex.yy.c -o compiler
./compiler < input.txt
```

If handwritten modules are changed to use headers rather than `#include "symtab.c"`, compile with:

```bash
gcc -Wall -Wextra parser.tab.c lex.yy.c symtab.c codeGen.c -o compiler
```

Lab 1’s provided `hello.s`/`hello.o` came from MinGW, so regenerate them locally if your platform is Linux:

```bash
gcc -E hello.c -o hello.i
gcc -S -masm=intel hello.c -o hello.s
gcc -c hello.c -o hello.o
objdump -M intel -d hello.o > hello.dump
gcc hello.o -o hello
./hello
```

---

# A structured practice plan

## Session 1 — Tokens

- Write regexes for identifiers, integers, floating constants, comments, and operators.
- Predict token streams by hand.
- Modify Lab 1 lexer and test malformed characters.

## Session 2 — Grammar

- Derive several Lab 2 inputs manually.
- Distinguish terminals, nonterminals, productions, start symbol, and empty production.
- Add parentheses and precedence.

## Session 3 — Semantics

- Draw the symbol-table linked list after each declaration.
- Test undeclared IDs, duplicates, and mismatched assignments.
- Repair initialized declarations and all arithmetic type actions.

## Session 4 — IR

- Convert expressions from infix to postfix/stack IR by hand.
- Track stack depth after every instruction.
- Add subtraction and multiplication end to end.

## Session 5 — Assembly

- Trace registers and flags in the four assembly programs.
- Map every IR opcode to an assembly template.
- Repair the stack frame, then add branches/labels.

## Mini-project

Extend the tiny language to compile:

```c
int a;
int b;
scan(a);
b = a * 2 + 10;
if (b > 20) {
    print(b);
}
```

Required steps:

1. Lexer tokens for `*`, `>`, `if`, braces.
2. Grammar with correct precedence.
3. Declaration and type checks.
4. IR operations `MUL`, `GT`, conditional jump, label.
5. Assembly templates and unique label generation.

If you can explain and implement that pipeline, you understand the central lesson of Labs 1–5.

---

# Quick self-test

1. Why is `hello.i` much larger than `hello.c`?
2. What is the difference between a lexeme and a token?
3. Why must the lexer include Bison’s generated header?
4. What does an empty grammar production accomplish?
5. Why can Lab 2 accept an undeclared variable?
6. What do `$1`, `$3`, and `$$` mean in Bison actions?
7. What information is stored in the Lab 3 symbol table?
8. Why does `int + double` fail in the submitted type checker?
9. Why does `a + 10` emit loads before `ADD`?
10. What is the stack effect of `ADD`?
11. What is the difference between a logical symbol-table address and a hardware address?
12. Which CPU flags/branches implement signed comparison?
13. Why should `esp`, rather than `ebp`, normally be changed to reserve stack space?
14. What additional IR instructions are needed for an `if` statement?

Try answering without looking back, then verify each answer in the relevant section.
