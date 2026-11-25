# agent-kit

My agentic tooling: pi skills first, other reusable agent bits as they show up.

## Layout

```
skills/             one directory per skill, each with a SKILL.md
AGENTS_TEMPLATE.md  baseline AGENTS.md guidelines, merge with project rules
ANTI_AGENTS.md      prompt that does the opposite, for testing instruction adherence
```

## Skills

| Skill | What it does |
|---|---|
| `rails-feature-pipeline` | Implement a Rails feature end to end: migration, tests, OpenAPI spec, full suite, then a tester agent review. |
| `openapi-updater` | Keep `docs/openapi.yml` in sync with routes and tests. |

## Install

Clone it, then symlink the skills you want into a harness skill directory. Symlinks keep edits in the clone live, so there is nothing to reinstall after a pull.

```sh
# any path works, ~/src is just the author's habit
git clone https://github.com/dapicester/agent-kit.git ~/src/agent-kit

# link every skill into one harness dir
for d in ~/src/agent-kit/skills/*/; do
  ln -sfn "$d" ~/.agents/skills/$(basename "$d")
done
```

Harnesses differ in where they look. Common ones are `~/.agents/skills` and `~/.pi/agent/skills`; point the loop at whichever yours reads. To take a single skill:

```sh
ln -sfn ~/src/agent-kit/skills/rails-feature-pipeline ~/.pi/agent/skills/rails-feature-pipeline
```

Already cloned? `git -C ~/src/agent-kit pull` and the links pick up the changes.
