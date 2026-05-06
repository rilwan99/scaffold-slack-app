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

## License

MIT.
