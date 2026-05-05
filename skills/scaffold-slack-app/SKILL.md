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
4. Write the six generated files into the current directory using the templates below.
5. Print the post-generation summary.

---

## 1. Preconditions

Before asking any interview questions, perform these checks:

### 1.1 — Working directory check

Run `ls -A` in the current directory. Treat the directory as **empty enough** if the only entries are any subset of: `.git`, `.gitignore`, `.DS_Store`.

- If empty enough: proceed silently.
- If there are other entries: list them to the user and ask:
  > "This directory contains existing files: `{{list}}`. The skill will write `manifest.json`, `package.json`, `tsconfig.json`, `app.ts`, `.env.example`, `.gitignore`, and `SETUP.md` here, overwriting any with the same names. Continue? (yes/no)"
  - If `no`: stop. Tell the user to re-run from an empty directory.
  - If `yes`: proceed.
- To check the home-directory case, run `pwd` and compare against `$HOME`. If the directory contains more than 20 entries OR the absolute path equals `$HOME`: require explicit confirmation with this distinct prompt:
  > "This looks like a busy directory (`{{path}}` with `{{count}}` entries) — running the scaffold here is unusual. Continue? (yes/no)"
  - If `no`: stop.
  - If `yes`: proceed.

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
| `commands` | Receive slash command invocations | The bot handles `<comma-separated /commands>` |
| `im:history` | Read DM history with the bot | The bot processes DMs from users |
| `im:read` | View basic info about DMs | Required alongside `im:history` |
| `im:write` | Open DM conversations | The bot can DM users back |

Only include rows for scopes that actually appear in `bot_scopes`. The filtered rows of this table become the value substituted into `{{scope_table_rows}}` in section 4.

---

## 4. File templates

[Filled in Task 7]

---

## 5. Post-generation checklist

[Filled in Task 8]
