# Architectural Analysis: boa_gc vs. oscars

**Document Purpose:** A comparative architectural teardown of the legacy `boa_gc` prototype and the experimental `oscars` garbage collector designed for the Boa JavaScript engine.

---

## 1. Executive Summary

The transition from `boa_gc` to `oscars` represents a fundamental shift from a monolithic, system-allocator-bound prototype to a modular, production-ready, size-class pool allocator. The primary drivers for this rewrite are eliminating the catastrophic $O(N)$ dynamic root detection, fixing memory fragmentation caused by scattered heap allocations, and laying the groundwork for future compacting (moving) collectors.

---

## 2. Memory Layout & Allocation

The most significant performance delta between the two systems lies in how OS memory is requested and structured.

| Feature | `boa_gc` (Current) | `oscars` (Next-Gen) |
| :--- | :--- | :--- |
| **Allocator Type** | Global System Allocator | `mempool3` (Size-Class Pool Allocator) |
| **Mechanic** | `Box::new` for every object | 12 fixed-size slot pools + Bump Pages |
| **Metadata Location** | Inline 16-byte `GcBox` header | Out-of-band bitmap (1 bit/slot) |
| **Deallocation** | Immediate `free()` via Box | Free-list recycling & deferred page drop |
| **Memory Locality** | Extremely poor (fragmented) | High (contiguous arrays of identical sizes) |

**Key Takeaway:** `oscars` achieves $O(1)$ allocation via bump-pointers and free-lists, completely bypassing the heavy system `malloc` overhead that plagues `boa_gc`. 

---

## 3. Rooting & Tracing Algorithms

The core logic of identifying live objects and reclaiming dead ones has been entirely re-engineered.

### Root Detection
* **`boa_gc`:** Uses an expensive $O(N)$ pre-pass called `trace_non_roots`. It iterates the *entire heap* to calculate incoming internal edges. Roots are dynamically deduced where `ref_count > non_root_count`.
* **`oscars`:** Abandons dynamic deduction. Uses a strict `root_count` on the `GcHeader` incremented via explicit handle clones. The `trace_non_roots` phase is completely eliminated.

### Marking Phase
* **`boa_gc`:** Uses an explicit `Tracer` struct with a `VecDeque` worklist. It is iterative and entirely safe from stack overflows, regardless of JS object graph depth.
* **`oscars`:** Uses **direct recursion** (`Trace::trace(&self.value, color)`). While it features tri-color epoch flipping (Grey-marking) for cycle protection, the recursive design introduces a critical stack overflow vulnerability on deep object graphs.

---

## 4. API Surface & Engine Integration

Because Boa's VM is tightly coupled to the shape of `Gc<T>`, the pointer APIs are the most complex integration boundary.

| API Element | `boa_gc` | `oscars` | Migration Impact |
| :--- | :--- | :--- | :--- |
| **Strong Pointer** | `Gc<T>` | `Gc<T>` (Wraps `ErasedPoolPointer`) | Drop-in replacement. |
| **Allocation** | `Gc::new(val)` | `Gc::new_in(val, &collector)` | **Major.** Requires passing a collector reference everywhere or building a thread-local shim. |
| **Cyclic Allocation** | `Gc::new_cyclic(...)` | *Not Implemented* | **Blocker.** Must be implemented for Boa AST nodes. |
| **Interior Mutability**| `GcRefCell<T>` | `GcRefCell<T>` | Preserved. Dynamic `BorrowFlag` works identically. |
| **Weak Collections** | `WeakMap` | `WeakMap` | Preserved. `oscars` owns the map memory directly. |

---

## 5. Procedural Macros (`#[derive(Trace)]`)

The `oscars_derive` crate successfully mimics the developer experience of `boa_gc` while generating vastly simplified code.

* **Code Generation:** Both generate a field-by-field traversal for structs and enums. Both automatically generate a `Drop` implementation that enforces `Finalize::finalize()`.
* **Elimination:** `oscars_derive` no longer generates `trace_non_roots`, halving the boilerplate.
* **Attributes:** `#[unsafe_ignore_trace]` remains functionally identical, allowing the safe bypass of raw data fields.
* **Bounds:** The `Trace` trait now takes a `TraceColor` epoch parameter instead of a mutable `Tracer` reference, requiring minor mechanical refactoring for manual implementations across the engine.

---

## 6. Critical Integration Path

To successfully replace `boa_gc` with `oscars` without causing VM regressions, three engineering blockers must be resolved:

1.  **Stack Safety Mitigation:** The recursive marking algorithm in `oscars` must be rewritten into an iterative worklist to survive ECMAScript Test262 deeply-nested object tests.
2.  **State Management Bridge:** The bytecode compiler must either be refactored to thread a `&Collector` to all `Gc::new_in` callsites, or a `thread_local!` fallback must be constructed in `oscars` to emulate the old global `Gc::new` behavior.
3.  **Cyclic Support:** `Gc::new_cyclic` must be implemented against the `mempool3` allocator to support standard self-referential JS structures like closures and environment records.
