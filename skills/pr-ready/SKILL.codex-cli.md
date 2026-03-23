---
name: pr-ready
description: Mark the current PR ready, prompt in chat to choose GitHub reviewers from a team, add reviewers with gh CLI, and post to the configured daily Slack PR thread using the existing `pr-ready-codex` session.
---

# PR Ready

Use this skill when the user asks to mark a PR ready and post the support-thread update.

Use the existing `pr-ready-codex` session for Slack posting.

## Configuration

All settings are stored in `~/.codex/pr-ready.json`. At the start of the workflow, read this file and ensure it is valid JSON before continuing.

If the config file is missing or invalid, run first-run setup:

1. Auto-detect `github_org` with `gh repo view --json owner --jq '.owner.login'`. Present the detected value and ask the user to confirm or override it.
2. Auto-detect `github_team` by running `gh api "orgs/<github_org>/teams" --paginate --jq '.[].slug'`.
   - If there are 4 or fewer teams, list them and ask the user to choose in chat.
   - If there are more than 4 teams, list all of them in chat and ask the user to type the team slug they want.
3. Ask the user for `channel_id`, with a hint that they can find it in Slack channel details.
4. Ask the user for `thread_match`, with a hint that they can copy a distinctive snippet from the daily PR thread starter.
5. Write `~/.codex/pr-ready.json` and continue with the workflow.

Expected format:

```json
{
  "channel_id": "C08L8AXAE0M",
  "thread_match": ":pr:s for the day",
  "github_org": "vercel",
  "github_team": "support-platform"
}
```

## Preconditions

- Slack must already be logged in in the existing `pr-ready-codex` browser session.
- Treat shell-available `agent-browser` as sufficient browser automation availability. If browser control is not exposed as a first-class tool, probe the shell for `agent-browser` before declaring the Slack step blocked.
- Do not automate Slack login. If the session is unavailable, not live, or logged out, stop and tell the user.
- Do not silently switch to another Slack browser session if `pr-ready-codex` is unavailable.
- Do not assume a listed session name is sufficient. `pr-ready-codex` must also open into a usable Slack app view, not a browser-handoff or redirect page.
- Treat `about:blank`, `Find your workspace | Slack`, and Slack `workspace-signin` URLs as logged-out or unusable states even if `agent-browser session list` still shows `pr-ready-codex`.
- If the session is on Slack's desktop-app handoff page but shows `open this link in your browser`, recover it in-place by clicking that browser fallback link once, then rerun the usability checks.
- If the user needs a visible browser for manual recovery and `--headed` is ignored because the daemon is already running, tell them to run `agent-browser --session pr-ready-codex close || true`, then `pkill -f agent-browser || true`, then relaunch with `agent-browser --session pr-ready-codex --headed open https://app.slack.com/client`.
- For the Slack posting step, use the existing `pr-ready-codex` session directly.

## Agent Workflow (Required)

1. Load config from `~/.codex/pr-ready.json`. If the file is missing or invalid, run first-run setup (see above).
2. Resolve current PR context (`gh pr view --json number,title,author,url`).
3. Mark the PR as ready for review (`gh pr ready`).
4. Fetch candidate reviewers from the configured GitHub team:

```sh
gh api "orgs/<github_org>/teams/<github_team>/members" --paginate --jq '.[].login'
```

5. Exclude the PR author from candidates.
6. Present the possible GitHub reviewers **in chat** before asking for a choice.
7. Prompt the user **in chat** to choose 1+ GitHub usernames (or `none`).
8. On reruns or retries, ask again instead of inferring reviewer intent from the current PR state.
9. Add selected reviewers with `gh pr edit <number> --add-reviewer <login>`.
10. Resolve Slack reviewer labels for each selected reviewer:
    - get their display name with `gh api users/<login> --jq '.name'`
    - use the GitHub display name as the primary Slack lookup text when it is present
    - if the GitHub display name is empty, fall back to the GitHub login
    - if Slack shows a candidate matching the intended reviewer, select that exact candidate before considering any fallback
11. Post to the daily Slack thread using the existing `pr-ready-codex` session (see Slack Posting below).
12. Copy the PR URL to clipboard with `pbcopy`.
13. Report outcome: PR ready status, reviewers added, Slack post result.

## Reviewer Mention Resolution

Resolve reviewer mentions dynamically instead of using a fixed map.

