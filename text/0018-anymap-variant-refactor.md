# Replace AnyMap's manual union with std::variant

The `any_map` / `AnyMap` class uses a hand-rolled discriminated union (a `union` of heap-allocated map pointers plus a `map_type` enum tag) with manual lifecycle management (`new`/`delete`, `copy_from`/`move_from`/`destroy`). The iterators mirror this: a union of heap-allocated STL iterator pointers with 460 lines of manual memory management. We will replace this with `std::variant`-based inline storage, collapse the class hierarchy, and remove ~850 lines of boilerplate.

## Status

Proposed

## Considered Options

### A: std::variant inline storage (chosen)

Replace `union { ordered_any_map*; unordered_any_map*; unordered_any_cimap*; } map` with `std::variant<ordered_any_map, unordered_any_map, unordered_any_cimap>`. Iterators become a small class wrapping `std::variant<oiter, uoiter, uociiter>`. The `any_map` base class is collapsed into `AnyMap`, with `using any_map = AnyMap` for source compatibility.

### B: Keep union, modernize lifecycle only

Replace raw `new`/`delete` with `std::unique_ptr` inside the union. Reduces leak risk but keeps all the switch-dispatch boilerplate, the two-class hierarchy, and the heap-allocated iterators. Net reduction: ~100 lines. Does not address the fundamental complexity.

### C: Type-erase via virtual base class

Replace the union with a `std::unique_ptr<MapBase>` where `MapBase` has virtual `find`/`begin`/`end`/`at` etc. Eliminates switch-dispatch but adds virtual call overhead on every map operation, prevents inlining, and requires a parallel virtual iterator hierarchy. More code, worse performance.

## Key Decisions

| Decision              | Choice                                     | Rationale                                                         |
|-----------------------|--------------------------------------------|-------------------------------------------------------------------|
| Storage               | Inline variant                             | Eliminates heap allocation, pointer indirection, manual lifecycle |
| Iterators             | Variant-wrapping class                     | Same public API, no heap alloc per iterator, ~50 lines vs ~460    |
| TypeChecked functions | Removed                                    | Callers use `std::get<T>()` on the public variant directly        |
| Class hierarchy       | Collapsed; `using any_map = AnyMap`        | One using-alias in tests/downstream; no behavior difference       |
| Friend access         | Removed                                    | Variant is public; `Properties`/`LDAPExpr` use `std::get<T>()`    |
| map_type enum         | Retained                                   | Construction tag; `GetType()` maps `variant.index()` to enum      |
| Export                | Class-level `US_Framework_EXPORT` retained | Simplicity over micro-optimization                                |
| ABI                   | Breaking change; full rebuild required     | Acceptable in monorepo/source-built contexts                      |

## Performance Analysis

### Improvements

**1. Eliminated heap allocation for map storage**

Before: Every `AnyMap` construction calls `new` to allocate the underlying STL map on the heap. Every copy does the same. Every destruction calls `delete`.

After: The variant holds the map inline. Construction, copy, and destruction operate on the object's own storage. This removes one `malloc`/`free` pair per `AnyMap` lifetime.

Impact: Significant in code paths that construct many short-lived `AnyMap` objects (e.g., manifest parsing, service property construction). Eliminates heap fragmentation from many small map allocations.

**2. Eliminated pointer indirection on every map access**

Before: Every `at()`, `find()`, `operator[]`, `size()`, `empty()` dereferences a pointer (`map.o->find(key)`).

After: `std::visit` or `std::get` operates directly on inline storage. One fewer cache miss per access.

Impact: Measurable on hot paths like LDAP filter evaluation that perform many lookups per expression.

**3. Eliminated heap allocation for iterators**

Before: Every `begin()`, `end()`, `find()` call allocates a heap iterator (`new oiter(...)`) inside the polymorphic iterator wrapper. Every iterator copy allocates again. Every destruction calls `delete`. A simple `for (auto it = m.begin(); it != m.end(); ++it)` loop does: 2 heap allocations (begin + end) + 1 per post-increment copy + 2 deletes minimum.

