# claude-skills

A collection of [Claude Code](https://claude.ai/code) skills — slash commands that run inside Claude Code sessions.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| [api-e2e](./api-e2e/) | `/api-e2e` | Diff branch, extract affected API endpoints, run curl tests, report pass/fail |

## Installation

### Option A — Claude Code CLI

```bash
# Install a specific skill
claude skills install https://github.com/kmtusher97/claude-skills/tree/main/api-e2e
```

### Option B — Manual (one skill)

```bash
SKILL=api-e2e
mkdir -p ~/.claude/skills/$SKILL
curl -sSL https://raw.githubusercontent.com/kmtusher97/claude-skills/main/$SKILL/SKILL.md \
  -o ~/.claude/skills/$SKILL/SKILL.md
```

### Option C — Clone and symlink (all skills)

```bash
git clone https://github.com/kmtusher97/claude-skills ~/claude-skills
# Then symlink whichever skills you want:
ln -s ~/claude-skills/api-e2e ~/.claude/skills/api-e2e
```

## Usage

Once installed, invoke any skill inside a Claude Code session:

```
/api-e2e
/api-e2e origin/dev     # diff against a custom base branch
```

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code)
- `curl` and `python3` in your shell

## License

MIT
