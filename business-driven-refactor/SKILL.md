---
name: business-driven-refactor
description: Business-first refactoring — untangle patched-up business logic by reconstructing business intent before touching code structure. Use when business meaning is lost in the code.
---

# Business-Driven Refactor

**Core belief**: messy code is often not laziness — the business itself got patched up. Refactoring without first understanding the business just neatly organizes garbage.

## Lenses

Apply these while reading code. Each names a specific way business intent and code reality misalign.

**Patch Layer** — later requirements attached to core logic through conditionals and special cases. Signature: you can "see time" — clear logic, then more and more else/if appended. Signals:
- Condition chains (`if type == "A" ... else if type == "B" ...`)
- "Also" comments (`// Also update inventory`)
- Functions named with "And" (`createOrderAndNotifyAndLog`)
- Magic numbers with no business explanation
- Commented-out code (previous person was also afraid to delete)
- Variable names that don't match what the variable actually holds

**Intent Drift** — code behavior no longer matches original business intent. The feature works, but the story the code tells and the story the business needs don't align. Signal: `pendingOrder` actually stores "paid but not shipped."

**Ghost Logic** — code still executing for a business need that no longer exists. Signature: nobody can answer "can we delete this?"

**Logic Fragmentation** — a single business rule scattered across files. Each fragment alone doesn't express complete business meaning.

**Business Invariant** — a rule that must not break no matter how the code changes. Many bugs originate from patches breaking invariants.

**Core Flow vs Exception Flow** — the primary path (>80%) vs edge cases. Red flag: exception flow overwhelms core flow — readers get lost in edge cases.

## Gap Categories

**Ghost Logic** — code doing something the business no longer needs.

**Fragmentation** — a rule spread across files; nowhere states the complete business meaning.

**Invariant Bypassed** — code structurally allows something that must never happen.

**Patches Overwhelming Core** — exception flows outweigh the core flow.

**Naming Betrayal** — code naming doesn't match business terminology.

## Execution

Archaeology → Confirmation → Gap Analysis → Realignment.

**Archaeology**: reconstruct business map from code and git history. Trace each entry point's happy path and branches. Flag every patch signal.

**Confirmation**: present findings to the user. Surface low-confidence items — the ones where code alone can't reveal business intent — and confirm them.

**Gap Analysis**: categorize every misalignment using the lenses above.

**Realignment**: for each gap, propose approach, migration path, priority, and risk. Core principle: **restructure first, change behavior never** — each step small enough to complete under test protection.

## Done When

1. Business map reconstructed from code and confirmed with user
2. Every business-code misalignment identified and categorized
3. Each gap has a realignment approach with priority and risk discussed
4. Newly clarified business concepts and confirmed invariants recorded

## Red Flags

- "Just refactor it, don't worry about the business" — refuse. Refactoring without understanding the business manufactures bugs.
- Chaotic git history — flag honestly: expect more low-confidence findings.
- Business invariant with no code constraints or test protection — high-priority; add tests before refactoring.

## Boundary

Best used when business logic has been patched repeatedly and the "why" is unclear. Skip for: structural refactors with clear business logic, pure technical debt, greenfield features, or when you already know exactly what to change.