After: Iterator wraps a `std::variant<oiter, uoiter, uociiter>` by value. Zero heap allocations. Copy is a trivial variant copy (all three iterator types are pointer-sized or close to it).

Impact: This is the single largest performance win. Iterator-heavy code (serialization, property enumeration, LDAP evaluation) was paying malloc/free per iterator operation. The `TypeChecked` fast-path functions existed specifically to work around this cost.

**4. Properties and LDAPExpr hot paths use std::get (zero dispatch overhead)**

Before: `TypeChecked` functions asserted the type at runtime, then dereferenced the pointer. The assert is stripped in release builds, leaving a raw pointer deref.

After: `std::get<T>(variant)` on a variant whose active alternative is known. In optimized builds, this compiles to the same direct access (the `bad_variant_access` check is elided when the compiler can prove correctness, and even when it can't, it's a single index comparison — same cost as the assert was in debug builds).

Impact: Neutral to slightly better (no pointer deref).

### Costs

**1. Increased sizeof(AnyMap)**

Before: ~16 bytes (one pointer + one uint8_t enum + padding).

After: ~56-72 bytes (size of the largest variant alternative, likely `unordered_any_cimap` which contains a hasher, equality comparator, and the hashtable state).

Impact: Negligible. `AnyMap` objects are not stored in arrays or contiguous containers. They live as individual objects (in `Any` wrappers, on the stack, in service properties). The 40-56 byte increase is dwarfed by the map's own internal heap allocations for buckets/nodes. The tradeoff — slightly larger object but one fewer heap allocation — is a net memory win for any map with more than zero elements.

**2. std::visit dispatch overhead on general-purpose methods**

Before: `switch(type)` with three cases.

After: `std::visit([](auto& m) { ... }, variant_)` — typically compiled to the same jump table or if-else chain.

Impact: Neutral. Modern compilers (GCC 9+, Clang 10+, MSVC 19.20+) optimize `std::visit` on small variants to the same code as a manual switch. Benchmarks consistently show zero overhead for variants with 3-4 alternatives.

**3. Iterator variant dispatch on increment/dereference**

Before (general path): switch on `iter_type` enum, dereference heap pointer.
After: `std::visit` on 3-alternative variant.

The old path had *two* costs: dispatch + pointer deref. The new path has only dispatch. Net: faster.

Before (TypeChecked path): direct STL iterator, zero overhead.
After (via `std::get` on public variant): direct STL iterator, zero overhead.

Impact: Neutral to faster.

### Summary

| Operation                     | Before                 | After               | Change                      |
|-------------------------------|------------------------|---------------------|-----------------------------|
| AnyMap construction           | 1 heap alloc           | 0 heap allocs       | Faster                      |
| AnyMap copy                   | 1 heap alloc           | 0 extra allocs      | Faster                      |
| Map access (`at`/`find`/`[]`) | pointer deref + switch | visit (inline)      | Faster (1 fewer cache miss) |
| Iterator construction         | 1 heap alloc each      | 0 heap allocs       | Much faster                 |
| Iterator copy                 | 1 heap alloc           | trivial copy        | Much faster                 |
| Iterator increment (general)  | switch + pointer deref | visit               | Faster                      |
| Iterator increment (hot path) | direct (TypeChecked)   | direct (`std::get`) | Neutral                     |
| sizeof(AnyMap)                | ~16 bytes              | ~56-72 bytes        | Larger (negligible)         |
| Memory per AnyMap instance    | object + heap map      | object (inline map) | Less total                  |

The refactor is a strict performance improvement on every operation that matters, with the only "cost" being a larger stack footprint that is irrelevant in practice.

## Consequences

- All consumers of `AnyMap` must recompile (ABI break).
- ~35 call sites in `Properties.cpp` and `LDAPExpr.cpp` change from `TypeChecked` calls to `std::get<T>()`.
- Downstream MathWorks code (9 files, ~18 references) continues to compile via `using any_map = AnyMap` alias.
- The `map_type` enum and nested typedefs (`ordered_any_map`, `unordered_any_map`, `unordered_any_cimap`) remain accessible at `AnyMap::`.