- For each selected GitHub reviewer, get their GitHub display name with `gh api users/<login> --jq '.name'`.
- Use the GitHub display name as the primary Slack lookup text. Only use the GitHub login when the display name is missing.
- In the Slack composer, open the mention picker or type `@<display name>` and keep filtering until the intended reviewer is unique or clearly visible.
- Match the intended reviewer using the full real name or the Slack option `aria-label`, not just the visible handle.
- If the intended reviewer is visible in the picker, selecting that exact option is required before any fallback.
- If the picker shows multiple candidates, continue filtering instead of guessing.
- After selection, verify the draft DOM contains a real Slack mention node for that reviewer, not plain `@text`.
- Only fall back to `<https://github.com/<login>|@<login>>` after attempting Slack mention resolution and failing to identify the intended reviewer in the picker.

## Slack Posting via Browser Automation

Use the existing `pr-ready-codex` session for this entire section.

Channel: the configured Slack channel from `~/.codex/pr-ready.json`
Daily thread match: message text containing the configured `thread_match` value from `~/.codex/pr-ready.json`

Steps:

1. Use the existing Slack browser session `pr-ready-codex`.
2. Probe browser availability in this order before stopping:
    - if browser automation is available directly, use it
    - otherwise run `agent-browser session list` from the shell and continue if it works
    - if `agent-browser` is missing or fails to run, then stop and report the Slack step is unavailable
3. If the session is unavailable, not live, or logged out after that probe, stop and tell the user that `pr-ready-codex` needs to be open and logged in first.
4. Run a Slack readiness preflight before posting:
    - confirm `pr-ready-codex` appears in `agent-browser session list` when using the shell fallback
    - fetch the page title with `agent-browser --session pr-ready-codex get title`
    - fetch the current URL with `agent-browser --session pr-ready-codex get url`
    - fetch a snapshot with `agent-browser --session pr-ready-codex snapshot ...`
    - verify the page title is not `Redirecting… | Slack` or similar browser-handoff text
    - verify the current URL is not `about:blank` and is not a Slack `workspace-signin` URL
    - verify the page title is not `Find your workspace | Slack`
    - verify the snapshot does not show `open this link in your browser`
    - verify the snapshot shows real Slack app UI such as the workspace shell, channel list, or message pane
    - if the session is on the browser-handoff page and exposes `open this link in your browser`, click that link in the same `pr-ready-codex` session, wait for navigation, and rerun the preflight once
    - if the session is on `about:blank`, `Find your workspace | Slack`, or a Slack `workspace-signin` URL and `/Users/samcx/.codex/memories/pr-ready-codex-slack-state.json` exists, run `agent-browser --session pr-ready-codex state load /Users/samcx/.codex/memories/pr-ready-codex-slack-state.json`, reopen `https://app.slack.com/client`, wait for navigation, and rerun the preflight once
    - after recovery, prefer the real web-client route (for example `app.slack.com/client/...`) and continue only if the snapshot now shows real Slack app UI
    - if the recovery step is unavailable or the rerun still fails, stop and tell the user that `pr-ready-codex` exists but is not in a usable Slack state
5. Navigate to the configured Slack channel in that session.
6. Re-snapshot after each navigation step.
   - Treat `@e...` refs as unstable across snapshots, menus, dialogs, autocomplete popovers, and thread reloads. Reacquire refs after each UI-changing action instead of reusing old refs blindly.
7. Find the daily thread starter whose visible text contains the configured `thread_match` value.
8. Open the thread and inspect replies for `#<prNumber>:`. Deduplicate only if the matching reply is clearly the current user's previously posted PR-ready reply for the current PR. Ignore unrelated replies, cross-posts, or other users' messages that happen to mention the same PR number.
9. If no duplicate exists, verify the thread reply textbox is empty before drafting. If it is not empty, clear it and confirm the thread reply `Send now` button is disabled before continuing. Always compose a new reply from the thread textbox; do not start from editing an existing message.
10. Resolve a stable thread-pane container for the open thread and scope all composer, send-button, and message-query actions to that container. After any UI-changing action, reacquire refs from that same container before continuing.
11. Reply in the thread with the formatted message below.
12. Treat the Slack reply composer as a Quill-style rich-text `contenteditable` editor with its own document model, not a plain textarea.
13. Preferred composition flow:
    - First build the draft body using Slack's editor behavior, not raw DOM replacement.
    - Preferred implementation order for the first line:
      1. insert literal `:pr: `
      2. create an inline anchor for visible `#<number>` pointing to the PR URL
      3. insert literal `: <title>` in the same paragraph
      4. insert a paragraph break
      5. insert `cc ` and resolve reviewer mentions
    - Insert a real clickable link on `#<number>` pointing to the PR URL returned by `gh pr view`.
    - Insert the PR title as literal text nodes, not by typing the full message line-by-line.
    - If the title contains `@`, `#`, or similar tokens, prevent Slack auto-linking by using literal text nodes or zero-width separators so only intended reviewer mentions resolve.
    - If reviewers were selected, leave the caret at the end of `cc `, then use the GitHub display name as the primary lookup text in Slack mention autocomplete.
    - If the intended reviewer appears in the picker, select that exact option. Do not fall back while a matching reviewer is visible.
    - Do not stop at visible `@name` text. Verify Slack shows the intended reviewer, select it explicitly, and confirm the draft DOM contains a real mention node for that reviewer before sending.
