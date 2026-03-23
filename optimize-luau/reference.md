# optimize-luau Reference Material

Lookup tables and detailed reference for the optimize-luau skill. Consult during execution as needed.

## Type System (Old Typechecker Only)

**Available NOW:**
- `--!strict` / `--!nonstrict` / `--!nocheck`
- Basic annotations: `local x: number`, `function f(a: string): boolean`
- Optional: `string?`, Union: `number | string`, Intersection: `T1 & T2`
- Generics: `function id<T>(x: T): T`, explicit instantiation: `f<<number>>()`
- Cast: `x :: string`
- Array: `{number}`, Dict: `{[string]: number}`, Table: `type T = { field: type }`
- `typeof()`, `export type`, typed variadics, alias defaults, singletons
- `never`, `unknown`, `read`/`write` modifiers
- Type refinements: `type()`, `typeof()`, `:IsA()`, truthiness, equality, `assert()`

**DO NOT recommend (requires new typechecker):**
- `keyof<T>`, `index<T,K>`, `rawkeyof<T>`, `rawget<T,K>`
- `getmetatable<T>`, `setmetatable<T,M>`
- User-defined type functions, negation types (`~T`)
- Relaxed recursive type restrictions

## Type Refinement Patterns

The compiler narrows types after certain checks, improving both type safety and native codegen. Use refinements to provide type information without explicit annotations where the type is known from a runtime check:

- **Truthiness:** `if x then` narrows `x` from falsy (`nil`/`false`)
- **Type guards:** `if type(x) == "number" then` narrows `x` to `number`
- **Typeof guards (Roblox):** `if typeof(x) == "Vector3" then` narrows to `Vector3`; `x:IsA("TextLabel")` narrows `Instance` to subclass
- **Equality:** `if x == "hello" then` narrows `x` to singleton `"hello"`
- **Assert:** `assert(type(x) == "string")` narrows `x` to `string` after the call
- **Composition:** Supports `and`, `or`, `not` for compound refinements

## Deprecated Patterns

| Deprecated | Replacement | Impact |
|---|---|---|
| `getfenv()` / `setfenv()` | `debug.info(i, "snl")` | Disables ALL optimizations. Even read-only `getfenv()` deoptimizes. |
| `loadstring()` | Restructure with `require` | Marks env "impure", disables imports |
| `table.getn(t)` | `#t` | Slower |
| `table.foreach(t, f)` | `for k, v in t do` | Slower |
| `table.foreachi(t, f)` | `for i, v in t do` | Slower |
| `wait()` | `task.wait()` | Deprecated Roblox API |
| `obj.Method(obj, ...)` | `obj:Method(...)` | Misses fast method call instruction |
| `string:method()` | `string.method(s)` | Fastcall: `string.byte(s)` faster than `s:byte()` |

## Lint Rules

| Lint | Name | Optimization impact |
|---|---|---|
| 10 | BuiltinGlobalWrite | Overwriting builtins disables fastcall |
| 22 | DeprecatedApi | Deprecated APIs: perf/correctness issues |
| 23 | TableOperations | Wrong index, redundant insert, `#` on non-arrays |
| 3 | GlobalUsedAsLocal | Global in one function -> should be local |
| 7 | LocalUnused | Dead locals |
| 12 | UnreachableCode | Dead code |
| 25 | MisleadingAndOr | `a and false or c` bugs |

## Bytecode Pattern Catalog

| Slow | Fast | Why |
|---|---|---|
| `math.floor(a / b)` | `a // b` | Dedicated VM opcode |
| `data[i].x = data[i].x + 1` | `data[i].x += 1` | LHS evaluated once |
| `a + (b - a) * t` | `math.lerp(a, b, t)` | Exact at endpoints, monotonic |
| `"pre" .. v .. "suf"` | `` `pre{v}suf` `` | Optimized `string.format` |
| Manual byte swap | `bit32.byteswap(n)` | CPU bswap |
| Manual log2 loop | `bit32.countlz(n)` | CPU instruction, ~8x faster |
| `cond and A or B` | `if cond then A else B` | Safe for falsy, one branch |

## Allocation Reduction

| Slow | Fast | Why |
|---|---|---|
| `local t = {}` in loop | `table.clear(t)` + reuse | Avoids GC pressure |
| Repeated `table.insert` | `table.create(n)` + indexed writes | Preallocated |
| Manual `pairs` clone | `table.clone(t)` | Faster, copies layout |
| `string.byte`/`char` loops | `string.pack`/`unpack` | Native implementation |
| String binary data | `buffer` type | Fixed-size, efficient |

## Inlining Rules

| Blocks inlining | Enables inlining | Why |
|---|---|---|
| `function Module.foo()` | `local function foo()` | Mutable table prevents |
| `function Obj:method()` | `local function method(self)` | OOP syntax not inlineable |
| Recursive function | Split base/recursive | Recursion prevents |
| Deep upvalue captures | Pass as parameters | Reduces closure cost |

## Native Codegen

