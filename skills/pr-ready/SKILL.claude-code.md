---
name: pr-ready
description: Mark PRs ready for review, assign GitHub reviewers from a team, and post to a daily Slack thread via MCP tools. Self-configures on first run.
---

Mark the current PR ready, prompt in chat to choose GitHub reviewers from a team, add reviewers with gh CLI, and post to a daily Slack thread via Slack MCP tools.

## Prerequisites

- `gh` CLI authenticated with access to your GitHub org
- Slack MCP tools available (`mcp__claude_ai_Slack__slack_*`)

## Configuration

All settings are stored in `~/.claude/pr-ready.json`. On first run, the agent must check if this file exists and is valid JSON.

**If the config file is missing or invalid**, run first-run setup:

1. **Auto-detect `github_org`**: Run `gh repo view --json owner --jq '.owner.login'` to get the org from the current repo. Present the detected value and ask the user to confirm or override.
2. **Auto-detect `github_team`**: Run `gh api "orgs/<github_org>/teams" --paginate --jq '.[].slug'` to list available teams. If there are 4 or fewer, use `AskUserQuestion`. Otherwise, list them and ask the user to type their choice.
3. **Ask for `channel_id`**: Ask the user for the Slack channel ID where daily PR threads are posted. Hint: they can find it by right-clicking a channel in Slack → "View channel details" → the ID is at the bottom.
4. **Ask for `thread_match`**: Ask the user what text pattern identifies the daily thread (e.g. `:pr:s for the day`). Suggest they copy a snippet from an existing thread message.

Write the config file and continue with the workflow.

**Config file format** (`~/.claude/pr-ready.json`):
```json
{
  "channel_id": "C08L8AXAE0M",
  "thread_match": ":pr:s for the day",
  "github_org": "vercel",
  "github_team": "support-platform"
}
```

## Agent Workflow (Required)

1. **Load config** from `~/.claude/pr-ready.json`. If missing, run first-run setup (see above).
2. Resolve current PR context (`gh pr view --json number,title,author,url,isDraft`).
3. Mark the PR as ready for review (`gh pr ready`). Skip if the PR is already marked ready.
4. Fetch candidate reviewers from the configured GitHub team:

```sh
gh api "orgs/<github_org>/teams/<github_team>/members" --paginate --jq '.[].login'
```

5. Exclude the PR author from candidates.
6. Prompt the user to choose 1+ reviewers (or `none`). If there are more than 4 candidates, list ALL candidates in chat first, then use `AskUserQuestion` with multiSelect showing up to 4 options — the user can select "Other" to type a name not shown. If there are 4 or fewer, use `AskUserQuestion` with multiSelect directly.
7. Add selected reviewers with `gh pr edit <number> --add-reviewer <login>`.
8. **Resolve Slack user IDs** for each selected reviewer:
   1. Get their display name: `gh api users/<login> --jq '.name'`
   2. Search Slack: `mcp__claude_ai_Slack__slack_search_users` with that name
   3. If multiple results, match by name. If no results, fall back to `<https://github.com/<login>|@<login>>` in the Slack message.
9. Post to the daily Slack thread using MCP tools (see Slack Posting below).
10. Copy the PR URL to clipboard with `pbcopy`.
11. Report outcome: PR ready status, reviewers added, Slack post result.

## Slack Posting via MCP Tools

Use the Slack MCP tools directly instead of calling a deployed endpoint.

Steps:

1. **Find the daily thread**: Use `mcp__claude_ai_Slack__slack_read_channel` with `channel_id` from config and `limit: 20`. Find the message whose text contains the `thread_match` pattern from config. Extract its `ts` (or `thread_ts`) as the thread timestamp.
2. **Check for duplicates**: Use `mcp__claude_ai_Slack__slack_read_thread` with the channel ID and the thread timestamp. Check if any reply already contains `api/pull/<prNumber>`. If so, skip posting and report it was deduped.
3. **Post the message**: Use `mcp__claude_ai_Slack__slack_send_message` with `channel_id`, `thread_ts`, and the formatted message.

## Posted Slack Format

```
:pr: <https://github.com/<org>/<repo>/pull/<number>|#<number>>: <title>
cc <@SLACK_USER_ID>, ...
```

- Derive the org and repo from the PR URL returned by `gh pr view`.
- Make the `#<number>` a Slack mrkdwn link to the GitHub PR URL.
- Do NOT include an Author line.
- Use Slack user mention syntax `<@SLACK_USER_ID>` for reviewers so they get pinged.
- If a reviewer's Slack ID can't be found, fall back to `<https://github.com/<login>|@<login>>`.
- If no reviewers were selected, omit the cc line entirely.
- No bullet points or list markers — just plain lines.
- Do NOT include a "Sent using Claude" line in the message.

## Important Behavior

- You MUST use Slack MCP tools (`mcp__claude_ai_Slack__slack_*`) for all Slack interactions. NEVER fall back to browser automation, Claude in Chrome (`mcp__claude-in-chrome__*`), or the `slack` skill. If the Slack MCP tools are not available, stop and tell the user.
- Do not rely on terminal `fzf` for agent-driven flows; always ask in chat first.
- If user chooses `none`, skip adding reviewers and omit the cc line.
- If the daily thread is not found, stop and tell the user.
- If the Slack post fails, show the error and do not claim success.
