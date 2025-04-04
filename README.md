# Singeli

Introductions: [Singeli as interpreter](doc/interpreter.md) | [Singeli as compiler](doc/compiler.md) -> [Purity and Ford write a min filter](doc/minfilter.md)

Singeli is a domain-specific language for building high-performance algorithms (including [SIMD](https://en.wikipedia.org/wiki/SIMD)) with flexible abstractions over code that corresponds to individual instructions. It's implemented in [BQN](https://mlochbaum.github.io/BQN), with a frontend that emits IR and a backend that converts it to C (the IR is simple, so that other backends like LLVM or machine code could be supported without much work).

With 5k lines in production [in CBQN](https://github.com/dzaima/CBQN/tree/master/src/singeli/src), I think Singeli counts as usable! See also [SingeliSort](https://github.com/mlochbaum/SingeliSort) and [1brc](https://github.com/dzaima/1brc). The core language should now be stable (and past breaking changes have been done with about a 6 month deprecation period). [Standard includes](include/) are important for programming comfortably and are less solid in some areas, particularly the more sophisticated SIMD instructions. But if a better design comes up we'd probably introduce a new include rather than make major breaking changes.

To compile input.singeli:

```
$ singeli input.singeli [-o output.c]
```

For options see `$ singeli -h` or [this section](#command-line-options). To run `singeli` as an executable, ensure that [CBQN](https://github.com/dzaima/CBQN) is installed as `bqn` in your executable path, or call as `/path/to/bqn singeli …`.

Debugging compilation errors is easy enough since they come with parsing or stack traces (if not it's a bug—please report), and `show{}` prints whatever you want at compile time. At runtime, `lprintf{}` provided by `include 'debug/printf'` prints what you pass to it, and the emitted C code is rather verbose but it embeds source function and variable names you can use to get your bearings. For tooling, the interactive [Singeli playground](https://github.com/dzaima/singeliPlayground) is a nice way to get parts of your code working without the awkward compile-debug loop, and [singeli-lsp](https://codeberg.org/dzaima/singeli-lsp) provides advanced editing features with parsing and name resolution.

Early design discussion for Singeli took place at [topanswers.xyz](https://topanswers.xyz/apl?q=1623); now it's in the [BQN forum](https://mlochbaum.github.io/BQN/community/forums.html).

## Language overview

Singeli is primarily a metaprogramming language. Its purpose is to build abstractions around CPU instructions in order to create large amounts of specialized code. Source code will tend to do complicated things at compile time to emit programs that do relatively simple things at runtime, so it's probably better to orient your thinking around what happens at compile time.

The primary tool for abstraction is the **generator**. Written with `{parameters}`, generators perform similar tasks as C macros, C++ templates, or generics, but offer more flexibility. They are expanded during compilation, and form a Turing-complete language. Generators use lexical scoping and allow recursive calls. Here's a generator that calls another one:

    def gen{func, arg} = func{arg, arg + 1}

In fact, `+` is also a generator, if it's defined. Singeli operators aren't built in; instead the user declares each as infix or prefix syntax for a generator. For example, the following line from [include/skin/cop.singeli](include/skin/cop.singeli) makes `+` a left-associative infix operator with precedence 30 (there can be one infix and one prefix definition).

    oper +  __add infix left  30

Generators can be extended with additional definitions. Each definition overrides the previous ones on its domain, which means that applying a generator searches backwards through all definitions visible in the current scope until it finds one that fits. `gen` above applies to any two arguments, but a definition can be narrowed using types and conditions:

    def gen{x:T, n, n, S if 'number'==kind{n} and T<=S} = {
      cast{S, x} + n
    }

Here, `x` must be a typed value, and have type `T`, and the two `n` parameters must match. And the condition following `if` must hold. Otherwise, previous definitions are tried and you get an error if there aren't any left.

The end goal here is to define functions for use by some other program. Functions are declared with the `fn` keyword and a parenthesis syntax at the top level, possibly with generator-like parameters. Types use `value:type` syntax rather than `type value`.

    fn demo{T}(a:T, len:u64) : void = {
      while (len > 0) {
        a = a + 1
        len = len - 1
      }
    }

Here you can see that code to be executed at runtime is written in an imperative style. `if`, `else`, and `while` control structures are provided. Statements are separated by newlines or semicolons; these two characters are equivalent.

If possible, iteration should be done with for-each loops. In SIMD programming these loops are very important and come in many varieties, taking vectorization and unrolling into account. So Singeli doesn't provide a single for loop structure, but rather a general mechanism for defining loops. Here's what a definition looks like:

    def for{vars,begin,end,iter} = {
      i:u64 = begin
      while (i < end) {
        iter{i, vars}
        ++i
      }
    }

While this loop *is* an ordinary generator (allowing other loops to call it), special syntax is used to invoke it on variables and a code block. Here's an example with two pointers:

    @for (src,dst over i from 0 to len) {
      src = 1 + dst
    }

Outside the loop, these variables are pointers, and inside it, they are individual values. It's the `iter` generator that performs this conversion, using the given index `i` to index into each pointer in `vars`. Each variable tracks whether its value was set inside the loop and writes to memory if this is the case.

## Generators

A generator is essentially a compile-time function, which takes values called parameters and returns a result. There are a few ways to create generators:

    # Anonymous generator, immediately invoked
    def a_sq = ({x} => x*x){a}

    # Named generator with two cases
    def min{a, b} = a
    def min{a, b if b<a} = b

    # Generated function
    fn triple{T}(x:T) = x + x + x

These are all effectively the same thing: there's a parameter list in `{}`, and a definition. When invoked, the body is evaluated with the parameters set to the values provided. Depending on the definition, this evaluation might end not with a static value, but with a computation to be performed at runtime (this always happens for functions). Generators are dynamically typed, and naturally have no constraints on the parameters or result. But the parameter list can include conditions that determine whether to accept a particular set of parameters. These are described in the next section; first let's go through the syntax for each case.

An anonymous generator is written `{params} => body`. If it's not the entire expression, it usually needs to be parenthesized. Otherwise `{params}` might be interpreted as a generator call on the previous word, for example. `body` can be either a single expression, or a block consisting of multiple expressions surrounded by `{}`.

A named generator is a statement rather than an expression: `def name{params} = body`. `name` can be followed by multiple parameter lists, defining a nested generator. This form also allows multiple definitions. When the generator is called, these definitions are scanned, beginning with the last, for one that applies to the arguments. That is, when applying the generator, it tests the arguments against all its conditions, then runs if they match and otherwise calls the previous definition. Sometimes it's useful to extend many generators in the same way; see [extend](#extend) for this use case.

A function can also have generator parameters. This case is discussed in the section on [functions](#functions).

### Parameter matching

A generator definition is only used when its specified parameter list matches those given, allowing multiple definitions to be effective. In addition to the names, the parameter list can include implicit and explicit constraints, such as:

* The number of parameters
* Two parameters with the same name must match
* A `par:typ` parameter must be a typed value
* An explicit condition `if cond` must hold (result in `1`)

The full system of [matching and destructuring](#matching) is described under `def`. The overall parameter list is matched in the same way as a tuple. This also means that (at most) one parameter slot can be variable-length if marked with a leading `...`. This "gathered" parameter corresponds to any number of inputs, and it's defined as the tuple of those values. For example, `def tup{...t} = t` is a possible implementation of the builtin `tup`, which returns the tuple of all parameters.

### Spread syntax

When calling a generator, any parameter slot may be preceded by `...` to expand it from a tuple into multiple parameters. For example, `gen{a, ...tup{b, c}, d}` expands to `gen{a, b, c, d}`. Any expression can follow: while `...` isn't an operator, it acts like it has infinitely low precedence.

### Partial application

A generator can be partially applied by using `.` in place of one or more parameters in call syntax. This doesn't call the generator, but creates a new generator whose parameters are used for those `.` positions. For example, `gen{., 5, .}` is a generator with two parameters, and when called on `{a, b}` the result is `gen{a, 5, b}`.

One parameter slot can also be replaced with `...` without any following expression. This stands for any number of parameters, so that for example when `gen{., 5, ...}` is called on one or more values, the first is taken for the `.` and the rest are passed at the end where the `...` is.

## Kinds of value

A generator is one kind of value—that is, something that's first-class at compile time. Like most values, it doesn't exist at runtime. Hopefully it's already done what's needed! In fact it's one of the more complicated kinds of value. Here's the full list:

| Kind      | Summary
|-----------|--------
| number    | A compile-time, high-precision number
| symbol    | A compile-time string
| tuple     | A compile-time list of values
| generator | Takes and returns values at compile time
| type      | A specific type that a runtime value can have
| constant  | A typed value known at compile time
| register  | A typed value, unknown until runtime
| function  | Takes and returns typed values at runtime
| label     | Target for `goto{}`

The simplest are discussed in this section, and others have dedicated sections below.

Numbers are floating-point, with enough precision to represent both double-precision floats and 64-bit integers (signed or unsigned) exactly. Specifically, they're implemented as pairs of doubles, giving about 105 bits of precision over the same exponent range as a double. Numeric literal syntax, which supports scientific notation and arbitrary bases up to 36, is described [below](#numeric-literals).

Symbols are Unicode strings, written as a literal using single quotes: `'symbol'`. They're used with the `emit{}` generator to emit instructions, and in `export{}` to identify the function name that should be exposed.

Constants consist of a value and a type. They appear when a value such as a number is cast, for example by creating a variable `v:f64 = 6` or with an explicit `cast{f64, 6}`. For programming, constants work like registers (variables), so there's never any need to consider them specifically. Just cast a compile-time value if you need it to have a particular type—say, when calling a function that could take several different types.

Labels are for `goto{}` and related builtins described [here](#program).

### Numeric literals

    number     = repeat* ( scientific | base )
    scientific = digit+ ( "." digit+ )? "e" "-"? digit+
    base       = ( "0x" | digit+ "b" ) ( digit | letter )+
    repeat     = digit+ ( "r" | "d" | "w" )

A number in Singeli starts with a digit but can contain letters, the decimal dot `.`, and an internal `-` as well; an initial `-` is not parsed as part of the number, but as a separate operator. Letters in numeric literals are case-insensitive and can contain underscores, which are ignored. Typical scientific notation such as `45` or `1.3e-12` is supported.

Integer hex literals can be written with `x` like `0xf3cc0`, but `b` can be used for a more general base. A decimal number between 2 and 36 before `b` gives the base, like `2b110101` or `32b0jbm1`. As with hex, digit `9` is followed by `a`.

An additional prefix allows repetition of digits to be specified. This repeats the mantissa digits, including leading 0s, as written, maintaining the base and exponent. The lowest-order digit always retains its position relative to the decimal point. Multiple prefixes can be specified as a form of documentation; they must have the same meaning.
- `r`: repeat the full sequence of digits, `3r12` means `121212`
- `d`: repeat up to the specified number, `5r123` means `23123` (starts at lowest-order digit)
- `w`: repeat up to specified bit width, `10w0x4f` means `0x34f` (base must be a power of 2)

## Definition

The `def` statement, with syntax `def target = value`, makes compile-time definitions. A value defined this way is constant in its scope (following the rules of lexical scoping), with the exception that if it's a generator it can still be extended by generator definition statements (`def target{params} = ...`). More precisely, the definition remains constant at compile time. If it's a [register](#registers), it will always refer to the same register, but the register can be mutable at runtime, allowing its value to be freely changed to another of the same type. Every copy of a mutable register made with `def` will reflect these changes, and assignments to it change the value.

### Matching

The target of a definition can be a plain name, but other forms are allowed for destructuring tuples and types, and adding conditions. Some of these are more useful for generator parameters than `def`, and in fact the operators `=` and `==` aren't allowed at the top level of a `def` target.

These are the simple targets:
* A name such as `param` matching any value
* `_` matching any value but not assigning it a name
* A numeric or symbol literal matching only that value
* `(expr)`, where expression `expr` must match the parameter when evaluated

And these are the compound targets:
* `a=b` matching targets `a` and `b` independently
* `a==b` equivalent to `a=(b)`
* `a:T` matching a typed value `a` with type `T`
* `*T` matching a pointer type
* `[k]T` matching a vector type
* `{a,b,...}` matching a tuple or tuple type

Each operator (`=`, `==`, `:`, `*`, `[k]`) is right-associative, that is, the left-hand side can only be a simple target or tuple while the right-hand side extends to the end of the expression. For example, `a:P=*V=[k]T` groups as `a:(P=(*(V=[k]T)))`, to match a variable `a` whose type is `P`, `*V`, and `*[k]T`.

Furthermore, any target may be followed by a condition such as `if a < 5`. Here `if` has to come after any operators, but it can appear in any component of a tuple matcher, or in the bracketed part `[k]` of a vector type matcher.

A tuple contains zero or more targets (and also can have an `if` condition inside even if there are no targets), and only matches a tuple or tuple type with the same length. However, it can also contain one "gathered" variable-length slot, indicated with a leading `...`, that matches zero or more values—so that the total number of values allowed is the number of non-gathered targets, or more. These values are treated as a tuple. So the target `{...a}` is identical to `a`, except that it requires the matched value to be a tuple, and if it happens to be a tuple type it will be converted to a tuple of types.

A final implicit condition is that any name that appears more than once has to be assigned to matching values, so `{a,a}` matches `tup{3,3}` but not `tup{3,4}`. This doesn't apply to the placeholder `_`, so `{_,_}` matches either tuple.

Parenthesized expressions and conditions can freely use names from anywhere else in the target. This is accomplished by evaluating them after all other matching requirements are checked and names are defined. So `{a if show{b>a}, b, b}` is a perfectly fine target, and the `show` is only shown in cases where both `b` values match. The location of the `if` condition doesn't affect its meaning at all, except that the relative ordering of conditions controls what order they're evaluated in.

Note that the type in `a:T` is also a target! A target such as `a:[8]i16` will assign to _both_ `a` and `i16`. To specify the exact type, use parentheses like `a:([8]i16)`.

## Operators

Operators are formed from the characters `!$%&*+-/<=>?\^|~`. Any number of these will stick together to form a single token unless separated by spaces. Additionally, non-ASCII characters can be used as operators, and don't stick to each other.

The `oper` statement defines a new operator, and applies to all code later in the scope (operators are handled with a Pratt parser, which naturally allows this). Here are the two declarations of `-` taken from [include/skin/cop.singeli](include/skin/cop.singeli).

    oper -  __neg prefix      30
    oper -  __sub infix left  30

The declaration lists the operator's spelling, generator, form (arity, and associativity for infix operators), and precedence. After the declaration, applying the operator runs the associated generator. More precisely, the generator name is looked up in the scope where the operator is used each time: by default the `oper` statement only associates the operator with a name and not any particular value. To assign a specific value instead, use a declaraction with the value—any expression—in parentheses such as `oper - (__neg) prefix 30`.

An operator can have at most one infix and one prefix definition. Prefix operators have no associativity (as operators can't be used as operands, they always run from right to left), while infix operators can be declared `left`, `right`, or `none`. With `none`, an error occurs in ambiguous cases where the operator is applied multiple times. The precedence is any number, and higher numbers bind tighter.

Parameters can be passed to operators before calling them, such as `a -{b} c`. This is converted to the generator call `__sub{b}{a,c}`. The arity is determined when it's called with operator syntax, and doesn't depend on any earlier calls with generator syntax.

## Types

Singeli's type system consists of the following type-kinds: basic `void` and primitive types as well as compound types constructed from several underlying types.

| Type kind   | Description                | Example display
|-------------|----------------------------|--------
| `void`      | Nothing                    | `void`
| `primitive` | A number                   | `i32`
| `vector`    | A list of same-type values | `[4]i32`
| `pointer`   | A pointer to memory        | `*f64`
| `function`  | A pointer to code          | `(i8,u1)->void`
| `tuple`     | Multiple values            | `tup{u1,[2]u32}`

The display isn't always valid Singeli code. Void and primitive types are built-in names but can be overwritten. The notation `[4]i32` resolves to `__vec{4,i32}`, where `__vec{}` is also a built-in name. And `*` indicates the built-in `__pnt{}`, which isn't defined automatically but is part of `skin/c`. The notation given for functions can't be used, and the tuple `tup{u1,[2]u32}` is technically a different value from an actual tuple type but can be used as one where a type is expected.

Primitive types are written with a letter indicating the quality followed by the width in bits. The list of supported types is given below. Note the use of `u1` for boolean data. A vector of booleans such as `[128]u1` is one important use.

    unsigned:   u1 u8 u16 u32 u64
    signed int:    i8 i16 i32 i64
    float:                f32 f64

A vector type indicates that the value should be stored in a vector or SIMD register, and the backend limits which vector sizes can be used based on the target architecture.

## Functions

Generators are great for compile-time computation, but all run-time computation happens in functions. Functions are declared and called with parenthesis syntax:

    fn times{T}(a:T, b:T) = a*b  # Type-generic function

    fn square{T}(x:T) : T = {
      times{T}(x, x)
    }

The body of a function can be either a plain expression like `a*b` above, or can be enclosed in curly braces `{}` to allow multiple statements. It returns the value of the last expression. The return type is given with `:` following the argument list, but can often be omitted (if it's left out and the body uses braces, `=` is also optional).

Function parameters like the `{T}` above are slightly different from arbitrary generator parameters: the function is only ever generated once for each unique set of parameters. Once generated, its handle is saved so that later calls return the saved function immediately (in contrast, a generator that declares a variable would make a new one each time). This avoids creating source code with lots of copies of functions, and also makes it possible for a function like `square{T}` to include recursion.

The parenthesis syntax to call a function is really just a nicer way to use the `call{}` builtin generator. As in a generator, such a call supports [spread](#spread-syntax) arguments with `...`.

A function argument can have a tuple type; in Singeli's compiled output it's flattened into multiple arguments. It can be made into a gathered argument with `...` syntax: inside a function `fn f(...a:tup{i8,i16}, b:i32)`, `a` is a tuple of registers, but the function is called as `f(a0, a1, b)` with `a0:i8` and `a1:i16`. Since the length of `a` is known based on its type, any number of arguments can be prefixed by `...`, unlike [gathered parameters](#gathered-parameters).

## Export

The `export` builtin exports values for use in the calling language. In C this means a non-`static` constant with that name is defined in the output file.

    export{'some_function', fn{i32}}     # Export as some_function()

    export{tup{'fn', 'alias'}, fn{i16}}  # Export with two names

    export{'twelve', cast{i16, 12}}      # Export a typed number

## Registers

A function's arguments, and variables that it manipulates at runtime, are represented with typed slots called registers. Registers are first-class values, meaning they can be passed around and manipulated at compile time. For example, a generator can take a register as a parameter and set its value. Copies of registers, made by passing parameters or `def` statements, exhibit aliasing, like pass-by-reference.

In a function, registers can be declared using `name : type = value` syntax, where the type can be omitted if the value is already typed. The initial value is required. A function argument is also declared as a register, but it doesn't use an initial value as that's passed in when the function is called.

    x:i32 = 25   # Type specified: numbers are untyped
    y := x + 1   # Type inferred

The runtime value of a register can be changed with `name = value` syntax, with no `:`.

    x = x + 1    # Increment

This does *not* change the compile-time value of `x` or `y`, which is a register. Declaration and reassignment do essentially the same thing at runtime, but at compile time they're two different things. Declaration is basically a `def` statement bundled with an assignment of the initial value. Assignment is an operation (in fact, a built-in operator) that acts on a register and a value, and can be used in generator calls. The left-hand side can be a full expression, as long as it resolves to a register at compile time—try `(if (0) a; else b) = c` for example. The built-in [file](include/skin/cmut.singeli) `skin/cmut` (part of `skin/c`) defines generators for C operators `+=`, `/=`, `>>=`, and so on, so you can write:

    x += 1
    ++x

At runtime, a register represents one value at a time, so whenever it's used it's the current value that will be visible. If you want to save the value somewhere, make another register, like `x0 := x`. This can be done even in a generator, so that `def clone{old} = { new:=old }` copies any typed value. So while `skin/cmut` doesn't define `x++` because Singeli has no postfix operators, it's possible to make a generator that copies the parameter, increments it, and returns the copy.

## Control flow

Singeli's built-in control flow statements are `if`-(`else`), and (`do`)-`while`. The semantics are just like in C, and the syntax is pretty similar too. You can use a single statement, or multiple statements enclosed in braces. Here are a few examples:

    if (a < 4) a = a + 1

    if (a > b) { c = a }
    else { c = b }

    do {
      doThing{c}
      c = c + 1
    } while (c < a)

The result of a condition has to have the boolean type `u1`, or be a compile-time number with value 0 or 1. The number option allows `if` to be used as a compile-time switch: it won't compile the "if" block if the condition is 0, or the "else" block if it's 1.

In any `if` or `while` condition, the pseudo-operators `and`, `or`, and `not` can be used to make a compound condition. These are implemented by manipulating jumps, never with logic instructions. `and` and `or` are short-circuiting, meaning that they don't evaluate the second part of the condition if the result is determined by the first.

### For loops

A for "loop" looks a lot like the for-each loops that are becoming common in high-level as well as low-level languages. Appearances are decieving, since it's really a special kind of generator call that can evaluate the block when it executes. But it covers the for-each functionality pretty well, with the right `for` generator.

    # Loop over three pointers with a specific range
    # Expressions in the descriptor like len/2 are only evaluated once
    @for (dst, a, b over i from len/2 to len) {
      dst = a + b + i
    }

    # Use "in" to give the element a different name from the pointer
    @other_for (x in src over n) x = 2*x

The name after `@` is just a name, and is called as a generator. The following definition of `for` gives a typical C-like loop. It's passed a generator `iter` that evaluates the block; the details are discussed below.

    def for{vars,begin,end,iter} = {
      i:u64 = begin
      while (i < end) {
        iter{i, vars}
        ++i
      }
    }

#### Descriptor

The *descriptor* is the part in parentheses, and lists the variables and range to use for the loop. It provides the `vars` (a tuple of pointers), `begin`, and `end` parameters to the `for` generator above, and it defines which names are used in the main block of the loop, which then forms the `iter` parameter.

Here's the pattern. Square brackets `[]` indicate an optional part, and the `…` means more names-maybe-in-pointers can appear, separated by commas.

    [name ["in" pointer], … "over"]
      [index ["from" begin] "to"]
        end

That's a lot of options! You can have any number of pointers, and can leave the index out—although an oddity is that you have to include it if you want a starting value different from the default of `0`. Some possibilities with the typical interpretations are listed below.

- `@for (num)`: repeat `num` times
- `@for (x in arr over len)`: iterate over elements
- `@for (i to num)`: iterate over a range
- `@for (i from a to b)`: same, with a half-open range `[a,b)`
- `@for (dst, src over i to len)`: iterate over two pointers with an index

Other than the fixed default starting point of `0`, the interpretation of the start and endpoint is all done by the generator that's used for the loop. It could ignore those values and always run exactly once, run backwards, or something else.

#### For generator

Once the descriptor and body are parsed, the following values are passed to whatever generator is named after `@`, and the result of the loop is whatever comes out. The names `vars` and so on are only for explanation, as they are nameless in Singeli compilation.

- `vars`: a tuple, the values of all pointers listed before "over"
- `begin`: the value of the expression after "from", or number 0
- `end`: the value of the expression at the end, after "to" if it's there
- `iter`: a generator that runs the for loop's body

Most of the time the generator will evaluate the block—otherwise what's the point? This is done with `iter{index, ptrs}`, where `ptrs` is a tuple of pointers (it doesn't have match `vars` created by the for loop), and `index` is an index. This generator:

- loads a value from each pointer with `load{p, index}`,
- evaluates the block using these loaded values and `index`,
- stores any modified values with `store{p, index, new_value}`, and
- returns the result of block evaluation (probably to be ignored)

Here are some examples.

    # The standard for loop, yet again
    def for{vars,begin,end,iter} = {
      i:u64 = begin
      while (i < end) {
        iter{i, vars}
        ++i
      }
    }

    # Loop expanded at compile time, implemented with recursion
    # Assumes begin and end are constant, to use compile-time if
    # Each iter{} call compiles to code with a constant index
    def for_const{vars,begin,end,iter} = {
      if (begin < end) {
        for_const{vars,begin,end-1,iter}
        iter{end-1, vars}
      }
    }

    # Loop over vectors of length vlen
    def for_vec{vlen}{vars,begin,end,iter} = {
      # Cast each pointer to a vector
      def vptr{ptr:P} = reinterpret{*[vlen]eltype{P}, ptr}
      def vvars = each{vptr, vars}

      # Endpoints for vector part
      def vb = (begin+(vlen-1))/vlen  # Round up
      def ve = end/vlen               # Round down

      # Do the loops
      for{ vars, begin,   vlen*vb, iter}
      for{vvars, vb,      ve,      iter}
      for{ vars, vlen*ve, end,     iter}
    }

`for_vec` showcases the flexibility of this approach. Since `for` is a normal generator, it can be called to avoid rewriting the same `while`-based logic everywhere. And since any pointer can be passed to `iter`, modified pointers `vvars` with a different type are fair game. This means `iter` will need to handle two different variable types, `T` and `[vlen]T`—fortunately Singeli is designed to support exactly this kind of polymorphism. And it's a requirement you control since you can decide what kind of for loop to use. Finally, `for_vec` takes another parameter *before* being called as a loop. It's invoked with, say, `@for_vec{8} (…)`.

### Match

The `match` structure performs compile-time case matching, similar to a switch-case statement. It wraps up the functionality of multiple `def` statements in an anonymous form. For example, this `match` statement:

    def result = match (x, y) {
      {{a,b}, c} => b
      {a, {b,c}} => tup{a,c}
      {...args} => 0
    }

Is equivalent to this sequence, except that it doesn't define `temp`.

    def temp{...args} = 0
    def temp{a, {b,c}} = tup{a,c}
    def temp{{a,b}, c} = b
    def result = temp{x, y}

Note the reverse order: in `match`, the first matching case will be used. The matched values (`(x, y)` above) can also be left out to create an anonymous generator. `match (x, y) {…}` behaves identically to `(match {…}){x, y}`.

## Extend

The `extend` keyword allows for programmable generator extension, so that the same extension can be applied to multiple generators easily. It's mainly used for more "internal" Singeli definitions like in `arch/c`. In short, this repetitive code:

    def sin{arg if arg<0} = -sin{-arg}
    def tan{arg if arg<0} = -tan{-arg}

could be rewritten like this:

    def extend odd{fn} = {
      def fn{arg if arg<0} = -fn{-arg}
    }
    extend odd{sin}
    extend odd{tan}

The `def extend` statement creates a special kind of generator that can only be called by an `extend` statement. This statement (which can appear at the top level or anywhere else) looks like a generator call, but the parameters, `sin` and `tan` above, all have to be names—the `extend` mechanism allows the call to modify their values. The generator (`odd` above) can be any expression, but needs parentheses in most cases if it's compound. This means additional information can be passed in with `extend (gen{args}){g0, g1}`, where `gen` is an ordinary generator whose body returns a generator defined with `def extend`.

## The top level

Top-level code is the statements that make up a file, `local` block, or body of an `if_inline`/`else` statement. Statements that could be used within a `def` or similar block context can also be used here, but there are some added features as well.

A different distinction prevents some features from being used at the top level. This is that code to be evaluated at runtime has to appear inside a function. This can be the [main function](#main), if the intent is to build a stand-alone executable. Also, a variable declared outside a function is constant and can't be modified with `=`. However, if it's a pointer to an array, it can be used in load and store instructions as usual.

### Including files

The `include` statement can be used at the top level of a program. It evaluates the specified file as though it were part of the current one. The filename is a symbol, which loads from a relative path if it starts with `.` and from Singeli's built-in scripts (kept in the [include/](include/) source folder) otherwise. The [option](#command-line-options) `-l` can be used to specify additional paths as well.

    include 'arch/c'    # Built-in library
    include './things'  # Relative path

As a result, an included file's definitions affect the file that includes it, any files that include *that* file, and other files included after that one.

### Local

The far reaching consequences of file including might not be wanted for all definitions, so the `local` keyword restricts a top-level definition to apply only to the current file, and not anything else that includes it.

    local def fn{a,b} = b              # Local generator
    local b:u8 = 3                     # Local typed constant
    local oper ++ merge infix left 30  # Local operator
    local include 'skin/c'             # Local lots of operators

The `local` keyword restricts the scope of compile-time value and operator definitions. It doesn't do anything at runtime: all the functions and so on are still placed together in one big output file.

A file can only included once per scope; inclusions after the first will be ignored.

For larger sets of definitions, `local` also allows a block syntax. The contents of the block behave like a separate file included with `local include`, and more `local` statements are allowed inside—they'll apply inside the block but not to the rest of the file.

    local {
      # All this stuff can be seen by the rest of the file,
      # but not outside it
      include 'skin/c'
      def t = 5
      def s = t + 1
    }

### if\_inline

Top-level `if` statements act as `if_inline` currently; they will be changed to act as normal `if` soon.

An `if_inline` statement is allowed at the top level, and follows the same basic syntax as `if` but has very different evaluation rules. This system is quite a mess, and hopefully a better solution will be found eventually; for now, try to avoid these!

The condition of an `if_inline` statement is evaluated in a special scope that only has access to built-in Singeli definitions, so that for example `if_inline (hasarch{'SSSE3'})` will work but not `if_inline (debug)` where `debug` has been defined previously. This is because the body of the statement doesn't have its own scope, but instead its definitions and operator declarations affect the scope it appears in—since the result of the `if_inline` statement isn't used, this is the only way for the body to define things. But if the condition could read from that scope, there's a circular dependency that can lead to paradoxes.

### Main

In order to build stand-alone Singeli programs, a `main` block defines code that will be run when the program is invoked. A full version might look like this:

    main(argc, argv) : i32 = {
      # Here argc:i32 and argv:**u8
    }

The two arguments `argc` and `argv` are the number of command-line arguments and their values, as in C. Only their names can be specified; they'll implicitly be given the types `i32` and `**u8`. Fewer names can be given (a single one will have type `i32`), and parentheses are optional if there are no arguments. The result type can be either `i32` or `void`, and will be inferred as in a typical function if omitted. This means `main` can be as simple as single expression:

    include 'debug/printf'
    main = lprintf{'Hello'}

### Config

The `config` statement allows compile-time configuration to be passed in as an argument to the build process. It has the same syntax as `def` except that the defined value must be a single name and not destructured. So `config var = 4` ordinarily gives `var` the value `4`. But if the argument `-c var='conf'` is passed when building, it'll be defined as `'conf'` instead. The command-line argument can be any code and behaves as though it was written in the definition instead, which means it can access local operators and definitions.

## Built-in generators

The following generators are pre-defined in any program. They're placed in a parent scope of the main program, so these names can be shadowed by the program.

### Program

| Syntax                 | Result
|------------------------|--------
| `emit{type,op,args…}`  | Call instruction `op` (a symbol, to be interpreted by the backend)
| `call{fun,args…}`      | Call a function
| `return{result}`       | Return `result` (optional if return type is `void`) from current function
| `export{name,value}`   | Export value for use by the calling language
| `require{name}`        | Require something from the calling language, such as a C header
| `makelabel{}`          | Create a label value
| `setlabel{label}`      | Set label to the current position
| `setlabel{}`           | Short for `setlabel{makelabel{}}`
| `goto{label}`          | Jump to the position set for a label
| `load{ptr,ind}`        | Return value at `ptr+ind` (builtin deprecated, get with `include 'arch/c'`)
| `store{ptr,ind,val}`   | Store `val` at `ptr+ind` (builtin deprecated, get with `include 'arch/c'`)

### Architecture

| Syntax               | Result
|----------------------|--------
| `setarch{feature…}`  | Set the current function's feature set
| `addarch{feature…}`  | Add features to the current function's set
| `hasarch{feature…}`  | Return 1 if all features are present and 0 otherwise
| `listarch{}`         | Current feature set as a tuple of symbols
| `witharch{fn,feat…}` | Return a generator like generated function `fn` but with the given feature set

An architecture feature is an uppercase symbol such as `'AVX2'`. Each function is created with a set of such features, which can be set outside the function using `witharch{}` or inside using `setarch{}` and `addarch{}`.

### Values

| Syntax                | Result
|-----------------------|--------
| `is{a,b}`             | Return 1 if the parameters match and 0 otherwise
| `hastype{val}`        | Return 1 if `val` is a typed value (constant, register, function)
| `hastype{val,type}`   | Return 1 if `val` is a typed value of the given type
| `type{val}`           | Return the type of `val`
| `kind{val}`           | Return a symbol indicating the kind of value
| `__set{reg,val}`      | Set value `val` for register `reg`, same as `reg = val`.
| `undefined{type}`     | Unspecified or uninitialized value of the given type
| `undefined{type,len}` | Pointer to undefined array with the given element type and length
| `show{vals…}`         | For debugging: print the parameters, and return it if there's exactly one

Possible `kind` results are `number`, `constant`, `symbol`, `tuple`, `generator`, `type`, `register`, and `function`.

### Types

| Syntax           | Result
|------------------|--------
| `width{type}`    | The number of bits taken up by `type`
| `eltype{type}`   | The underlying type of a vector or pointer type
| `vcount{[n]t}`   | The number of elements `n` in a vector type
| `cast{type,val}` | `val` converted to the given type
| `quality{type}`  | Quality of primitive type: unsigned `'u'`, int `'i'`, or float `'f'`
| `isfloat{type}`  | 1 if `type` is floating point and 0 otherwise
| `issigned{type}` | 1 if `type` is signed integer and 0 otherwise
| `isint{type}`    | 1 if `type` is integer and 0 otherwise
| `typekind{type}` | A symbol indicating the nature of the type
| `primtype{q, w}` | The primitive type with quality `q` and width `w`
| `__pnt{t}`       | Pointer type with element `t`
| `__vec{n,t}`     | Vector type with element `t` and length `n`; equivalent to `[n]t`
| `tuptype{t…}`    | Tuple type with the given element types
| `fntype{t…, res}`| Function type with the given argument types and result type `res`
| `unfntype{ft}`   | Decompose a function type, giving `fntype` parameters as a tuple

Possible `typekind` results are `void`, `primitive`, `vector`, `pointer`, `function`, and `tuple`.

#### Typecasts

Casting generators convert untyped constants to typed, or convert between types, under specific conditions. In each case an untyped constant can be converted to any type that contains it. The conditions and behavior for converting between types are shown below.

| Syntax                  | Requirement if typed                      | Conversion
|-------------------------|-------------------------------------------|-----------
| `cast{type,val}`        | `val`'s type matches `type`               | Same value
| `promote{type,val}`     | `val`'s type is a subset of `type`        | Same value
| `reinterpret{type,val}` | `val`'s type has the same width as `type` | Same binary representation

### Generators

| Syntax                    | Result
|---------------------------|--------
| `bind{gen,param…}`        | `gen{param…, ...}`
| `withenv{name,value,gen}` | `gen` but with environment `name` set to `value` during evaluation
| `getenv{name}`            | Value for `name` in environment
| `memoize{gen}`            | `gen` but return a saved result if parameters match a previous call

The program's environment used by `withenv` and `getenv` associates symbols (`name` in the table) with values. It's a form of dynamic scoping, in that associations are created and freed according to the generator call stack.

### Tuples

| Syntax                 | Result
|------------------------|--------
| `tup{elems…}`          | Create a tuple of the parameters
| `range{len}`           | The tuple 0, 1,… len-1
| `merge{tups…}`         | Concatenate elements from multiple tuples into one tuple
| `length{tuple}`        | Return the length of a tuple or symbol
| `select{tuple,ind}`    | Select the `ind`th element (0-indexed) from the tuple
| `slice{tup,start,end}` | Take slice from `start` to (optional) `end`, like Javascript
| `apply{gen,tuple}`     | `gen{...tuple}`
| `each{gen,tuple…}`     | Map a generator over the given tuples
| `symchars{symbol}`     | Return the characters of a symbol, as a tuple of symbols
| `fmtnat{num}`          | Format natural number as symbol
| `findmatches{i, f}`    | Return a tuple with, for each element of `f`, the indices of all matching elements in `i`

Generators `merge{}` and `slice{}` also work if applied to symbols instead of tuples.

### Arithmetic

Arithmetic functions are named with a double underscore, as they're meant to be aliased to operators. The default definitions work on compile-time numbers, and sometimes types. The definitions `fn{x}` or `fn{x,y}` for numbers are shown, with a C-like syntax plus `**` for exponentiation and `//` for floored division.

| `__neg` | `__shr`  | `__shl` | `__add` | `__sub` | `__mul` | `__div` | `__mod` |
|---------|----------|---------|---------|---------|---------|---------|---------|
| `-x`    | `x//2**y`| `x*2**y`| `x+y`   | `x-y`   | `x*y`   | `x/y`   | `x%y`   |

| `__and` | `__or`  | `__xor` | `__not` |
|---------|---------|---------|---------|
| `x&y`   | `x\|y`  | `x^y`   | `!x`    |

| `__eq`  | `__ne`  | `__lt`  | `__gt`  | `__le`  | `__ge`  |
|---------|---------|---------|---------|---------|---------|
| `x==y`  | `x!=y`  | `x<y`   | `x>y`   | `x<=y`  | `x>=y`  |

The following builtins do the same thing as C's math.h functions with the same names.

| `__abs`        | `__floor` | `__ceil` | `__min` | `__max` |
|----------------|-----------|----------|---------|---------|
| Absolute value | Floor     | Ceiling  | Minimum | Maximum |

Built-in arithmetic is pervasive over tuples, meaning that if one or both arguments are tuples it will map over them, recursively until reaching non-tuples.

## Command-line options

The arguments to the `singeli` command are input files and options in any order. Options all have a short form with one `-` and a long form with two. All except `-h` are followed by an additional argument.

`-h`, `--help`: Print short descriptions for all compilation options.

`-o`, `--out`: File path for compiled output; otherwise print directly to stdout.

`-t`, `--target`: Output type: `c` for C code, `cpp` for C++ (avoiding "crosses initialization" errors) and `ir` for Singeli IR. The IR format may not be stable, and that setting is currently just used for development.

`-a`, `--arch`: List of architecture features in the target system.

`-i`, `--infer`: Inference type, or when to assume one architecture feature implies another: `strict` to use only specified dependencies, `loose` to assume based on currently-existing architectures.

`-l`, `--lib`: Library paths to search in `include` statements. So `-l path` means `include 'x'` will check for `lib/x.singeli`, and `-l lib=path` means `include 'lib/x'` will check for `path/x.singeli`. All paths implied by `-l` are searched in order, followed by Singeli's built-in includes.

`-c`, `--config`: Specify the value of a `config` variable. For example, `config var=4` normally acts as `def var=4`, but with `-c var='conf'` it will act as `def var='conf'` instead.

`-p`, `--pre`: A preamble to be placed before the emitted C output.

`-n`, `--name`: Prefix to use for C functions and global variables, instead of `si_` (short for Singeli). If multiple files built with Singeli are to be included in the same project, this helps to avoid name conflicts.

`-os`, `--show`: Destination for `show{}` builtin. Set to `stdout` by default; may also be set to `stderr`, `none` to discard it, or `file=` followed by a path to write to a file on program exit.

`-oe`, `--errout`: Destination for errors when the compilation fails. Set to `stderr` by default; other options same as `-os`. Furthermore, `bqn` is identical to `none` except that an error exits the current compilation rather than the program as a whole.

`-d`, `--deplog`: A file path. A list of dependencies—files read while compiling—will be placed in this file. This way an incremental compiler framework can check these files to see if this compilation needs to be re-run.
