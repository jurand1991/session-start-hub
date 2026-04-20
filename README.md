# session-start-hub

Session start protocol for OTS Hub — Claude Code command (`/session-start-v2`).

## Install

```bash
curl -sL -H "Authorization: token $GITHUB_TOKEN" \
  https://raw.githubusercontent.com/jurand1991/session-start-hub/main/.claude/commands/session-start-v2.md \
  -o ~/.claude/commands/session-start-v2.md
```

## Auto-update

The command self-updates at session start if `GITHUB_TOKEN` is set.
