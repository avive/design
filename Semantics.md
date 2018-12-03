# Semantics

This document explains the high-level design of WebAssembly code: its types, constructs, and
semantics.
WebAssembly code can be considered a *structured stack machine*; a machine where most computations use a stack
of values, but control flow is expressed in structured constructs such as blocks, ifs, and loops.
In practice, implementations need not maintain an actual value stack, nor actual data structures for control; they
need only behave [as if](https://en.wikipedia.org/wiki/As-if_rule) they did so.
For full details consult [the formal Specification](https://github.com/WebAssembly/spec),
for file-level encoding details consult [Binary Encoding](BinaryEncoding.md),
and for the human-readable text representation consult [Text Format](TextFormat.md).

Each function body consists of a list of instructions which forms an implicit *block*.
Execution of instructions proceeds by way of a traditional *program counter* that advances
through the instructions.
Instructions fall into two categories: *control* instructions that form control constructs and *simple* instructions.
Control instructions pop their argument value(s) off the stack, may change the
program counter, and push result value(s) onto the stack.
Simple instructions pop their argument value(s) from the stack, apply an operator to the values,
and then push the result value(s) onto the stack, followed by an implicit advancement of
the program counter.

All instructions and operators in WebAssembly are explicitly typed, with no overloading rules.
Verification of WebAssembly code requires only a single pass with constant-time
type checking and well-formedness checking.

WebAssembly offers a set of language-independent operators that closely
match operators in many programming languages and are efficiently implementable
on all modern computers. Each operator has a corresponding simple instruction.

The [rationale](Rationale.md) document details why WebAssembly is designed as
detailed in this document.

[:unicorn:][future general] = Planned [future][future general] feature

## Types

WebAssembly has the following *value types*:

  * `i32`: 32-bit integer
  * `i64`: 64-bit integer
  * `f32`: 32-bit floating point
  * `f64`: 64-bit floating point

Each parameter and local variable has exactly one [value type](Semantics.md#types). Function signatures
consist of a sequence of zero or more parameter types and a sequence of zero or more return
types. (Note: in the MVP, a function can have at most one return type).

Note that the value types `i32` and `i64` are not inherently signed or
unsigned. The interpretation of these types is determined by individual
operators.

## Local variables

Each function has a fixed, pre-declared number of *local variables* which occupy a single
index space local to the function. Parameters are addressed as local variables. Local
variables do not have addresses and are not aliased by linear memory. Local
variables have [value types](#types) and are initialized to the appropriate zero value for their
type (`0` for integers, `+0.` for floating-point) at the beginning of the function,
except parameters which are initialized to the values of the arguments passed to the function.

  * `get_local`: read the current value of a local variable
  * `set_local`: set the current value of a local variable
  * `tee_local`: like `set_local`, but also returns the set value

The details of index space for local variables and their types will be further clarified,
e.g. whether locals with type `i32` and `i64` must be contiguous and separate from
others, etc.

## Control constructs and instructions

WebAssembly offers basic structured control flow constructs such as *blocks*, *loops*, and *ifs*.
All constructs are formed out of the following control instructions:

 * `nop`: no operation, no effect
 * `block`: the beginning of a block construct, a sequence of instructions with a label at the end
 * `loop`: a block with a label at the beginning which may be used to form loops
 * `if`: the beginning of an if construct with an implicit *then* block
 * `else`: marks the else block of an if
 * `br`: branch to a given label in an enclosing construct
 * `br_if`: conditionally branch to a given label in an enclosing construct
 * `br_table`: a jump table which jumps to a label in an enclosing construct
 * `return`: return zero or more values from this function
 * `end`: an instruction that marks the end of a block, loop, if, or function

Blocks are composed of matched pairs of `block` ... `end` instructions, loops with matched pairs of 
`loop` ... `end` instructions, and ifs with either `if` ... `end` or `if` ... `else` ... `end` sequences.
For each of these constructs the instructions in the ellipsis are said to be *enclosed* in the
construct.

### Branches and nesting

The `br`, `br_if`, and `br_table` instructions express low-level branching and are hereafter refered to simply as branches.
Branches may only reference labels defined by a construct in which they are enclosed.
For example, references to a `block`'s label can only occur within the `block`'s body.

In practice, outer `block`s can be used to place labels for any given branching
pattern, except that the nesting restriction makes it impossible to branch into the middle of a loop
from outside the loop. This limitation ensures by construction that all control flow graphs
are well-structured as in high-level languages like Java, JavaScript and Go.
Notice that a branch to a `block`'s label is  equivalent to a labeled `break` in
high-level languages; branches simply break out of a `block`, and branches to a `loop`
correspond to a "continue" statement.

### Execution semantics of control instructions

Executing a `return` pops return value(s) off the stack and returns from the current function.

Executing a `block` or `loop` instruction has no effect on the value stack.

Executing the `end` of a `block` or `loop` (including implicit blocks such as in `if` or for a function body) has no effect on the value stack.

Executing the `end` of the implicit block for a function body pops the return value(s) (if any) off the stack and returns from the function.

Executing the `if` instruction pops an `i32` condition off the stack and either falls through to the next instruction
or sets the program counter to after the `else` or `end` of the `if`.

Executing the `else` instruction of an `if` sets the program counter to after the corresponding `end` of the `if`.

Branches that exit a `block` or `if` may yield value(s) for that construct.
Branches pop result value(s) off the stack which must be the same type as the declared
type of the construct which they target. If a conditional or unconditional branch is taken, the values pushed
onto the stack between the beginning of the construct and the branch are discarded, the result value(s) are
pushed back onto the stack, and the program counter is updated to the end of the construct. 

Branches that target a `loop` do not yield a value; they pop any values pushed onto the stack since the start of the loop and set the program counter to the start of the loop.

The `drop` operator can be used to explicitly pop a value from the stack.

The implicit popping associated with explicit branches makes compiling expression languages straightforward, even non-local
control-flow transfer, requiring fewer drops.

Note that in the MVP, all control constructs and control instructions, including `return` are
restricted to at most one value.

## Calls

Each function has a *signature*, which consists of:

  * Return types, which are a sequence of value types
  * Argument types, which are a sequence of value types

WebAssembly doesn't support variable-length argument lists (aka
varargs). Compilers targeting WebAssembly can instead support them through
explicit accesses to linear memory.

In the MVP, the length of the return types sequence may only be 0 or 1. This
restriction may be lifted in the future.

Direct calls to a function specify the callee by an index into the 
[function index space](Modules.md#function-index-space).

  * `call`: call function directly

A direct call to a function with a mismatched signature is a module verification error.

Indirect calls to a function indicate the callee with an `i32` index into
a [table](#table). The *expected* signature of the target function (specified
by its index in the [types section](BinaryEncoding.md#type-section)) is given as
a second immediate.

  * `call_indirect`: call function indirectly

Unlike `call`, which checks that the caller and callee signatures match
statically as part of validation, `call_indirect` checks for signature match
*dynamically*, comparing the caller's expected signature with the callee function's
signature and and trapping if there is a mismatch. Since the callee may be in a
different module which necessarily has a separate [types section](BinaryEncoding.md#type-section),
and thus index space of types, the signature match must compare the underlying 
[`func_type`](https://github.com/WebAssembly/spec/blob/master/interpreter/spec/types.ml#L5).
As noted [above](#table), table elements may also be host-environment-defined
values in which case the meaning of a call (and how the signature is checked)
is defined by the host-environment, much like calling an import.

In the MVP, the single `call_indirect` operator accesses the [default table](#table).

Multiple return value calls will be possible, though possibly not in the
MVP. The details of multiple-return-value calls needs clarification. Calling a
function that returns multiple values will likely have to be a statement that
specifies multiple local variables to which to assign the corresponding return
values.

## Constants

These operators have an immediate operand of their associated type which is
produced as their result value. All possible values of all types are
supported (including NaN values of all possible bit patterns).

  * `i32.const`: produce the value of an i32 immediate
  * `i64.const`: produce the value of an i64 immediate
  * `f32.const`: produce the value of an f32 immediate
  * `f64.const`: produce the value of an f64 immediate

## 32-bit Integer operators

Integer operators are signed, unsigned, or sign-agnostic. Signed operators
use two's complement signed integer representation.

Signed and unsigned operators trap whenever the result cannot be represented
in the result type. This includes division and remainder by zero, and signed
division overflow (`INT32_MIN / -1`). Signed remainder with a non-zero
denominator always returns the correct value, even when the corresponding
division would trap. Sign-agnostic operators silently wrap overflowing
results into the result type.

  * `i32.add`: sign-agnostic addition
  * `i32.sub`: sign-agnostic subtraction
  * `i32.mul`: sign-agnostic multiplication (lower 32-bits)
  * `i32.div_s`: signed division (result is truncated toward zero)
  * `i32.div_u`: unsigned division (result is [floored](https://en.wikipedia.org/wiki/Floor_and_ceiling_functions))
  * `i32.rem_s`: signed remainder (result has the sign of the dividend)
  * `i32.rem_u`: unsigned remainder
  * `i32.and`: sign-agnostic bitwise and
  * `i32.or`: sign-agnostic bitwise inclusive or
  * `i32.xor`: sign-agnostic bitwise exclusive or
  * `i32.shl`: sign-agnostic shift left
  * `i32.shr_u`: zero-replicating (logical) shift right
  * `i32.shr_s`: sign-replicating (arithmetic) shift right
  * `i32.rotl`: sign-agnostic rotate left
  * `i32.rotr`: sign-agnostic rotate right
  * `i32.eq`: sign-agnostic compare equal
  * `i32.ne`: sign-agnostic compare unequal
  * `i32.lt_s`: signed less than
  * `i32.le_s`: signed less than or equal
  * `i32.lt_u`: unsigned less than
  * `i32.le_u`: unsigned less than or equal
  * `i32.gt_s`: signed greater than
  * `i32.ge_s`: signed greater than or equal
  * `i32.gt_u`: unsigned greater than
  * `i32.ge_u`: unsigned greater than or equal
  * `i32.clz`: sign-agnostic count leading zero bits (All zero bits are considered leading if the value is zero)
  * `i32.ctz`: sign-agnostic count trailing zero bits (All zero bits are considered trailing if the value is zero)
  * `i32.popcnt`: sign-agnostic count number of one bits
  * `i32.eqz`: compare equal to zero (return 1 if operand is zero, 0 otherwise)

Shifts counts are wrapped to be less than the log-base-2 of the number of bits
in the value to be shifted, as an unsigned quantity. For example, in a 32-bit
shift, only the least 5 significant bits of the count affect the result. In a
64-bit shift, only the least 6 significant bits of the count affect the result.

Rotate counts are treated as unsigned.  A count value greater than or equal
to the number of bits in the value to be rotated yields the same result as
if the count was wrapped to its least significant N bits, where N is 5 for
an i32 value or 6 for an i64 value.

All comparison operators yield 32-bit integer results with `1` representing
`true` and `0` representing `false`.

## 64-bit integer operators

The same operators are available on 64-bit integers as the those available for
32-bit integers.
