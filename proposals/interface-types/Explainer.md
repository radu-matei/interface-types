# Interface Types Proposal

The proposal adds a new set of **interface types** to WebAssembly that describe
high-level values (like strings, arrays, records and variants) along with a new
set of instructions that produce and consume interface types which are designed
to be highly optimizable. The proposal is semantically layered on top of the
WebAssembly [core spec] and thus can be implemented in terms of an unmodified
core WebAssembly implementation. This new layer wraps WebAssembly's *core
modules* with a new *kind* of WebAssembly module called a **component**.
Components provide the WebAssembly ecosystem a highly embeddable, composable,
virtualizable and language-neutral unit of code reuse and distribution.

1. [Problem](#problem)
2. [Use Cases](#use-cases)
3. [Additional Requirements](#additional-requirements)
4. [Proposal](#proposal)
   1. [Interface Types](#interface-types)
   2. [Lifting and Lowering Instructions](#lifting-and-lowering-instructions)
   3. [Adapter Functions](#adapter-functions)
   4. [Components](#components)
   5. [Lazy Lifting: Avoiding Intermediate Copies by Design](#lazy-lifting-avoiding-intermediate-copies-by-design)
   6. [Ownership Instructions: Reliably Releasing Resources In the Presence of Laziness](#ownership-instructions-reliably-releasing-resources-in-the-presence-of-laziness)
   7. [Exception and Trap Safety](#exception-and-trap-safety)
5. [Use Cases Revisited](#use-cases-revisited)
6. [Additional Requirements Revisited](#additional-requirements-revisited)
7. [FAQ](#FAQ)
8. [TODO](#TODO)


## Problem

As a compiler target, WebAssembly provides only low-level types that aim to be
as close to the underlying machine as possible allowing each source language to
implement its own high-level types efficiently on top of the low-level types.
However, when modules from multiple languages need to communicate with each
other or the host, this raises the question of how to exchange high-level
values at the interface.
 
The current best approach for dealing with this problem is to generate
JavaScript *glue code* that uses the powerful [JS API] to convert between
values at interface boundaries. However, the JavaScript glue code approach has
several limitations:
* It depends on the host including a JavaScript implementation, which not all
  hosts do or can.
* JavaScript glue code incurs unnecessary overhead due to copying, transition
  costs and loss of static type information.

Ideally, a solution to this problem would only depends on language-independent
WebAssembly standards or conventions.


## Use Cases

To help motivate the proposed solution, we consider 3 use cases. After the
proposal is introduced, the use cases are [revisited below](#use-cases-revisited).


### Optimizing calls to Web APIs

A long-standing source of friction between WebAssembly and the rest of the
Web platform (and the original motivating use case for this proposal) is that
calling Web APIs from WebAssembly usually requires thunking through JavaScript
which hurts performance and adds extra complexity for developers to think
about. Technically, since they are exported as JavaScript functions, Web
IDL-defined functions can be directly imported and called by WebAssembly
through the [JS API]. However, Web IDL functions usually take and produce
high-level Web IDL types like [`DOMString`] and [Sequence] while WebAssembly
can only pass numbers.

With Interface Types, ideally WebAssembly could create Web IDL-compatible
high-level types and pass them directly to statically-typed browser entry
points without transit through JS.


### Defining language-neutral interfaces like WASI

While many [WASI] signatures can be expressed in terms of `i32`s and,
in the future, references to [type imports][Type Import], there is still a need
for WASI functions to take and return higher-level value types like strings or
arrays. Currently, these values are specified in terms of linear memory. For
example, in [`path_open`], a string is passed as a pair of `i32`s which
represent the offset and length, respectively, of the string in linear memory.

However, specifying this single, fixed representation for data types like
strings will become problematic if:
* WebAssembly gets access to GC memory with the [GC] proposal and the caller
  wants to pass a `ref array u8`;
* the host has a native string type which is not the same as the
  GC `ref array u8` (e.g., a JS/DOM string) which WASI would ideally accept
  without copy; or
* the need arises to allow more than just UTF-8 encoding.

Ideally, WASI APIs would be able to use simple high-level types that could be
constructed from a variety of source representations.

There's also another, more subtle problem with passing `i32`s: they implicitly
refer to the caller's memory. Currently, this requires that all WASI client
modules must export their memory with a designated name (`"memory"`) and it
also requires special tricks for WASI implementations to be able to find
this memory. Ideally, WASI APIs would simply take values without the client
or the implementation needing to expose its memory at all.


### Enabling Shared-Nothing Linking of WebAssembly Modules

While WebAssembly intentionally supports [native dynamic linking]
(a.k.a. "shared-everything" linking; cf. [shared-everything dynamic linking in
the Module Linking proposal][shared-everything-example]), this type of linking
is more fragile:
* Modules must carefully coordinate (at the toolchain and source level) to
  share data and generally avoid clobbering each other.
* Modules must agree on versions for common shared libraries since the
  libraries' state is generally shared by all clients.
* Corruption in one module can affect other modules, with certain bugs
  only manifesting with certain combinations of modules.
* Modules are less able to encapsulate, leading to unwanted representation
  dependencies or violations of the Principle of Least Authority.

In contrast, "shared-nothing" linking refers to a linking scheme in which each
module defines and encapsulates its own memories and tables and, in doing so,
ensures a degree of isolation that mitigates the sources of fragility listed
above. However, there is currently no host-independent way to implement even
basic shared-nothing linking operations like copying an array of bytes from one
module's linear memory to another.

With interface types available to use at the boundary between two
modules, exports and imports can take, e.g., an abstract sequence of bytes,
or a string, or a sequence of pairs of strings and integers, etc. It is then
the wasm engine that takes care of copying data between two modules' linear
memories, allowing all modules to fully encapsulate their state.


### Scoping Shared-Everything Linking

Shared-nothing linking should not, however, be mutually exclusive with
shared-everything linking: just because two modules don't share state doesn't
mean they can't share code. Indeed, the [Module Linking] proposal [explicitly
supports][shared-everything-use-case] having multiple wasm programs that share
modules, but not memories, but there is no provision for how those independent
programs can communicate with each other. While they could communicate
indirectly through a pipe/channel/socket/message API, with Interface Types,
they should additionally be able to be communicate efficiently through
synchronous function calls.

TODO:
* show diagram that shows two-level topology as desired end-state
  (using "component")


## Additional Requirements

TODO:
* Avoid baking in the assumption of linear memory or GC and allow
  zero-copy when both sides use a compatible, immutable, identity-less
  GC references.
* Allow efficient conversion to and from host values (e.g., JS strings)
  with zero copy possible when both sides agree on representation.
* Avoid committing to a single value representation for all time (e.g.,
  string encoding).
* Avoid unnecessary copies to temporaries and O(n) engine-internal
  allocations (not in linear or GC memory). Reducing copy overhead allows
  finer-granularity use of shared-nothing linking, which are, in turn, better
  for security and composability.
* Avoid reliance on a sufficiently-smart compiler to optimize effectively;
  optimization should be enabled by construction.
* Allow semver-compatible interface evolution with similar expressivity to
  what is enabled by ProtoBuf without runtime overhead.
* Avoid fragile ABIs that allow sloppy or out-of-date toolchains to manifest
  late or hard-to-debug errors when linked with well-behaved new toolchains.


## Proposal

The proposal is introduced bottom-up, starting with the new types, then the
instructions that produce/consume the new types, then the new functions that can
use the new instructions, then the new kind of module that can contain the new
functions.


### Interface Types

The wasm core spec defines the set [`valtype`] of low-level value types (`i32`,
`i64`, `f32`, `f64` and a growing set of [reference types], starting with
`externref` and `funcref`). The Interface Types proposal introduces a *new*
set `intertype` of *high-level* value types which are mostly disjoint from
`valtype`. The set `intertype` is inductively generated from the grammar below,
containing only types of finite nesting (unlike `valtype` which, starting with
the [function references] proposal, can contain cyclic function types).

```
intertype ::= f32 | f64
            | s8 | s16 | s32 | s64 | u8 | u16 | u32 | u64
            | string
            | (array <intertype>)
            | (record (field <name> <id>? <intertype>)*)
            | (variant (case <name> <id>? <intertype>*)*)
```

(Note: more types are planned for addition as listed in [TODO](#TODO).)

Also unlike `valtype`, where reference values refer to mutable state with
identity and have (non-coercive) subtyping constrained by low-level memory
layout requirements, `intertype` values are transitively immutable and
subtyping has coercion semantics that allow more flexible subtyping. Since
copying is essential to the use of interface types in the first place,
coercions can be folded into the copy at little, if any, extra cost. The type
constructors of `intertype` are summarized as follows:

| Type Constructor | Description of values | Subtyping allowed |
| ---------------- | --------------------- | ---------------- |
| `f32`, `f64` | same as core wasm | `f32 <: f64` |
| `s8`, `u8`, `s16`, `u16`, `s32`, `u32`, `s64`, `u64` | explicitly signed, ranged integer values | `I1 <: I2` if `I1`'s range is included in `I2`'s |
| `string` | a list of [Unicode scalar values] | |
| `array` | a list of homogeneous `intertype` values with eagerly-computed length | `(array T) <: (array T')` if `T <: T'` |
| `record` | a tuple of named heterogeneous values | `(record Tᵢ*) <: (record T'ᵢ)` if `Tᵢ <: T'ᵢ`, matched by name, allowing `T` to have extra, ignored fields |
| `variant` | a discriminated union of named heterogeneous tuples of values | `(variant Tᵢ*) <: (variant T'ᵢ)` if `Tᵢ <: T'ᵢ`, matched by name, allowing `T'` to have extra, ignored cases |

An example type using these type constructors together is:
```
(array
  (record
    (field "person"
      (record
        (field "name" string)
        (field "age" u8)))
    (field "mood"
      (variant
        (case "happy")
        (case "sad")))))
```
Should consumers of this type wish to start considering ages beyond 256 or moods
beyond these two, due to the flexible subtyping rules, consumers of this type
can backwards-compatibly switch from `u8` to `u16` and add cases to the variant.

While `variant` is sufficiently general to capture a number of more-specialized
types that often arise in programming languages, it can still be beneficial for
toolchains to have these specialized types represented explicitly as interface
types. Thus, three text-format and binary-format [abbreviations] are included.
As with the existing wasm text format, the abbreviation on the left-hand side of
the `≡` is expanded during parsing to the right-hand side. Analogous decoding
rules are added to binary format.
```
                             bool ≡ (variant (case "True") (case "False"))
             (option <intertype>) ≡ (variant (case "None") (case "Some" <intertype>))
(result <intertype> <intertype>?) ≡ (variant (case "Ok" <intertype>) (case "Error" <intertype>?))
```
Given these abbreviations, a toolchain generating source-level declarations for
an interface-typed API can generate the more-specialized (and often more
concise and efficient) source-language types corresponding to `bool`, `option`
and `result`.


### Lifting and Lowering Instructions

Along with each interface type, the proposal adds new instructions for producing 
and consuming the interface type. These instructions fall into two categories:
* **Lifting instructions** produce interface types, going from
  lower-level types to higher-level types.
* **Lowering instructions** consume interface types, going from higher-level
  types to lower-level types.

Lifting and lowering instructions thus act as the "bridge" between core wasm and
interface types and their instruction signatures include types from both
`valtype` and `intertype`. By having the conversion between `valtype` and
`intertype` explicit and programmable, this proposal allows for a wide gamut
of low-level value representations to be passed directly *in situ* (without the
wasm module performing an intermediate serialization step) and for this set of
options to grow over time.

As with core wasm instructions, lifting and lowering instructions can dynamically
fail and, when they do, they do so by trapping. In the future, alternative failure
options (like throwing an [exception]) may be added.

The following subsections list the initial proposed set of lifting and lowering
instructions, grouped by interface type.

#### Integers

Similar to the core wasm numeric conversions, a matrix of integral conversion
instructions are given which allow converting between the 2 sign-less core
wasm integer types and the 8 explicitly-signed integral interface types:
```
uX.from_iY : [iY] -> [uX]
sX.from_iY : [iY] -> [sX]
iY.from_uX : [uX] -> [iY]
iY.from_sX : [sX] -> [iY]
where
 - X is one of 8, 16, 32 or 64
 - Y is one of 32 or 64
```

If the destination bitwidth is smaller than the source bitwidth and the dynamic
value is outside of the destination's range (determined by the explicit
signedness of the instruction), the instruction traps.

For example, the following instruction sequence gives an explicit sign to an
otherwise-ambiguous `i32` with its high-bit set:
```wasm
i32.const 0x80000000  ;; push i32
u32.from_i32  ;; pop i32, push u32
```

#### Strings

Recalling that strings are lists of abstract [Unicode scalar values], all
string lifting and lowering instructions need to specify a concrete encoding.
For broad coverage across a wide set of languages and string representations,
the initial proposal contains two encodings: UTF-8 and UTF-16.

The three string lifting and lowering instructions and their signatures are:
```
string.lift_memory <encoding>? <memory>? : [i32, i32] -> [string]
string.size <encoding>? : [string] -> [i32]
string.lower_memory <encoding>? <memory>? : [i32, string] -> []
```
When absent:
* the default `<encoding>` is UTF-8
* the default `<memory>` is 0

The `string.lift_memory` produces a `string` by decoding it from the linear
memory identified by `memory` according to the [Encoding API] for `encoding`
with the [error mode] set to `fatal`. If decoding results in a failure (with
causes including the presence of surrogate code points), `string.lift_memory`
traps.

The `string.size` is used to query how much memory `string.lower_memory`
will require to encode the given `string` so that the containing code can
allocate enough.

For example, a C-style (null-terminated) string could be lifted via the
expression (using `let` as proposed in the [function references] proposal):
```wat
let (result string) (local $ptr i32)
  local.get $ptr  ;; push source address
  (call $strlen (local.get $ptr))  ;; push source length
  string.lift_memory  ;; read string from source
end
```

Conversely, a C-style string could be lowered via the expression:
```wat
let (result i32) (local $str string)
  (call $malloc_cstr (string.size (local.get $str)))  ;; push destination address
  local.get $str  ;; push source string
  string.lower_memory  ;; copy source into destination address
end
```
where `$malloc_cstr` is a function that wraps `$malloc` by adding 1 to the
requested byte count and initializing the final byte to null before return.

#### Records

Records are simply tuples of smaller interface values and thus can be produced
and consumed by simple stack operations:
```wasm
record.lift $R : [$F*] -> [$R]
record.lower $R : [$R] -> [$F*]
where
  - $R refers to a (record (field <name> $F)*)
```

For example, given a type definition of a record type:
```wasm
(type $Coord (record (field "x" s32) (field "y" s32)))
```
a `$Coord` value can be lifted from two `i32`s by the instruction sequence:
```wasm
(s32.from_i32 (i32.const 1))  ;; push s32
(s32.from_i32 (i32.const 2))  ;; push s32
record.lift $Coord  ;; pop two s32s, push record
```
and lowered back into two `i32`s by the instruction sequence:
```wasm
record.lower $Coord  ;; pop record, push two s32s
let (result i32 i32) (local $x s32) (local $y s32)  ;; pop s32s into locals
  (i32.from_s32 (local.get $x))  ;; push i32
  (i32.from_s32 (local.get $y))  ;; push i32
end
```

#### Arrays

The initial lifting and lowering instructions of arrays are oriented towards contiguous
allocations. In the future, more sophisticated instructions could be used for producing
and consuming arrays with other segmented representations.

The array lifting instruction's signature is:
```wasm
array.lift T* $A <instr>* end : [T*, i32] -> [$A]
where:
  - $A refers to an (array E)
  - <instr>* : [i32] -> [E]
```
This instruction is effectively a loop; given an `i32` pair `(begin, count)`, it
executes the `<instr>*` body `count` times, passing the body an `i32` that
starts at `begin` and is incremented by `stride` on each successive iteration.

Array lowering is similar to string lowering in that two instructions are used:
one to determine how much memory to allocate, and one to lower into the
already-allocated memory:
```wasm
array.count $A : [$A] -> [i32]
array.lower T* $A <instr>* end : [T*, $A] -> [T*]
where
  - $A refers to an (array E)
  - <instr>* : [T*, E] -> [T*]
```
As with lifting, `array.lower` is effectively a loop, executing the `<instr>*`
body `array.count` times, passing an `i32` that starts at initial `i32` passed in
and is incremented by `stride` on each successive iteration.

For example, given a type definition of an array type:
```wasm
(type $Coord (record (field "x" s32) (field "y" s32)))
(type $Points (array $Coord))
```
a `$Points` value can be lifted from a pointer to a contiguous array of `i32` pairs:
```wasm
array.lift $Points 8  ;; pop source address and length
  ;; i32 element pointer has been pushed on entry
  let (result $Coord) (local $elemPtr i32)
    s32.from_i32 (i32.load (local.get $elemPtr))
    s32.from_i32 (i32.load offset=4 (local.get $elemPtr))
    record.lift $Coord  ;; convert two s32s on the stack to the record element
  end
end  ;; on exit, a record element must have been pushed
```
and lowered back into the same:
```wasm
let (local $arr $Points)
  (call $malloc (i32.mul (array.count (local.get $arr)) (i32.const 8))  ;; alloc destination
  local.get $arr
  array.lower $Points 8  ;; pop destination pointer and array
    ;; i32 element pointer and element value have been pushed on entry
    let (local $elemPtr i32) (local $coord $Coord)
      (record.lower $Coord (local.get $coord))  ;; decompose element into fields
      (i32.store (local.get $elemPtr))
      (i32.store offset=4 (local.get $elemPtr))
    end
  end
end
```
With this design, implementations have a fair amount of flexibility determining
the representation of the low-level array being lifted or lowered. For example,
elements could be stored either in-line (with arbitrary padding and packing) or
out-of-line (as a owned or reference-counted pointer) or something in-between.
Moreover, the lifting and lowering don't have to have the same representation
since both meet at the `array` type in the middle.

#### Variants

Variants are the dual of records (records being [product types] and variants
being [sum types]). Like arrays, variant lifting and lowering needs to
perform control flow and thus variant instructions have nested body expressions
just like array instructions.

The initial lifting instruction of variants is oriented towards using a dense
`i32` index to discriminate the variant. Calls into core wasm can be used
before the lifting instruction to map arbitrary discrimant representations to a
dense `i32`. In the future, more sophisticated instructions could be added
that allow inline testing using arbitrary expressions. The variant lifting
instruction's signature is:
```wasm
variant.lift <valtype>* $V <instr>* end : [<valtype>*] -> [$V]
where
 - $V refers to a (variant (case <name> C*)*)
 - <instr>* : [<valtype>*] -> [C<sub>last</sub>]
```
The `i32` flowing into the `variant.lift` instruction is used to select which
`$case` to execute, with a value of `0` selecting the first case, `1` selecting
the second, etc, with the last `$case` being selected by default for all other
values. Since variant subtyping allows arbitrary reordering, a module is always
able to choose the appropriate default case for itself.

The variant lowering instruction's signature is:
```wasm
variant.lower <valtype1>* $V <valtype2>* [case <instr>* end]* : [<valtype1>* $V] -> [<valtype2>*]
where
 - $V refers to a (variant (case <name> C*)*)
 - the number of cases in $V and variant.lower's body is the same
 - for each case: <instr>* : [<valtype1>* C*] -> [<valtype2>*]
```
This instruction allows arbitrary values to flow into and out of the cases,
which enables multiple lowering implementation strategies: writing into
an already-allocated memory location that is passed in or producing new values
that flow out.

For example, given a type definition for variant that encodes an optional
coordinate argument:
```wasm
(type $Coord (record (field "x" s32) (field "y" s32)))
(type $MaybeCoord (variant (case "None") (case "Some" $Coord)))
```
a `$MaybeCoord` value can be lifted from a pointer that is either null (which
conveniently has the `i32` value `0`) or a pointer to a pair of `i32`s:
```wasm
variant.lift $MaybeCoord  ;; pops one i32
  case  ;; the "None" case if the i32=0 (the i32 is left on the stack)
    drop
  end  ;; on case exit, "None" requires the stack be empty
  case  ;; the "Some" case if the i32!=0 (the i32 is left on the stack)
    let (result $Coord) (local $ptr i32)
      (i32.load (local.get $ptr))  ;; load the record's fields
      (i32.load offset=4 (local.get $ptr))
      record.lift $Coord
    end
  end  ;; on case exit, "Some" requires a single $Coord value on the stack
end
```
and lowered back into the same:
```wasm
variant.lower $MaybeCoord i32  ;; pops a variant from the stack
  case  ;; for the "None" case, the stack is empty on entry
    i32.const 0  ;; push a null pointer
  end  ;; all cases must end with an i32 on the stack
  case  ;; for the "Some" case, the stack contains a $Coord on entry
    (call $malloc (i32.const 8))  ;; allocate destination pointer
    let (param $Coord) (result i32) (local $ptr i32)
      record.lower $Coord  ;; decompose element into fields
      i32.store (local.get $ptr)
      i32.store offset=4 (local.get $ptr)
      local.get $ptr
    end
  end ;; all cases must end with an i32 on the stack
end
```
Alternatively, the indirection and `$malloc` could be avoided by representing
`$MaybeCoord` as 3 scalar `i32`s that get passed into `variant.lift` and out
of `variant.lower`. Moreover, the lifting and lowering can use different
representations since they meet at the `variant` type in the middle.


### Adapter Functions

Because the Interface Types proposal is *layered* on the core wasm spec (in
the same manner as the [JS API], [C API] and [Web API]), the new types and
instructions introduced above can't simply be used in core wasm functions
because core wasm functions can only contain *core* wasm types and instructions.
Instead, the proposal defines a *new kind* of function, distinct from existing
core wasm functions, called an **adapter function**. Adapter functions have the
same structure as core functions but allow a different set of types and
instructions.

For types, adapter functions allow a superset of the core wasm `valtype`:
```
adaptertype ::= valtype | intertype
```
By allowing a mix of both `valtype` and `intertype`, adapter functions enable
the use of lifting and lowering instructions, which manipulate both.

Similarly, for instructions, adapter functions allow a superset of the core
wasm [`instr`]:
```
adapterinstr ::= instr | new instructions in this proposal
```

TODO: rewrite this

For instructions, adapter functions do *not* similarly extend the core wasm
set of instructions ([`instr`]) with lifting and lowering instructions. The
reason is that adapter functions are designed to ensure a new property that is
not in general ensured by core wasm functions:

> The **Single Reaching Lift (SRL)** property requires that each use of an
> interface type is reached by exactly one lifting instruction definition.

The reason the SRL property is valuable is described in more detail in a
[later section](#lazy-lifting-avoiding-intermediate-copies-by-design), but a
short summary is that SRL allows guaranteed elimination of intermediate copies
without relying on compiler magic. 

TODO: what we need is:
* since we're importing core instruction validation/execution, they reject
  `intertype` types and values.
  * Exception: `drop` works, b/c it's polymorphic. `select` does not.
* `intertype` values may only be manipulated by new instructions
* lifting, lowering are new, preserve SRL
* enhanced:
  * call: require non-recursive --> inline
  * let: simply allow intertype
  * local.set: not allowed for intertype

To ensure SRL, adapter functions start with `instr` and remove all the
[control instructions] with the exception of:
* `unreachable`, since traps cannot rejoin
* `throw` (but not `try`), when [exception handling] is added
* `call`, by restricting it to be non-recursive and then interpreting the
  callee to be inlined into the caller

This definition is fairly conservative and could even be loosened, while
preserving SRL, to allow blocks and branches when  interface types were not
present in the block types.

With this restriction in place, a new set `adapterinstr` is defined which
contains the remainder of `instr` extended with the new instructions added
by this proposal.

With `valtype` replaced by `adaptertype` and `instr` replaced by
`adapterinstr`, adapter functions work just like core wasm functions. For
example, the following is a trivial adapter function:
```wasm
(func (result s8)
  i32.const 42
  s8.from_i32
)
```

In general, we can categorize adapter functions into three categories:

**Export adapter functions** use only `intertype` in their signatures and use
lifting and lowering instructions to wrap a call a core wasm function. An
example export adapter function is:
```wasm
(func $print_adapter (param $msg string)
  (call $malloc_cstr (string.size (local.get $msg)))
  let (local $cstr i32)
    (string.lower_memory (local.get $cstr) (local.get $msg))
    call $core_print
  end
)
```

**Import adapter functions** use only `valtype` in their signatures and use
lifting and lowering instructions to wrap a call to an imported function
that only uses `intertype` in its signature. An example import adapter function
is:
```wasm
(func $log_adapter (param $msgPtr i32) (param $msgLen i32)
  (string.lift_memory (local.get $msgPtr) (local.get $msgLen))
  call $imported_log
)
```

**Helper adapter functions** contain an arbitrary mix of types and are used
to implement export and import adapter functions. An example might be a
helper function that lifts a C-string and thus be used by both an
export adapter (for lifting parameters) or an import adapter (for lifting
results):
```wasm
(func $lift_cstring (param $cstr i32) (result string)
  local.get $cstr
  (call $strlen (local.get $cstr))
  string.lift_memory
)
```


### Components

TODO:
* ok, we have a new kind of adapter function, where does it go?
  * can't go in existing core wasm section (b/c layered spec)
* put it in a custom section?
  * well, we also need new types, new imports, new exports, ...
  * it'd be like a whole mini-module
  * and how do we "link" it to the core module?
* fundamentally, interface types "wraps" the core wasm spec
* so we define a new kind of "interface type" module that wraps a core module
* "interface type module" sounds bad, so call it **component** (historical reason...)
* components contain
  * type section, allowing core and interface types
  * import section: no memory/table/global, intertyped function, modules and components
  * function+code section: only adapter functions
  * start section
  * export section: only functions with `intertype` signatures
  * module/modulecode section: nested module or component instances
  * instance section: instantiation of modules or components
  * alias section: alias either module or component instances
* Examples
* Note on validation: allow function section in the mix
  * Component instances created *before* nested instances
  * Nested instances are stateful
  * Calling export of nested instance before created traps
  * Component start function called after last nested instance created
* Binary format
  * core wasm module physically embedded (in chars/bytes).  Just like .wast embeds .wat
  * can reuse the same text/binary parser/decoder and validator: just conditionalize on "kind"
    * in text format: `component`
    * in binary format: split "version" `u32` into `version` and `kind` `u16`s, use new `kind`


### Lazy Lifting: Avoiding Intermediate Copies By Design

One of the [Additional Requirements](#additional-requirements) mentioned above
is to avoid intermediate O(n) copies when using Interface Types. Without
further nuance, the instructions presented in the previous section would appear
to violate that requirement by requiring an intermediate copy to reify
`interval` values between lifting and lowering.

For example, in the following instruction sequence:
```wasm
(string.lift_memory (call $create_string))
...
let (local $str string)
  (call $malloc (string.size (local.get $str)))
  local.get $str
  string.lower_memory
  ...
end
```
while we might hope that the wasm engine would be able to copy directly
from the source memory passed to `string.lift_memory` to the destination
memory passed to `string.lower_memory`, from the engine's perspective, the
intervening call to `$malloc` may mutate the linear memory read by
`string.lift_memory` and thus an intermediate copy must be made just in case.
This is also the simplest possible case, so we can conclude that, without
any help, most interface-typed values would end up with an intermediate copy.

The root problem here is that standard [eager evaluation] semantics specify
the order effects to be:
1. read from linear memory
2. perform arbitrary side effects
3. write to linear memory

What we'd like is to reorder steps 1 and 2 so that the read was always
immediately followed by the write. But...

TODO:
* these will practically almost always be in separate adapter functions, so we
  just can't do that.
* what we need is to somehow change the *spec* so that the read always happened
  *right before* the write
* How? We can employ a less-common evaluation strategy called [lazy evaluation],
  but only for lifting instructions.
* The idea behind lazy evaluation is ...
* Haskell has this. Often coupled with purity, so reordering is unobservable.
  Laziness + Side Effects usually equals madness, but in this case, it's exactly
  what we just said we wanted.
* In the context of adapters at interface boundaries copying from source to 
  destination, there should be no problem. The compiler can ensure this.
  Not something programmer writes by hand.

* But doesn't lazy evaluation add overhead? Not with the static use-def
  property. Compiler can *statically* determine where a lifting instruction
  is consumed and thus compile *all* lifting and lowering instructions into
  fused direct-copy operations.

* For example, this byte array lifting+lowering
  * Example
* would be fused by the compiler into, effectively, this core wasm code:
  * Example


### Ownership Instructions: Reliably Releasing Resources in the Presence of Laziness

TODO
* Laziness avoids copies, but adds complexity in side-effectful world
  (which is why laziness is usually coupled with purity).
* Mostly it's not a problem for interface types b/c they're only used
  at module boundaries and can be generated by a compiler that is
  careful.
* There is one persistent issue though: ownership.
* In particular: how does m

* What to do when function returns interface value?
* How to pass value that is freed right after being read?
* With laziness: don't have a good place to put the 'free`
* It depends on the lifetime of the lazy value
* So add first-class instruction and type to `adaptertype`: `(own T)`


### Exception and Trap Safety

TODO
* Default: exceptions turn to traps at boundary
* Why necessary default
* Puts the burden on exception-safe languages to make explicit as variant return type
* In theory, could add exception-specification to interface types, but it is
  morally equivalent, especially once we recognize the `result` abbreviation

TODO
* Early wasm discussions: can you observe memory after trap? Call into an instance?
* Yes, we had to because...
* But this is terrible for the ecosystem
* With IT, have shared-nothing boundary, can enforce b/c IT takes the role of host
* So we specify this for components.


## Use Cases Revisited

TODO

### Optimizing calls to Web APIs (revisited)

TODO:
* Define mapping from Web IDL to interface types
* Valuable for incorporating Web IDL specs into WASI

The way this is to be achieved is by extending the Web IDL specification to
include a "WebAssembly binding" section which describes how WebAssembly types
(including the new interface types added by this proposal) are converted to and
from Web IDL types.

Since both Web IDL and WebAssembly are statically-typed, this specification
would start by defining when two Web IDL and WebAssembly types **match**. When
two types match, the specification would define how values of the two types are
converted back and forth.

In particular, going down the list of [Web IDL types]:
* [`any`]: the existing WebAssembly [`externref`] type is already effectively
  the same as `any` in Web embeddings.
* [Primitive types]: WebAssembly value types already can be efficiently
  converted back and forth.
* [`DOMString`]: since the WebAssembly `string` type is defined as a
  sequence of [Unicode code points] and `DOMString` is defined as a sequence of
  16-bit [code units], conversion would be UTF-16 encoding/decoding, where 
  lone surrogates in a `DOMString` decode to a surrogate code point.
* [`USVString`]: a WebAssembly `string` is a superset of `USVString`.
  Conversion to a `USVString` would follow the same strategy as
  [`DOMString`-to-`USVString` conversion] and map lone surrogates to the
  replacement character.
* [`ByteString`]: as a raw sequence of uninterpreted bytes, this type is probably
  best converted to and from a WebAssembly sequence interface type.
* [`object`], [`symbol`], [Frozen array] types: as JS-specific types, these
  could either be converted to and from `externref` or be statically typed via
  reference to a [type import].
* [Interface] types: while Web IDL defines interfaces as namespace structures
  containing methods, fields, etc., for the specific purpose of typing Web API
  functions, a Web IDL Interface type just defines an abstract reference type
  used in Web API function signatures. WebAssembly can represent this type with
  either an `externref` (implying dynamic type checks) or via reference to 
  [type import].
* [Callback], [Dictionary], [Sequence] types: these would be converted to and
  from WebAssembly closure, record, and sequence interface types, respectively.
* [Record] types: WebAssembly does not currently have plans for an "ordered map"
  interface type. This type appears to be infrequently used in APIs, and as long
  as that stays the case and uses remain cold, JS objects could be synthesized
  instead.
* [Enumeration], [Nullable], [Union] types: these would be converted to and
  from WebAssembly variant interface types, by imposing various matching
  requirements on the variant type.
* [Annotated] types: the annotations don't change the representation, but imply
  additional dynamic checks on argument values.
* [`BufferSource`] types: `ArrayBuffer`s could be converted to and from
  WebAssembly sequence types, while views would depend on first-class
  slice/view reference types being added to WebAssembly, which has been
  discussed but is not yet officially proposed.

An important question is: what happens, at instantiation-time, when the Web IDL
and WebAssembly signatures don't match. One option would be to throw an error,
however, this would lead to several practical compatibility hazards:
* Browsers sometimes have slightly incompatible Web IDL signatures (which can
  occur for historical compatibility reasons).
* Sometimes Web IDL specifications are refactored over time (e.g., changing a
  `long` to a `double`), assuming the coercive semantics of JavaScript.
* JavaScript Built-in functions that today have no Web IDL signature might be
  imbued with a Web IDL signature by a future iteration of the spec that is
  subtly incompatible with extant uses that depend on coercieve JS semantics.

To address all these concerns, on WebAssembly/Web IDL signature mismatch,
instantiation would fall back to first converting all WebAssembly values to JS
values and then passing those JS values to the existing, more-coercive Web IDL
ECMAScript binding. To help well-intentioned developers avoid unintended
performance degradation, WebAssembly engines could emit warning diagnostics on
mismatch.

TODO: describe the JS API


### Defining language-neutral interfaces like WASI (revisited)

TODO:
* talk about source-language generation from .wit files
* still want some mechanism to include common typedefs: hence .witx


### Enabling "shared-nothing linking" of WebAssembly modules (revisited)

TODO
* example: how to generate source-language client definitions from .wasm


### Scoping Shared-Everything Linking (revisited)

TODO
* reify diagram with code


## Additional Requirements Revisited

TODO


## FAQ

TODO:
* How does this relate to ESM-integration?
  * As with JS API: the outside world only sees the component
  * Component (instance) imports work symmetric to Module (instance) imports in Module Linking
  * With Interface Types, ESM-integration can yield glue-code-less wasm modules
  * The only thing left to achieve parity with JS is the ability for wasm modules to import Web APIs,
    which we could get "for free" in the future from the revised [get-originals] proposal


## TODO

* slices
* an `any` type
* lifting and lowering between `intertype`s and core wasm abstract reference types
* a mechanism for specifying the receiver of a method call ([#87](https://github.com/WebAssembly/interface-types/issues/87))
* types for describing resources and the passing of handles to resources between components
* closures and a mechanism for managing the lifetime of closure state




[Core Spec]: https://webassembly.github.io/spec/core
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[Web API]: https://webassembly.github.io/spec/web-api/index.html
[C API]: https://github.com/WebAssembly/wasm-c-api
[Abbreviations]: https://webassembly.github.io/reference-types/core/text/conventions.html#abbreviations
[`name`]: https://webassembly.github.io/spec/core/text/values.html#names
[`valtype`]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype
[`instr`]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr
[Control Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions

[Reference Types]: https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md
[`externref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype

[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md

[Exception Handling]: https://github.com/webassembly/exception-handling

[Type Import]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports

[Module Linking]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md
[shared-everything-use-case]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md#shared-everything-dynamic-linking
[shared-everything-example]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Example-SharedEverythingDynamicLinking.md

[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[WASI]: https://github.com/webassembly/wasi
[`path_open`]: https://github.com/WebAssembly/WASI/blob/master/design/WASI-core.md#path_open

[Native Dynamic Linking]: https://en.wikipedia.org/wiki/Dynamic_linker
[Product Type]: https://en.wikipedia.org/wiki/Product_type
[Sum Type]: https://en.wikipedia.org/wiki/Sum_type
[Eager Evaluation]: https://en.wikipedia.org/wiki/Eager_evaluation
[Lazy Evaluation]: https://en.wikipedia.org/wiki/Lazy_evaluation
[SSA Form]: https://en.wikipedia.org/wiki/Static_single_assignment_form

[ECMAScript Binding]: https://heycam.github.io/webidl/#ecmascript-binding
[Web IDL Types]: https://heycam.github.io/webidl/#idl-types
[Primitive Types]: https://heycam.github.io/webidl/#dfn-primitive-type
[Dictionary]: https://heycam.github.io/webidl/#idl-dictionaries
[Callback]: https://heycam.github.io/webidl/#idl-callback-function
[Sequence]: https://heycam.github.io/webidl/#idl-sequence
[Record]: https://heycam.github.io/webidl/#idl-record
[Enumeration]: https://heycam.github.io/webidl/#idl-enumeration
[Interface]: https://heycam.github.io/webidl/#idl-interfaces
[Union]: https://heycam.github.io/webidl/#idl-union
[Nullable]: https://heycam.github.io/webidl/#idl-nullable-type
[Annotated]: https://heycam.github.io/webidl/#idl-annotated-types
[Typed Array View]: https://heycam.github.io/webidl/#dfn-typed-array-type
[Frozen array]: https://heycam.github.io/webidl/#idl-frozen-array
[`BufferSource`]: https://heycam.github.io/webidl/#BufferSource
[`any`]: https://heycam.github.io/webidl/#idl-any
[`object`]: https://heycam.github.io/webidl/#idl-object
[`symbol`]: https://heycam.github.io/webidl/#idl-symbol
[`DOMString`]: https://heycam.github.io/webidl/#idl-DOMString
[`ByteString`]: https://heycam.github.io/webidl/#idl-ByteString
[`USVString`]: https://heycam.github.io/webidl/#idl-USVString
[`DOMString`-to-`USVString` conversion]: https://heycam.github.io/webidl/#dfn-obtain-unicode

[Unicode Scalar Values]: https://unicode.org/glossary/#unicode_scalar_value
[Encoding API]: https://encoding.spec.whatwg.org
[Error Mode]: https://encoding.spec.whatwg.org/#error-mode
