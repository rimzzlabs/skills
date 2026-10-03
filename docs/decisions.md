# Decisions

## 2026-10-04 — split rts into typescript and react

**Decision:** Rename the `rts` skill to `typescript`. Move the React rules
(components, hooks, JSX composition, WAI-ARIA, `useEffect`, component file
grouping) and the React libraries (TanStack Query, Zustand, React Hook Form)
into a new `react` skill. `react` depends on `typescript`; `typescript` does not
depend on `react`.

**Why:** One skill covered two concerns. Backend TypeScript code loaded React
rules it does not use, and the React rules had no room to grow. Two skills with
one concern each are easier to read and to install separately.

**Rejected:**

- Keep one skill and add a React section. The file would pass the 600–800 line
  cap that its own rule 8 sets.
- Make `react` standalone. It would duplicate function style, `Result` errors,
  and immutability rules, and the two copies would drift.

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
