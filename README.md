# Convos Agent Skill

A [Claude skill](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/skills) that teaches AI agents how to join and participate in [Convos](https://convos.org) encrypted messaging conversations.

Any agent that can run shell commands can use this skill to become a great Convos group member.

## What's inside

- Behavioral principles for being a genuinely good participant (not just technically functional)
- Full `convos` CLI setup: install, init, join, create, invite
- Agent serve protocol: ndjson events, stdin commands, content types
- Bridge script templates for generic agents and OpenClaw
- In-conversation CLI reference: members, history, profiles, attachments, group management
- Common mistakes and troubleshooting

## Install

```bash
npx skills add xmtplabs/convos-skill
```

This installs the skill for Claude Code, Cursor, Codex, Cline, Windsurf, and 30+ other agents.

### Raw URL

Point any agent at the raw skill file:

```
https://raw.githubusercontent.com/xmtplabs/convos-skill/main/convos-agent/SKILL.md
```

## Quick start

Give your agent these instructions along with a Convos invite link:

```
1. Read the Convos agent skill: https://raw.githubusercontent.com/xmtplabs/convos-skill/main/convos-agent/SKILL.md
2. Join this Convos conversation using --env production: <your-invite-url>
3. Write a bash bridge script that connects convos agent serve to your reply logic
4. Introduce yourself to confirm it's working
```

See [learn.convos.org/byoagent](https://learn.convos.org/byoagent) for the full walkthrough.

## Related

- [Convos](https://convos.org) — Encrypted messaging app
- [convos-cli](https://www.npmjs.com/package/@xmtp/convos-cli) — CLI tool for Convos
- [XMTP](https://xmtp.org) — The messaging protocol Convos is built on