14. Do not use multiline typing into the Slack rich-text composer. Do not build the full draft incrementally with newline typing.
15. After mentions resolve, verify the draft DOM still shows:
    - linked `#<number>`
    - `#<number>` inline with `:pr:` and the title on the same first paragraph, not split across separate paragraphs
    - unchanged title text with no Slack-created mention nodes inside the title
    - only the intended reviewer mentions as Slack mention nodes
    - no raw GitHub URL line unless you are intentionally using the emergency fallback
16. If a reviewer mention does not resolve to a unique Slack mention after attempting Slack autocomplete, fall back to `<https://github.com/<login>|@<login>>` for that reviewer and continue.
17. After sending, re-snapshot and verify the posted reply shows:
    - exactly one new reply for the PR
    - a linked `#<number>` node
    - a real line break between the PR line and the `cc` line when reviewers were included
    - Slack mention nodes for reviewers when reviewer mentions were intended
    - no raw GitHub URL line unless you intentionally used the emergency fallback
18. Treat send verification as a hard gate. If you cannot identify exactly one newly posted reply authored by the current user for the current PR, stop. Do not edit or delete anything except a conclusively identified malformed reply that you just posted for this exact PR in this exact thread.
19. After sending, also verify the thread reply textbox is empty and no lingering unsent draft remains, then save the session state to `/Users/samcx/.codex/memories/pr-ready-codex-slack-state.json` with `agent-browser --session pr-ready-codex state save /Users/samcx/.codex/memories/pr-ready-codex-slack-state.json`.
20. If a partial message is posted accidentally, only edit or delete it if all of the following are true:
    - it is in the current thread
    - it is authored by the current user
    - it contains the current `#<prNumber>:` marker
    - it matches the just-posted malformed reply you are recovering
   Otherwise stop, leave existing messages untouched, and report failure.
21. If the daily thread is not found, stop and tell the user. Do not post a top-level channel message.

## Slack Composer Safety

- Never use `agent-browser type` or `fill` with multiline content in a Slack rich-text composer. Slack may send the first line immediately and leave the rest as a lingering draft.
- Never set Slack composer content with raw `innerHTML`, `outerHTML`, or whole-node `textContent` replacement. That can bypass Slack's internal editor model and produce malformed posts.
- If Slack opens an unexpected dialog or popover while composing, such as `Add link`, close it, re-snapshot, and reacquire refs before continuing.
- Before navigating or drafting, verify the session is in a real Slack app view. A session parked on `Redirecting… | Slack`, `about:blank`, `Find your workspace | Slack`, a Slack `workspace-signin` URL, or a page containing `open this link in your browser` is not usable for `pr-ready`.
- If the session is on Slack's handoff page and exposes `open this link in your browser`, use that in-page browser fallback once before declaring the session unusable.
- If the session is on `about:blank` or a Slack sign-in page and a saved state exists at `/Users/samcx/.codex/memories/pr-ready-codex-slack-state.json`, load that state and reopen `https://app.slack.com/client` before declaring the session unusable.
- If the user needs to recover the session manually in a visible window, close the session, kill the daemon, and relaunch with `agent-browser --session pr-ready-codex --headed open https://app.slack.com/client`.
- Before composing, verify the thread reply textbox is empty. If it is not, clear it first and confirm the thread reply `Send now` button is disabled before continuing.
- Resolve the thread reply textbox, send button, and any message actions from the thread-pane container. Never click a global `Send now`, `More options`, or similarly labeled control when multiple matching controls are present.
- Prefer posting one new reply from the thread composer. Do not use ArrowUp edit mode or edit-in-place flows for normal retries.
- Compose the body in two phases:
  - draft the linked PR number and literal title via DOM/eval
  - resolve reviewer mentions afterward via Slack autocomplete
