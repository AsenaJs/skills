# AsenaJS Agent Skills

Agent Skills that teach AI coding agents (Claude Code, Cursor, Copilot, Windsurf, and any tool supporting the [Agent Skills](https://agentskills.io) standard) how to write correct [AsenaJS](https://asena.sh) code.

## Install

```bash
npx skills add asenajs/skills
```

Or copy `skills/asena/` into your project's skill directory (e.g. `.claude/skills/asena/` for Claude Code).

## Skills

| Skill | Description |
|---|---|
| [`asena`](skills/asena/SKILL.md) | Umbrella skill for the whole framework: core rules, bootstrap, import paths, and per-feature reference files (controllers, DI, middleware, WebSocket, microservices, database, testing, CLI, adapters) loaded on demand |

## What the skill knows

- Only the **published** npm API — the version matrix is at the top of [SKILL.md](skills/asena/SKILL.md).
- Distilled from the official docs at [asena.sh](https://asena.sh); the full docs remain reachable per page via `https://asena.sh/raw/<path>.md` (see `references/docs-map.md`).

**Synced with asena.sh docs as of 2026-08-17, @asenajs/asena 0.10.1.**

## Maintenance (release checklist)

On every framework/package release:

1. Update the version matrix in `skills/asena/SKILL.md` — versions are not aligned across packages, this drifts first.
2. Regenerate `references/docs-map.md` from `Website/public/llms.txt` (mechanical reformat), and spot-check that every `https://asena.sh/raw/...` URL in the references still resolves (the correct form has NO `/docs` segment).
3. Grep the docs for new `:::warning Upgrading` / "New in 0.x" blocks — they are a ready-made diff of what the skill must change.
4. Re-run the eval prompts (CRUD scaffold, WebSocket namespace, createTestApp test, pitfall probe) in a fresh agent session and type-check the output against the published packages.
5. Diff `@asenajs/asena`'s `package.json` `exports` field against the import cheatsheet — and diff the reference files' import statements against it too, not just SKILL.md.
6. Diff the deliberately duplicated snippets (bootstrap, middleware, validator, `onError`) between SKILL.md and the references — they must stay semantically identical.
7. Diff the llms.txt page list — added/removed pages mean new/removed reference sections.
8. Bump the sync line above and the `version` field in the SKILL.md frontmatter.

## License

MIT
