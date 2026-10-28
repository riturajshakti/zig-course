# Module 06 — Comptime & Generics ⭐

**Zig's actual differentiator.** Other languages bolt a second, weaker language
onto the first for compile-time work: C macros, C++ templates, Rust's trait
solver and `macro_rules!`, Go's `go:generate`. Zig runs **ordinary Zig** in the
compiler. Generics are functions, types are values, and reflection is a `for`
loop.

If you know TypeScript generics or Rust traits, the surprise is how little
syntax there is. There are no angle brackets, no `where` clauses and no `derive`
— just functions that take and return types.

> **0.17.0 redesigned reflection.** `@typeInfo(T).@"struct"` has
> **`field_names` + `field_types`** as parallel arrays. There is **no
> `.fields`**, no `f.name`, no `f.type`, and `std.meta.fields(T)` is a
> `@compileError`. `@Type` was replaced by `@Int(.unsigned, bits)`. Every
> reflection tutorial written before 0.17 will fail to compile — lesson 6.5
> shows the exact errors so you recognize them.

## Lessons

| # | Lesson | What you get |
|---|---|---|
| 6.1 | [comptime Basics](./01-comptime-basics.md) | the same function run in the compiler and on the CPU, comptime-known vs runtime, `comptime` blocks and vars, generated lookup tables, the 1000-branch quota and `@setEvalBranchQuota`, `@compileError`, `@compileLog`, comptime `if` proven to delete code, `assert` as a build failure |
| 6.2 | [Types Are Values](./02-types-are-values.md) | `@TypeOf(u32) == type`, arrays of types, generics as plain functions, the `TitleCase` convention, **type caching** (`Pair(u8,u8) == Pair(u8,u8)`), `@typeName` surprises, `switch` returning different structs, and which forms keep a readable type name |
| 6.3 | [Generic Functions](./03-generic-functions.md) | `comptime T: type` vs `anytype` side by side, generic `max`/`swap`/`sum`/`zip`, duck typing with no interfaces, **peer type resolution** (`@TypeOf(a, b)`), constraints via `@typeInfo` + `@compileError`, and a constraint-check table |
| 6.4 | [Generic Containers](./04-generic-containers.md) | `Stack(T)` from raw `realloc`, `Queue(T)` over `ArrayList`, `const Self = @This()`, allocator-per-call, `deinit`, **how `std.ArrayList(T)` is literally this**, nesting `Stack(Stack(u8))`, and a fixed-capacity `RingBuffer(T, n)` with no heap |
| 6.5 | [Reflection](./05-reflection.md) | the **0.17 parallel-array redesign** and the errors the old API gives, `inline for` over fields, `@field` read *and* write, field defaults, a recursive auto-serializer, a generic `eql` (and why `==` refuses structs), `@hasDecl`, `std.meta` helpers, reflecting on a module |
| 6.6 | [comptime Costs](./06-costs.md) | one compiled copy per type, counting them with `@compileLog`, **measured**: +25,040 bytes for 55 instantiations, **11.6 s vs 1.1 s** compile time for the same answer, comptime tables as shipped `.rodata`, `anytype` vs explicit error quality, when a tagged union beats generics |

Every program is complete and runnable — copy a block into a `.zig` file and
`zig run` it as-is. **The failing ones are labelled.** Comptime teaches largely
through compile errors, so this module has more deliberate failures than any
other; the error text shown is the real output, and learning to read it is the
skill.

## The mental model

Three sentences that cover almost everything:

1. **A type is a value of type `type`.** So a generic is a function taking
   `comptime T: type` and returning a type — no new syntax required.
2. **Type functions are memoized on their arguments.** `Stack(u8)` called in two
   files is the *same* type; `Stack(u8)` and `Stack(u16)` are different ones,
   each with its own compiled methods.
3. **`comptime` means "evaluated before the program runs."** Branches with a
   comptime-known condition are deleted, not predicted; comptime loops leave no
   loop behind; comptime data becomes bytes in the binary.

## The interview answer

> Zig has no separate template language — generics are functions that take
> `comptime` type parameters and return a type, so `ArrayList(T)` is literally a
> function call whose result is memoized on its arguments. That memoization is
> what makes the types compatible across files. Reflection is `@typeInfo` plus
> `inline for`, which replaces derive macros for serialization and comparison;
> in 0.17 it exposes fields as parallel `field_names`/`field_types` arrays. The
> cost is monomorphization — one compiled copy per instantiation, which I'd
> measure before generalizing — and compile time, since the comptime interpreter
> is far slower than the compiled code it replaces. When the set of types is
> closed or needs to vary at runtime, a tagged union or a vtable like
> `std.mem.Allocator` is the right tool instead.

---

Next: [Module 07 — Standard Library Tour](../07-stdlib-tour/)