- If you use `agent-browser eval`, use it only for narrowly-scoped editor interactions. Do not replace the entire draft DOM.
- If Slack's toolbar link UI is flaky, prefer narrowly-scoped editor operations such as `createLink` or targeted `insertHTML` for the inline `#<number>` anchor rather than falling back immediately. If one inline-link attempt fails verification, stop retrying rich-link composition and use the emergency fallback or report failure.
- Do not treat `agent-browser get text` as sufficient validation for this workflow. Visible text can hide bad internal formatting.
- When the PR title contains `@`, `#`, or similar tokens, insert the title as literal text so Slack does not convert parts of the title into mentions or links.
- Resolve reviewer mentions only after the PR-number link and literal title text are already in place.
- Use the GitHub display name as the primary Slack lookup text. Do not use the GitHub login first when a display name exists.
- When resolving reviewer mentions, require both:
  - an explicit selection of the intended reviewer from Slack's picker
  - a real mention node in the draft DOM after selection
- If the intended reviewer is visible in Slack's picker, fallback to a GitHub link is not allowed.
- Before sending, verify the draft DOM shows:
  - a linked `#<number>` node
  - the first paragraph has the form `:pr: <linked #<number>>: <title>`
  - no `ts-mention` nodes inside the title
  - reviewer mentions as `ts-mention` nodes when they were intended to resolve
  - if the fallback format is being used intentionally, the raw PR URL must be on its own line and not merged into the title or `#<number>` token
- After sending, verify the posted reply DOM shows:
  - exactly one new reply for the PR
  - a linked `#<number>` node
  - a real `<br>` or equivalent line break between the PR line and the `cc` line when reviewers were included
  - reviewer mentions rendered as Slack mention/link nodes with reviewer identity metadata when reviewer mentions were intended
  - no raw GitHub URL line unless you intentionally used the emergency fallback
- Do not treat accessibility snapshots alone as the source of truth for line breaks after sending. Slack's accessibility tree can flatten visible text; use posted-message DOM inspection as the primary validation signal.
- Treat hover-gated message menus as last-resort recovery tools. Do not open a message menu unless you have already identified the exact current-user PR reply node you intend to act on.
- After sending, also verify the thread reply textbox is empty and there is no lingering unsent draft. If the composer is still populated after a send attempt, treat the send as failed and stop.
- Do not edit or delete any message unless you have positively matched it as the current user's malformed reply for the current PR in the current thread.
- If a partial or malformed message is posted accidentally, prefer the emergency fallback or a clean new retry from the reply textbox. Only use edit/delete recovery when the current-user identity and exact target message have been verified first.

## Posted Slack Format

Required format:

```text
:pr: <https://github.com/<org>/<repo>/pull/<number>|#<number>>: <title>
cc @Reviewer One @Reviewer Two
```

If the Slack composer does not preserve the mrkdwn syntax when typed literally, use the editor UI to link the visible `#<number>` text to the PR URL.

Emergency fallback only if the link UI fails after one retry:

```text
:pr: #<number>: <title>
<https://github.com/<org>/<repo>/pull/<number>>
cc @Reviewer One @Reviewer Two
```

- Do not include an Author line.
- Derive `<org>` and `<repo>` from the PR URL returned by `gh pr view`.
- Treat the PR title as plain text in the composer. For example, `[@api/foo]` must remain literal text, not a Slack mention.
- If no reviewers were selected, omit the `cc` line entirely.
- Do not use bullets or list markers.
- Keep the PR number marker in the message as `#<number>:` so dedupe remains reliable.
- If you must use the emergency fallback, report that the Slack message is degraded instead of claiming the formatting is correct.

## Important Behavior

- Do not rely on terminal `fzf` for agent-driven flows; always ask in chat first.
- If the user chooses `none`, skip adding reviewers and post without the `cc` line.
- If the task is a rerun of `pr-ready`, do not silently reuse or silently clear reviewer intent. Ask again.
- If Slack is unavailable, logged out, or the daily thread is not found, stop and tell the user instead of guessing.
- If post-send verification fails, leave unrelated messages untouched, keep or clear only the current draft, and report the exact verification failure.
- If the Slack post fails midway through browser automation, report the exact step that failed and do not claim success.
