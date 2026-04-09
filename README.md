# claude-skills

A collection of [Claude Code](https://claude.ai/code) skills — slash commands that run inside Claude Code sessions.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [api-e2e](./plugins/api-e2e/) | `/api-e2e:api-e2e` | Diff branch, extract affected API endpoints, run curl tests, report pass/fail |

## Installation

### Option A — Plugin marketplace (recommended)

```
/plugin marketplace add kmtusher97/claude-skills
/plugin install api-e2e@kmtusher97-claude-skills
```

### Option B — Manual (one skill)

```bash
SKILL=api-e2e
mkdir -p ~/.claude/skills/$SKILL
curl -sSL https://raw.githubusercontent.com/kmtusher97/claude-skills/main/plugins/$SKILL/skills/$SKILL/SKILL.md \
  -o ~/.claude/skills/$SKILL/SKILL.md
```

### Option C — Clone and symlink (all skills)

```bash
git clone https://github.com/kmtusher97/claude-skills ~/claude-skills
ln -s ~/claude-skills/plugins/api-e2e/skills/api-e2e ~/.claude/skills/api-e2e
```

## Usage

Once installed via the plugin marketplace, invoke any skill inside a Claude Code session:

```
/api-e2e:api-e2e
/api-e2e:api-e2e origin/dev     # diff against a custom base branch
```

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- `curl` and `python3` in your shell

## License

MIT