**`--!native` (whole-file) -- use sparingly:**
- Compiles every function in the file to native code, consuming the per-module 1M instruction limit and the per-experience memory budget.
- Only appropriate for files where most/all functions are computational: math utilities, physics solvers, buffer serializers, numerical algorithms.
- Wasteful on typical game scripts where most functions are Roblox API calls, table construction, or string operations -- these don't benefit from native codegen but still eat the budget.

**`@native` (per-function) -- preferred for most files:**
- Annotate only functions that perform computational work benefiting from JIT: tight loops, `Vector3`/`CFrame` arithmetic, `buffer` ops, `bit32` ops, numerical math.
- Place directly above the function declaration. Inner functions do NOT inherit it.
- Top-level code runs once at `require()` time; `@native` provides no meaningful benefit there.
- This is the correct default strategy for mixed scripts with both hot computational code and API-heavy orchestration.

**Functions that do NOT benefit from `@native`:**
- Roblox API calls (`Instance:Clone()`, `:FindFirstChild()`, `:GetChildren()`, `:WaitForChild()`, event connections) -- execution time is in C++/engine, not Luau bytecode
- Table construction without computation (allocation-bound; the allocator is C code native codegen can't speed up)
- Heavy Luau library calls (`string.format`, `string.match`, `string.gmatch` -- C implementations that native codegen can't accelerate)
- One-shot initialization code
- Event handler registration and wiring
- `require()` calls and module setup
- Coroutine/task scheduling (`task.spawn`, `task.defer`, `task.wait`)

**Functions that benefit from `@native`:**
- Tight numerical loops (physics, interpolation, noise generation)
- Mathematical operations on table data (reading values, computing, writing back -- the Roblox docs explicitly cite this)
- `Vector3`/`CFrame` arithmetic (native generates specialized vector code with proper type annotations)
- `buffer` read/write operations (efficient native lowering)
- `bit32` operations (maps to CPU instructions)
- Hot paths called every frame (RunService handlers with computational work, not API orchestration)

**Hurts native:** `getfenv`/`setfenv`, wrong types to typed functions, non-numeric math args, breakpoints, size limits (64K instructions/block, 32K blocks/function, 1M/module).

**Budget management:** The 1M instruction limit is per-module; the memory budget for native code is per-experience, shared across ALL native scripts. Use `--record-stats=function` to check each function's native instruction count and `debug.dumpcodesize()` in Studio to see per-experience usage. A few well-chosen `@native` functions outperform blanket `--!native` across many files.

## CLI Tools

### `luau-compile` -- Static bytecode/codegen analysis (primary tool)

Modes (mutually exclusive): `binary` (raw bytecode), `text` (human-readable opcodes with source annotations), `remarks` (source with inlining/allocation comments -- **most useful**), `codegen` (native assembly, requires `--target`).

Options: `-O<n>` (0-2, default 1; `-O2` enables inlining), `-g<n>` (debug level 0-2), `--target=<arch>` (`x64`, `x64_ms`, `a64`, `a64_nf`; use `x64_ms` on Windows), `--record-stats=<total|file|function>` (JSON stats: bytecodeInstructionCount, lowerStats with spills/skipped/blocks/errors), `--bytecode-summary` (opcode distribution, requires `--record-stats=function`), `--stats-file=<name>` (default `stats.json`), `--timetrace` (trace.json), `--vector-lib`/`--vector-ctor`/`--vector-type` (always set for Roblox).

### `luau-analyze` -- Type checking and linting

Modes: omitted (typecheck + lint), `--annotate` (source with all inferred types inline -- **critical for finding `any`**).

Options: `--mode=strict` (force strict even without directive), `--formatter=plain` (machine-parseable), `--formatter=gnu` (grep-friendly), `--timetrace`.

### `luau` -- Runtime execution, profiling, and coverage

**Limited for Roblox scripts** (most depend on engine APIs). Only usable for pure-logic modules. For Roblox scripts, use Studio Script Profiler and `debug.dumpcodesize()`.

Options: `-O<n>`, `-g<n>`, `--codegen` (native execution), `--profile[=N]` (sampling at N Hz, default 10000, outputs `profile.out`), `--coverage` (outputs `coverage.out`), `--timetrace`, `-a` (program args).

### `luau-ast` -- AST dump

No flags. Outputs full JSON AST with node types, source locations, variable scopes, type annotations. Useful for programmatic analysis: function nesting depth, function body size (monolith detection), closure captures (upvalue analysis), table construction patterns, loop-invariant expression detection.

## VM Architecture

**Fastcall builtins:** `assert`, `type`, `typeof`, `rawget`/`rawset`/`rawequal`/`rawlen`, `getmetatable`/`setmetatable`, `tonumber`/`tostring`, most `math.*` (not `noise`/`random`/`randomseed`), `bit32.*`, some `string.*`/`table.*`. Partial: `assert` (unused return + truthy), `bit32.extract` (constant field/width), `select(n, ...)` O(1). With `-O2`: constant-arg builtins folded; `math.pi`, `math.huge` folded.

**Inline caching:** HREF-style for table fields. Compiler predicts hash slot; VM corrects. Best: compile-time field name, no metatables, uniform shapes.

**Import resolution:** `math.max` resolved at load time. Invalidated by `getfenv`/`setfenv`/`loadstring`.

**GC:** Incremental mark-sweep with "GC assists" (allocation pays proportional GC). Paged sweeper (16 KB pages). No `__gc` metamethod.

**Compiler:** Multi-pass. Constant folding across functions/locals. Upvalue optimization. `-O2`: inlining, loop unrolling (constant bounds), aggressive constant folding. Interprocedural limited to single modules.

**Table length:** `#t` is O(1) cached, worst case O(log N) branch-free binary search. `table.insert`/`table.remove` update the cache.

**Upvalues:** ~90% immutable. Immutable = no allocation, no closing, faster access. Mutable = extra object.

**Closures:** Cached when no upvalues, or all upvalues immutable and module-scope.

## Documentation to Consult

Read these docs during execution if needed for specific detail. Two doc trees:
- `~/.cursor/docs/usergenerated/luau/` -- Official Luau site docs (guides, types, reference, getting-started)
- `~/.cursor/docs/usergenerated/luau-rfcs/` -- Luau RFCs (language proposals, library additions)

**Luau site docs (primary):**
- `~/.cursor/docs/usergenerated/luau/guides/performance.md` -- **The official Luau performance guide.** Covers interpreter, compiler, fastcalls, inline caching, imports, table creation, GC, inlining, loop unrolling, upvalues, closures.
- `~/.cursor/docs/usergenerated/luau/guides/profile.md` -- Profiling guide (flame graphs, sampling)
- `~/.cursor/docs/usergenerated/luau/types/` -- Full type system docs (basic-types, tables, generics, refinements, unions-and-intersections, object-oriented-programs, roblox-types, considerations)
- `~/.cursor/docs/usergenerated/luau/getting-started/lint.md` -- All 28 lint rules
- `~/.cursor/docs/usergenerated/luau/reference/library.md` -- Complete library reference
- `~/.cursor/docs/usergenerated/luau/getting-started/compatibility.md` -- Luau vs Lua differences

**Luau RFCs (specific features):**
- `~/.cursor/docs/usergenerated/luau-rfcs/function-inlining.md` -- Inlining RFC (no user `@inline`, automatic at `-O2`)
- `~/.cursor/docs/usergenerated/luau-rfcs/syntax-attribute-functions-native.md` -- `@native` per-function attribute
- `~/.cursor/docs/usergenerated/luau-rfcs/function-table-create-find.md`, `function-table-clear.md`, `function-table-clone.md`, `function-table-freeze.md` -- Table operations
- `~/.cursor/docs/usergenerated/luau-rfcs/function-math-lerp.md` -- `math.lerp`
- `~/.cursor/docs/usergenerated/luau-rfcs/function-bit32-byteswap.md`, `function-bit32-countlz-countrz.md` -- CPU-level bit ops
- `~/.cursor/docs/usergenerated/luau-rfcs/function-buffer-bits.md`, `type-byte-buffer.md` -- Buffer type and ops
- `~/.cursor/docs/usergenerated/luau-rfcs/vector-library.md` -- Vector library and fastcalls
- `~/.cursor/docs/usergenerated/luau-rfcs/function-string-pack-unpack.md` -- Binary string operations
- `~/.cursor/docs/usergenerated/luau-rfcs/syntax-floor-division-operator.md` -- `//` dedicated opcode
- `~/.cursor/docs/usergenerated/luau-rfcs/syntax-compound-assignment.md` -- `+=` single LHS evaluation
- `~/.cursor/docs/usergenerated/luau-rfcs/deprecate-getfenv-setfenv.md` -- Why fenv disables optimizations
- `~/.cursor/docs/usergenerated/luau-rfcs/deprecate-table-getn-foreach.md` -- Deprecated table functions
- `~/.cursor/docs/usergenerated/luau-rfcs/generalized-iteration.md` -- Modern iteration patterns
- `~/.cursor/docs/usergenerated/luau-rfcs/generic-functions.md`, `never-and-unknown-types.md`, `syntax-type-ascription.md` -- Type system RFCs

**Cursor rules:**
- `~/.cursor/rules/usergenerated/luau.mdc` -- Luau language rules and optimization tips

**Roblox docs:**
- `~/.cursor/docs/usergenerated/roblox/en-us/luau/native-code-gen.md` -- Native codegen (size limits, Vector3 annotation impact)
- `~/.cursor/docs/usergenerated/roblox/en-us/luau/type-checking.md` -- Type annotation syntax and modes
- `~/.cursor/docs/usergenerated/roblox/en-us/performance-optimization/improve.md` -- Script computation, memory, physics, rendering
- `~/.cursor/docs/usergenerated/roblox/en-us/performance-optimization/design.md` -- Event-driven patterns, frame budgets
- `~/.cursor/docs/usergenerated/roblox/en-us/luau/variables.md` -- Local vs global performance
- `~/.cursor/docs/usergenerated/roblox/en-us/luau/scope.md` -- Scope performance implications
