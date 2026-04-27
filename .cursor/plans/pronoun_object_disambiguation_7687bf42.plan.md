---
name: pronoun object disambiguation
overview: Refine pronoun resolver heuristics to resolve object pronouns in multi-candidate windows without reducing precision, while preserving current conservative behavior for subject pronouns.
todos:
  - id: resolver-role-metadata
    content: Add subject/object role metadata and helper-based object disambiguation in pronoun-resolver.ts
    status: completed
  - id: resolver-tests
    content: Add and adjust resolver unit tests for object-disambiguation and ambiguity guardrails
    status: completed
  - id: architecture-update
    content: Document object-pronoun disambiguation behavior and rationale in web-app/Architecture.md
    status: completed
isProject: false
---

# Pronoun Resolver Object-Ambiguity Plan

## Goal

Improve object-pronoun resolution for cases like `Tom killed him` while keeping subject-pronoun ambiguity conservative and preserving existing API/UI contracts.

## Scope

- In scope: resolver logic and resolver unit tests.
- Out of scope: route contracts, API payload changes, UI behavior changes.

## Files To Change

- `[/Users/roopamgarg/Development/narrative-checker/web-app/src/lib/pronoun-resolver.ts](/Users/roopamgarg/Development/narrative-checker/web-app/src/lib/pronoun-resolver.ts)`
- `[/Users/roopamgarg/Development/narrative-checker/web-app/src/lib/pronoun-resolver.test.ts](/Users/roopamgarg/Development/narrative-checker/web-app/src/lib/pronoun-resolver.test.ts)`
- `[/Users/roopamgarg/Development/narrative-checker/web-app/Architecture.md](/Users/roopamgarg/Development/narrative-checker/web-app/Architecture.md)`

## Design Decisions

- Add pronoun role metadata to tracked pronouns:
  - `subject`: `he`, `she`, `they`
  - `object`: `him`, non-possessive `her`, `them`
- Role assignment is lexical-form based (lookup by token text), not POS-based role inference.
- Keep `isPossessiveHer()` as a hard gate before role assignment.
- Preserve existing logic for:
  - single-candidate windows (`distinct.size === 1`) for subject pronouns
  - hint/class conflict skips (`canResolvePronoun === false`)
  - subject pronouns with multiple candidates (still skip).
- For object pronouns, run deterministic same-sentence exclusion before the crowding gate:
  1. Exclude each candidate entity that has at least one proper-name mention of that same entity in the pronoun sentence before the pronoun token (same-sentence named entities are treated as likely local subject/agent).
  2. Recompute `distinct.size` on filtered candidates and apply existing crowding rule: resolve only when exactly one distinct candidate remains.
  3. If 0 or >1 remain, skip (no recency tie-break guessing).
  4. Preserve the existing post-crowding agreement check (`canResolvePronoun`) on the single surviving candidate.
- Subject-pronoun logic remains unchanged; only object pronouns get this pre-crowding exclusion.
- Intra-sentence processing remains left-to-right with no feedback loop into mention extraction or hint building.

## Implementation Steps

1. Extend `TrackedPronoun` type with `role: "subject" | "object"` and update `getTrackedPronoun()` mappings.
2. Add a focused helper in resolver module (pure function) to filter object-pronoun candidates before crowding evaluation.
  - Proposed signature:
   `filterObjectPronounCandidates(distinct, pronounSentenceIndex, pronounTokenIndex, pronounClass, globalHints): Map<string, PersonMention[]>`
  - Helper docs must clarify exclusion is per-entity (not global): a candidate is removed only if that candidate appears as a proper-name mention in the same sentence before the pronoun.
3. Wire helper into the main resolution loop for all `trackedPronoun.role === "object"` cases before the existing `distinct.size` crowding gate check.
4. Keep applied replacement generation and stats updates consistent with current behavior.
5. Update tests to codify new behavior and ambiguity guards.
6. Update architecture docs with the new object-pronoun disambiguation rule and rationale.

## Test Plan

- Add/adjust resolver tests:
  - `Adam was a good person. Tom killed him.` resolves `him -> Adam`.
  - `Adam met Tom. He smiled.` remains skipped unchanged.
  - `Tom warned Adam. Adam ignored him.` resolves `him -> Tom`.
  - ambiguous multi-candidate object case still skips (no guessed winner).
  - standalone in-sentence-only object candidate: `Tom killed him.` remains skipped.
  - multi-candidate survivors after object filter (e.g. `Adam met Tom. Chris saw him.`) remain skipped.
  - mixed possessive/object `her` boundary: `Aria tightened her grip. The light struck her.` -> `Aria tightened her grip. The light struck Aria.`
  - adjust existing conflict-downgrade expectation to reflect object same-sentence exclusion: in `Alex said he was ready. Alex then told her to hurry. They waited.`, `her` remains unchanged while `he` and `They` still resolve.
  - hint-regression guard: `Adam was brave. Eve liked him. The crowd cheered her.` remains unchanged for `her` (skip), preventing false positives from pre-crowding gender filtering.
- Run resolver and route tests to confirm no regressions outside resolver behavior.

## Risks And Mitigations

- Risk: over-resolving ambiguous objects.
  - Mitigation: require unique survivor after object-specific filtering; otherwise skip.
- Risk: accidental behavior drift for subject pronouns.
  - Mitigation: preserve existing subject ambiguity skip path and add explicit test coverage.
- Risk: maintainability in hot loop.
  - Mitigation: keep new logic in a small pure helper function.

## Acceptance Criteria

- New object-pronoun examples resolve as expected.
- Subject-pronoun ambiguity behavior remains unchanged.
- Existing contract surfaces remain unchanged.
- Resolver tests pass with new cases and no regressions.
- `Architecture.md` reflects updated resolver decision flow.

