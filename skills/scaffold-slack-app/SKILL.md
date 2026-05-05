---
name: scaffold-slack-app
description: Use when starting a new Slack bot or Slack app project — runs an interview, generates a Slack app manifest with the right scopes/events, scaffolds a TypeScript Bolt starter in Socket Mode, and produces a SETUP.md with admin-request copy and exact dashboard navigation steps. Trigger phrases include "build a slack bot", "scaffold a slack app", "new slack integration", "create a slack app".
allowed-tools:
  - Read
  - Write
  - Bash
---

# Scaffold Slack App

Interview the user about a new Slack bot, then scaffold a working TypeScript Bolt starter (Socket Mode) into the current directory along with a `SETUP.md` containing dashboard navigation steps and a copy-paste admin-request message.

## When to use

Use this skill at the very start of a new Slack bot project, when the user has just created an empty directory (or is about to). Do NOT use this skill to modify an existing Slack project — v1 is scaffold-only.

## Procedure (high level)

1. Run preconditions check.
2. Conduct the interview (5 questions, asked one at a time).
3. Apply the answer→manifest mapping rules.
4. Write the seven generated files into the current directory using the templates below.
5. Print the post-generation summary.

---

## 1. Preconditions

Before asking any interview questions, perform these checks:

### 1.1 — Working directory check

Run `ls -A` in the current directory and `pwd && echo $HOME` to capture the path and home directory. Then route to ONE of these branches (in order — first match wins):

- **Busy or home directory** — if the absolute path equals `$HOME` OR the entry count exceeds 20, ask:
  > "This looks like a busy directory (`{{path}}` with `{{count}}` entries) — running the scaffold here is unusual. Continue? (yes/no)"
  - If `no`: stop.
  - If `yes`: proceed.
- **Non-empty (small)** — if the directory has entries beyond the allowlist `.git`, `.gitignore`, `.DS_Store`, ask:
  > "This directory contains existing files: `{{list}}`. The skill will write `manifest.json`, `package.json`, `tsconfig.json`, `app.ts`, `.env.example`, `.gitignore`, and `SETUP.md` here, overwriting any with the same names. Continue? (yes/no)"
  - If `no`: stop. Tell the user to re-run from an empty directory.
  - If `yes`: proceed.
- **Empty enough** — only allowlisted entries (or none): proceed silently.

### 1.2 — Git preflight

Check whether `.git/` exists in the current directory.

- If it exists: proceed silently.
- If it does not exist: ask:
  > "Initialize a git repository here? (yes/no, default yes)"
  - If `yes` or empty answer: remember this decision. The actual `git init -b main` runs in section 5.1, after all files are written.
  - If `no`: skip git init entirely.

### 1.3 — pnpm check (non-blocking)

Run `command -v pnpm`. If it returns nothing, remember this — at the end, the post-generation summary will include a one-line install hint:
> "pnpm not detected. Install it with `npm install -g pnpm` or `corepack enable` before running `pnpm install`."

Do NOT block generation on this; the user can install pnpm later.

---

## 2. Interview

Ask these questions one at a time, in order. Wait for the user's answer before asking the next question. Hold the answers in conversation context as a JSON-shaped "answers" object — you'll use it in section 3.

### Q1 — App identity

Ask:
> "Let's start with identity. I need three things:
> 1. **App name** (e.g., `Acme Bot`) — shown in Slack's app directory and install dialog.
> 2. **One-line description** (e.g., `Posts daily standup reminders`) — shown to admins approving the install.
> 3. **Bot user handle** (e.g., `acmebot`) — the @-handle users type to mention the bot. Lowercase, no spaces."

Record as `answers.identity = { name, description, bot_user_handle }`.

If the bot_user_handle contains spaces or uppercase: normalize (lowercase, replace spaces with `-`) and echo the normalized value back to the user before continuing.

### Q2 — Workspace audience

Ask:
> "Is this app for a single internal workspace (admin-installed), or distributable to many workspaces? (`internal` / `distributable`, default `internal`)"

Record as `answers.audience = "internal" | "distributable"`. Empty answer → `internal`. Any other answer: clarify and re-ask.

### Q3 — Receives DMs

Ask:
> "Should users be able to DM the bot directly? (yes/no)"

Record as `answers.dms = true | false`.

### Q4 — Responds to @mentions

Ask:
> "Should the bot respond when users @mention it in channels? (yes/no)"

Record as `answers.mentions = true | false`.

### Q5 — Slash commands

Ask:
> "Any slash commands? If yes, list them comma-separated (e.g., `/foo, /bar`). If none, say `no`."

