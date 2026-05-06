# scaffold-slack-app

A Claude Code skill that scaffolds a TypeScript Slack Bolt app (Socket Mode) into the current directory after a short interview. Generates the Slack app manifest with the right scopes, a Bolt starter with handler stubs, and a `SETUP.md` with the exact Slack dashboard navigation for retrieving tokens — plus a copy-paste workspace-admin request.

## Why

Creating a Slack bot for a company workspace is friction-heavy: you have to know which scopes to ask for up front, navigate a dashboard whose labels keep shifting, and write a justification message to your workspace admin every time you add a scope. This skill compresses all of that into a 5-question interview, so you can skip straight to writing bot logic.

## Install

### As a Claude Code plugin (recommended)

```
/plugin marketplace add rilwan99/scaffold-slack-app
/plugin install scaffold-slack-app
```

### Manual

Copy `skills/scaffold-slack-app/` into your `~/.claude/skills/` directory:

```bash
git clone https://github.com/rilwan99/scaffold-slack-app
cp -R scaffold-slack-app/skills/scaffold-slack-app ~/.claude/skills/
```

## Usage

In an empty directory (or one with only `.git`/`.gitignore`):

```
> build a slack bot
```

Claude will pick up the trigger, walk you through five questions, and write the project files. Then read the generated `SETUP.md` for next steps.

## What it generates

- `manifest.json` — Slack app manifest with the scopes/events derived from your answers.
- `package.json`, `tsconfig.json` — Node 22, strict TypeScript, pnpm.
- `app.ts` — Bolt App in Socket Mode with stub handlers per capability.
- `.env.example`, `.gitignore`.
- `SETUP.md` — admin-request copy + scope justification table + numbered dashboard steps with exact navigation paths.

## What v1 does NOT do

- HTTP transport mode (Socket Mode only).
- Re-runs / scope-drift detection.
- Interactivity (modals, buttons, shortcuts).
- App home, workflow steps, file handling.
- Sensitive scopes (`channels:history`, `groups:history`).
- Languages other than TypeScript.

These are deliberate v1 omissions to keep the skill small and focused.

## Manual smoke tests

After making changes to `SKILL.md`, run these by hand in three throwaway directories:

1. **all-yes:** answer yes to DMs, mentions, and slash commands `/standup, /skip`. Verify generated `manifest.json` matches `tests/smoke/all-yes/expected/manifest.json` and `pnpm install && pnpm typecheck` succeed.
2. **mentions-only:** only mentions = yes. Verify generated manifest matches `tests/smoke/mentions-only/expected/manifest.json`.
3. **all-no:** all capability questions = no, confirm the sanity gate fires, answer yes to proceed. Verify manifest matches `tests/smoke/all-no/expected/manifest.json`.

Diff with `diff <(jq -S . manifest.json) <(jq -S . tests/smoke/<scenario>/expected/manifest.json)` to ignore key ordering.

## License

MIT.
