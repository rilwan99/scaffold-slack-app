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
  > "This directory contains existing files: `<list>`. The skill will write `manifest.json`, `package.json`, `tsconfig.json`, `app.ts`, `.env.example`, `.gitignore`, and `SETUP.md` here, overwriting any with the same names. Continue? (yes/no)"
  - If `no`: stop. Tell the user to re-run from an empty directory.
  - If `yes`: proceed.
- If the directory contains more than 20 entries OR the absolute path resolves to the user's home (`$HOME`): require explicit confirmation regardless of contents. Same yes/no prompt.

### 1.2 — Git preflight

Check whether `.git/` exists in the current directory.

- If it exists: proceed silently.
- If it does not exist: ask:
  > "Initialize a git repository here? (yes/no, default yes)"
  - If `yes` or empty answer: run `git init -b main` after the interview, before writing files (so the first commit can include the generated files in a follow-up). Do NOT run `git init` yet.
  - If `no`: skip git init entirely.

### 1.3 — pnpm check (non-blocking)

Run `command -v pnpm`. If it returns nothing, remember this — at the end, the post-generation summary will include a one-line install hint:
> "pnpm not detected. Install it with `npm install -g pnpm` or `corepack enable` before running `pnpm install`."

Do NOT block generation on this; the user can install pnpm later.

---

## 2. Interview

[Filled in Task 5]

---

## 3. Generation rules (answer → manifest)

[Filled in Task 6]

---

## 4. File templates

[Filled in Task 7]

---

## 5. Post-generation checklist

[Filled in Task 8]