Parse the answer:
- `no` / empty → `answers.slash_commands = []`.
- Otherwise: split on `,`, strip leading and trailing whitespace from each token. For each entry:
  - If it doesn't start with `/`, prepend `/`.
  - If it contains internal whitespace (e.g., `/bar baz`): treat as malformed, skip it, and collect into a `malformed` list.
- After parsing, if `malformed` is non-empty: echo back to the user:
  > "These entries look malformed (slash commands can't contain spaces): `<list>`. I'll skip them and use `<clean list>`. Continue? (yes/no)"
  - If `no`: re-ask Q5.
  - If `yes`: proceed with the clean list.

Record as `answers.slash_commands = ["/foo", "/bar", ...]`.

### Sanity gate — all-no answers

After Q5, if `answers.dms === false && answers.mentions === false && answers.slash_commands.length === 0`:
- Ask:
  > "You answered no to DMs, mentions, and slash commands — this generates a bot that does nothing. Continue anyway? (yes/no)"
  - If `no`: stop, do not generate. Suggest re-running.
  - If `yes`: proceed with baseline-only manifest.

---

## 3. Generation rules (answer → manifest)

Apply these rules to the `answers` object to derive a `manifest_data` object. The next section's templates consume `manifest_data`.

### 3.1 — Derived sets

Initialize:
```
bot_scopes = new Set()
bot_events = new Set()
slash_command_entries = []
org_deploy_enabled = false
```

### 3.2 — Mapping table

Apply each rule in order. Adding to a `Set` is idempotent (duplicates are silently deduped — this is how `chat:write` ends up listed once even when both DMs and Mentions add it).

| Condition | Action |
|---|---|
| Always | (nothing — baseline manifest below) |
| `answers.audience === "distributable"` | `org_deploy_enabled = true` |
| `answers.dms === true` | add `im:history`, `im:read`, `im:write`, `chat:write` to `bot_scopes`; add `message.im` to `bot_events` |
| `answers.mentions === true` | add `app_mentions:read`, `chat:write` to `bot_scopes`; add `app_mention` to `bot_events` |
| For each `cmd` in `answers.slash_commands` | add `commands` to `bot_scopes`; append `{ command: cmd, description: "TODO", usage_hint: "", should_escape: false }` to `slash_command_entries` |

### 3.3 — Final `manifest_data` shape

```
manifest_data = {
  display_information: {
    name: answers.identity.name,
    description: answers.identity.description,
  },
  features: {
    bot_user: {
      display_name: answers.identity.bot_user_handle,
      always_online: true,
    },
    slash_commands: slash_command_entries,   // ⚠️ OMIT this key entirely when slash_command_entries === [] — Slack rejects an empty array
  },
  oauth_config: {
    scopes: {
      bot: Array.from(bot_scopes).sort(),
    },
  },
  settings: {
    event_subscriptions: {
      bot_events: Array.from(bot_events).sort(),  // ⚠️ OMIT the entire `event_subscriptions` parent key when bot_events is empty
    },
    org_deploy_enabled: org_deploy_enabled,
    socket_mode_enabled: true,
    token_rotation_enabled: false,
  },
}
```

**Omission rules** (to keep the manifest clean):
- If `slash_command_entries` is empty, do not include the `features.slash_commands` key at all.
- If `bot_events` is empty, do not include the `settings.event_subscriptions` key at all.
- If `bot_scopes` is empty (the all-no path), include `oauth_config.scopes.bot: []`.

### 3.4 — Derived data for SETUP.md

Build a `scope_justifications` map for the SETUP.md scope table:

| Scope | What it allows | Why this app needs it |
|---|---|---|
| `app_mentions:read` | Receive `app_mention` events when users @mention the bot | The bot replies to @mentions |
| `chat:write` | Post messages as the bot | The bot sends replies and messages |
| `commands` | Receive slash command invocations | The bot handles the registered slash commands |
| `im:history` | Read DM history with the bot | The bot processes DMs from users |
| `im:read` | View basic info about DMs | Required alongside `im:history` |
| `im:write` | Open DM conversations | The bot can DM users back |

Only include rows for scopes that actually appear in `bot_scopes`. The filtered rows of this table become the value substituted into `{{scope_table_rows}}` in section 4.

---

## 4. File templates

Render the templates below by substituting `{{placeholders}}` from `answers` and `manifest_data`. Use the `Write` tool to write each file to the current directory. There are six template sections that produce seven files (Template 5 produces two: `.env.example` and `.gitignore`).

