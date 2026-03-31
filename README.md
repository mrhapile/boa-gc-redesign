# Boa Garbage Collector Redesign

This repository contains a deep architectural analysis of the Boa JavaScript engine's current garbage collector (`boa_gc`) and the experimental `oscars` GC.

## What this repo includes

- Phase 1 → Design decisions and why `boa_gc` is limiting
- Phase 2 → Project structure, dependencies, and architecture
- Phase 3 → Deep dive into allocator and collector internals
- Phase 4 → Developer experience and derive macros
- GC Comparison → Side-by-side analysis and integration strategy

## Goal

To understand, evaluate, and propose a safe integration strategy for a production-ready garbage collector in Boa.

## Key Insights

- `boa_gc` is limited by allocation and root detection complexity
- `oscars` improves allocation and architecture but introduces integration challenges
- The hardest problem is not GC algorithms, but API and engine assumptions

---

This work is part of my preparation for contributing to Boa’s GC redesign.
