# agent-kit

My agentic tooling: pi skills first, other reusable agent bits as they show up.

## Layout

```
skills/             one directory per skill, each with a SKILL.md
AGENTS_TEMPLATE.md  baseline AGENTS.md guidelines, merge with project rules
ANTI_AGENTS.md      prompt that does the opposite, for testing instruction adherence
```

`agents/`, `prompts/`, `snippets/` and `docs/` get added when they actually have something in them.

## Skills

| Skill | What it does |
|---|---|
| `rails-feature-pipeline` | Implement a Rails feature end to end: migration, tests, OpenAPI spec, full suite, then a tester agent review. |
| `openapi-updater` | Keep `docs/openapi.yml` in sync with routes and tests. |

## Install

Skills are symlinked into the harness skill dirs, so edits here apply immediately. Per skill:

```sh
ln -sfn ~/src/agent-kit/skills/rails-feature-pipeline ~/.pi/agent/skills/rails-feature-pipeline
ln -sfn ~/src/agent-kit/skills/openapi-updater ~/.agents/skills/openapi-updater
```

Install every skill into one harness dir:

```sh
for d in ~/src/agent-kit/skills/*/; do ln -sfn "$d" ~/.agents/skills/$(basename "$d"); done
```