Substitution rules:
- `{{name}}` → `answers.identity.name`
- `{{description}}` → `answers.identity.description`
- `{{bot_user_handle}}` → `answers.identity.bot_user_handle`
- `{{package_name}}` → `answers.identity.bot_user_handle` (already normalized lowercase, no spaces)
- `{{manifest_json}}` → `JSON.stringify(manifest_data, null, 2)` — render with the omission rules from section 3.3 applied
- `{{handler_stubs}}` → concatenation of the per-capability code blocks listed in template 4 below
- `{{scope_table_rows}}` → markdown table rows from the filtered `scope_justifications` (section 3.4)
- `{{setup_step_extras}}` → conditional steps (slash commands UI, distribution UI) listed in template 6
- `{{cmd}}` → the current slash-command string (e.g., `/foo`); used only inside the per-command stub loop in template 4

**Escaping rule:** When substituting `{{name}}`, `{{bot_user_handle}}`, or `{{cmd}}` into a TypeScript string or template literal, escape backticks (`` ` `` → `` \` ``) and backslashes (`\` → `\\`) to avoid breaking the generated code.

### Template 1 — `manifest.json`

Write the file `manifest.json` with the contents of `{{manifest_json}}`. (The value is the pretty-printed JSON of `manifest_data` with the section 3.3 omission rules applied.)

### Template 2 — `package.json`

````json
{
  "name": "{{package_name}}",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "dev": "tsx watch app.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@slack/bolt": "4.7.2",
    "dotenv": "17.4.2"
  },
  "devDependencies": {
    "@types/node": "22.19.17",
    "tsx": "4.21.0",
    "typescript": "5.9.3"
  }
}
````

### Template 3 — `tsconfig.json`

````json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2023"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "outDir": "dist"
  },
  "include": ["app.ts"]
}
````

### Template 4 — `app.ts`

The template has a fixed prelude and a fixed startup block, with `{{handler_stubs}}` between them. Build `{{handler_stubs}}` by concatenating only the stubs whose capability the user enabled.

**Prelude (always emitted):**

````typescript
import "dotenv/config";
import bolt from "@slack/bolt";

const { App } = bolt;

const token = process.env["SLACK_BOT_TOKEN"];
const appToken = process.env["SLACK_APP_TOKEN"];
if (!token || !appToken) {
  throw new Error("SLACK_BOT_TOKEN and SLACK_APP_TOKEN must be set in .env (see SETUP.md).");
}

const app = new App({ token, appToken, socketMode: true });
````

**Mention stub (emit if `answers.mentions === true`):**

````typescript
app.event("app_mention", async ({ event, say }) => {
  await say({
    thread_ts: event.ts,
    text: `Hi <@${event.user}>! I'm {{bot_user_handle}}. Replace this stub in app.ts.`,
  });
});
````

**DM stub (emit if `answers.dms === true`):**

````typescript
app.message(async ({ message, say }) => {
  if (!("channel_type" in message) || message.channel_type !== "im") return;
  if ("subtype" in message && message.subtype) return;
  await say("Got your DM. Replace this stub in app.ts to add real logic.");
});
````

**Slash-command stubs (emit one per command in `answers.slash_commands`):**

For each `cmd` (e.g., `/foo`):

````typescript
app.command("{{cmd}}", async ({ ack, respond }) => {
  await ack();
  await respond(`{{cmd}} received. Replace this stub in app.ts.`);
});
````

**Startup block (always emitted, last):**

````typescript
const port = Number(process.env["PORT"] ?? 3000);
await app.start(port);
// Replace with your preferred logger before deploying.
console.log(`⚡️ {{name}} is running (Socket Mode)`);
````

### Template 5 — `.env.example` and `.gitignore`

Two files. Write both.

`.env.example`:

```
SLACK_BOT_TOKEN=xoxb-replace-me
SLACK_APP_TOKEN=xapp-replace-me
```

`.gitignore` — use the `Read` tool to check whether the file already exists and what it contains. If absent, `Write` it with the lines below. If present, `Write` it with the existing content plus any of the lines below that are not already in it.

```
.env
node_modules/
dist/
.DS_Store
```

### Template 6 — `SETUP.md`

The template branches on `answers.audience` for the admin-request copy and on `answers.slash_commands.length > 0` for an extra setup step.

````markdown
# {{name}} — Setup

## What this app does

{{description}}

## Scopes requested and why

| Scope | What it allows | Why this app needs it |
| --- | --- | --- |
{{scope_table_rows}}

## Workspace-admin request

<!-- INTERNAL ONLY: emit this block when audience === "internal", omit when "distributable". Strip these HTML comments from the output. -->

