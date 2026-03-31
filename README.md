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

## Detailed Analysis

- [Phase 1 — Blueprint Analysis](./phase1_blueprint_analysis.md)
- [Phase 2 — Project Anatomy](./phase2_project_anatomy.md)
- [Phase 3 — Engine Internals](./phase3_engine_room.md)
- [Phase 4 — Developer Experience](./phase4_developer_experience.md)
- [Final Comparison — boa_gc vs oscars](./boa_gc_vs_oscars_analysis.md)
