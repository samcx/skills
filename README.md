# Skills

A collection of reusable skills for AI coding agents.

## Available Skills

| Skill | Description |
|-------|-------------|
| [pr-ready](skills/pr-ready/SKILL.md) | Mark PRs ready for review, assign GitHub reviewers from a team, and post to a daily Slack thread |

## Installation

Install a skill using the [skills CLI](https://skills.sh):

```sh
npx skills add samcx/skills --skill pr-ready -a claude-code
# or
pnpx skills add samcx/skills --skill pr-ready -a claude-code
# or
bunx skills add samcx/skills --skill pr-ready -a claude-code
```

## Agent Support

### pr-ready

| Agent | Status |
|-------|--------|
| Claude Code | ✅ |
| Codex CLI | 🔜 |
| Other agents | ✖️ |

**Prerequisites:**
- [`gh` CLI](https://cli.github.com/) — authenticated with access to your GitHub org
- [Slack MCP tools](https://modelcontextprotocol.io/) — for posting to Slack threads
