# skills

Personal collection of **agent skills** — portable across AI coding agents
(Claude Code, Codex, and any agent that reads `.agents/skills/` or
`.claude/skills/`). Each skill lives in `skills/<name>/` with a `SKILL.md` at
its root.

## Install

```bash
npx skills@latest add rimzzlabs/skills
```

This uses the [skills.sh](https://skills.sh) CLI. It writes the canonical skill
under `.agents/skills/<name>/` and links `.claude/skills/<name>/` to it, so both
universal agents and Claude Code find each skill.

Add a single skill instead of the whole set:

```bash
npx skills@latest add rimzzlabs/skills/ste
```

## Reference

- **[ste](./skills/ste/SKILL.md)** — Rewrite and check technical writing against
  ASD-STE100 (Simplified Technical English): short sentences, active voice,
  one idea per sentence, consistent terms.
- **[rfc](./skills/rfc/SKILL.md)** — Draft a design RFC about a topic (prose in
  STE) and open it as a GitHub issue with `gh`.
- **[work-on](./skills/work-on/SKILL.md)** — Open a living GitHub issue to track
  a task, then keep it updated with decisions and status as the work moves, so
  human and agent share one source of truth.
- **[rts](./skills/rts/SKILL.md)** — TypeScript/JavaScript conventions applied
  automatically when writing JS/TS/JSX/TSX: function style, argument limits,
  declarative code, error handling, file size, and more.

## Layout

```
skills/
  <skill-name>/
    SKILL.md        # frontmatter (name, description) + instructions
    *.md            # optional companion files (templates, references)
```
