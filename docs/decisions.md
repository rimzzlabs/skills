# Decisions

## 2026-09-19 — rts: ts-belt and ts-pattern for data handling

**Decision:** The `rts` skill prefers `@mobily/ts-belt` (pinned to
`4.0.0-rc.5`) for data transformation and `ts-pattern` for branching on more
than two cases. The agent asks the user before it installs them. Nested
ternaries are not permitted. A single flat ternary between two simple values is
permitted.

**Why:** Nested ternaries are hard to read. ts-belt gives immutable, typed
functions that compose with `pipe()`. ts-pattern gives exhaustive matching, so
the compiler finds a missing case.

**Rejected:**

- A ban on all ternaries. It conflicts with simple code in SKILL.md rule 18.
- lodash, ramda, or remeda as the utility library. rts uses one library for this
  work, not several.
- An unpinned ts-belt version. Version 4 is a release candidate, and its API can
  change between releases.

**Open:** ts-belt `R` versus the hand-written `Result` from SKILL.md rule 7. The
agent asks the user for each project.