> Hi! I'd like to install a Slack app called **{{name}}** in our workspace.
>
> **What it does:** {{description}}
>
> **Scopes requested:** see the table above.
>
> The app's manifest is in the attached `manifest.json`. To install:
> 1. Go to <https://api.slack.com/apps> → **Create New App** → **From a manifest**.
> 2. Pick our workspace, paste the manifest, click **Create**.
> 3. Click **Settings → Install App → Install to Workspace** and approve the scopes.
> 4. Send me back the **Bot User OAuth Token** (starts with `xoxb-`).
>
> Happy to walk through it together if useful.

<!-- DISTRIBUTABLE ONLY: emit this block when audience === "distributable", omit when "internal". Strip these HTML comments from the output. -->

> This app is set up for distribution to multiple workspaces. To list it publicly:
> 1. Go to <https://api.slack.com/apps> → your app → **Settings → Manage Distribution**.
> 2. Complete the checklist (icon, support email, etc.) and submit for review.
> 3. Slack's review process typically takes several business days.
>
> For per-workspace installs by individual admins, direct workspace admins to your hosted install URL once available.

## Setup steps

### 1. Create the app from the manifest

1. Go to <https://api.slack.com/apps> and click **Create New App**.
2. Choose **From a manifest** → select your workspace.
3. Paste the contents of `manifest.json` and click **Next** → **Create**.

### 2. Install the app to your workspace

1. In the left sidebar, click **Settings → Install App**.
2. Click **Install to Workspace** and approve the scopes.
3. (For internal workspace bots: if you are not an admin, your admin must do this step after they approve your request.)

### 3. Get your bot token (`SLACK_BOT_TOKEN`)

1. After install, the **Install App** page shows a **Bot User OAuth Token** starting with `xoxb-`.
2. Copy it and paste into `.env` as `SLACK_BOT_TOKEN=xoxb-...`.

### 4. Get your app-level token (`SLACK_APP_TOKEN`) — required for Socket Mode

1. In the left sidebar, click **Settings → Basic Information**.
2. Scroll down to **App-Level Tokens** and click **Generate Token and Scopes**.
3. Name it (e.g., `socket-mode`), click **Add Scope**, add `connections:write`.
4. Click **Generate**. Copy the token starting with `xapp-`.
5. Paste into `.env` as `SLACK_APP_TOKEN=xapp-...`.

### 5. Enable Socket Mode

1. In the left sidebar, click **Settings → Socket Mode**.
2. Toggle **Enable Socket Mode** on.
   (The manifest already declares this, but the toggle must be flipped in the UI for the connection to accept your app token.)

{{setup_step_extras}}

### Final — Run the bot

```
cp .env.example .env
# fill in the two tokens
pnpm install
pnpm dev
```

---

> Slack dashboard labels can shift over time. If a path here is stale, the canonical reference is <https://api.slack.com/start/overview>.
````

**`{{setup_step_extras}}` rules** (concatenate in this order, omitting any whose condition is false):

If `answers.slash_commands.length > 0`:

````markdown
### 6. Edit slash command metadata (optional)

1. In the left sidebar, click **Features → Slash Commands**.
2. For each command, click the pencil icon and fill in **Short Description** and **Usage Hint** (the manifest leaves these as placeholders).
````

If `answers.audience === "distributable"`:

````markdown
### 7. Set up public distribution (optional)

1. In the left sidebar, click **Settings → Manage Distribution**.
2. Complete the checklist (icon, support email, OAuth redirect URLs).
3. Submit for review when ready.
````

---

## 5. Post-generation checklist

After all seven files are written:

### 5.1 — Run conditional follow-ups

- If the user said `yes` to git-init in section 1.2 and `.git/` does not exist: run `git init -b main`.
- Do NOT run `pnpm install` automatically. The user runs it.

### 5.2 — Print the chat summary

Print exactly this template, substituting the bracketed values:

```
✅ Scaffolded {{name}} into the current directory.

Files written:
  - manifest.json       (Slack app manifest)
  - package.json        (Node 22, pnpm)
  - tsconfig.json       (strict TypeScript)
  - app.ts              (Bolt handler stubs)
  - .env.example        (token placeholders)
  - .gitignore
  - SETUP.md            ← read this next

Next steps:
  1. Open SETUP.md — it has the admin-request message and the exact dashboard
     navigation for getting your tokens.
  2. cp .env.example .env, fill in the two tokens.
  3. pnpm install && pnpm dev
```

If pnpm was not detected in section 1.3, append:

```

  ⚠️ pnpm not detected. Install it with `npm install -g pnpm` or `corepack enable`.
```

If the user opted into git-init, append:

```

  ✓ Initialized a fresh git repo. Make your first commit when you're ready.
```
