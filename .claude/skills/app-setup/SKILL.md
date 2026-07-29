---
name: app-setup
description: "1. Checks your tools, creates your own GitHub repo and sets your level. Run this first."
---

You are the Setup skill — your job is to verify the user has everything they need,
create their own GitHub repo, and set their level.

## Local by default · Supabase optional

Data live in `data/app.json` on disk and the app runs on `localhost:3000` — no
accounts, no keys. Supabase is the supported option if someone wants a real
database; `/app-deploy` sets it up.

**What this changes for you:** the checklist is short. Five things — Node, npm,
Git, GitHub CLI, Claude Code. **No database or hosting accounts.** If someone asks
"don't I need to sign up for something?", the answer is no, and the option is
there later via `/app-deploy` if they want it.

## Adapting to level

This skill **sets** the level (step 7), so `.participant-level` doesn't exist yet.
Behave as basic — neutral tone, concise.

## Process

Run these checks one at a time and show the result after each:

### 1. Node.js
Run `node -v`.
- ✓ if version 20+
- ✗ if missing or older. Say: "Install Node.js 20+ from https://nodejs.org"

### 2. npm
Run `npm -v`.
- ✓ if it works
- ✗ if missing. Say: "npm ships with Node.js — try reinstalling Node"

### 3. Git
Run `git --version`.
- ✓ if it works
- ✗ if missing. Say: "Install Git from https://git-scm.com"

### 4. GitHub CLI
Run `gh --version`.
- ✓ if it works
- ✗ if missing. Say: "Install the gh CLI from https://cli.github.com — we need it
  to create the repo and work with pull requests."

**Note:** the GitHub CLI matters here (repo, issues, PRs). If the participant
can't or won't install it, the workshop still works, just with manual steps on
the web.

### 5. GitHub auth (BLOCKER for step 8)

If `gh` exists, verify it **functionally** — do not try to read the scope list.
Scope strings are unreliable: `read:org` already covers the identity lookup, and a
perfectly working login often shows no `read:user` at all. What matters is whether
these two actually succeed:

```bash
gh auth status          # is there a login at all?
gh api user -q .login   # can it resolve the identity?
```

**Evaluation:**

- ✓ `gh api user` returns a username → **you are done, move on.** Do not comment on
  scopes, do not suggest refreshing anything.
- ✗ Not logged in → **BLOCKER**. Say: "Log in with `gh auth login`. Choose
  GitHub.com → HTTPS → Authenticate Git with credentials → Login with a web
  browser. When you're done, run `/app-setup` again."
- ✗ Logged in but `gh api user` fails → the token is genuinely missing permissions.
  Which fix depends on how they logged in:

  **a) Browser auth (`gh auth login` via browser):**
  ```bash
  gh auth refresh -s repo,read:user
  ```
  Opens a browser, approve, done.

  **b) Token auth (`gh auth login --with-token`, typically corporate):**
  `gh auth refresh` **does not work** with token auth — a new PAT is needed. Say:
  > "Your token can't read your account. Go to
  > https://github.com/settings/tokens → Generate new token (classic) → tick
  > `repo` (the whole block) and `read:user`. Copy the new token and run
  > `gh auth login --with-token < token.txt`. Then run `/app-setup` again."

  **If they fix it** → run `/app-setup` again.
  **If they can't** (corporate restrictions, no permission to generate a PAT with
  `repo` scope) → skip step 8, create `.github-pending`, and continue. It blocks
  `gh repo create --push`, not the workshop — everything else works locally and the
  repo can be set up by hand later.

**Detecting the auth type** (helps you pick a or b): `gh auth status` shows
`Token: gho_*` (browser/oauth) vs `ghp_*` (personal access token, usually
`--with-token`). Not 100% reliable, but a good hint.

### 6. Claude Code
Nothing to test — if the user is running this skill, Claude Code works.
Mark it ✓ automatically.

### 7. Calibrate the level

This step matters — it sets how every other skill will treat them.

Say this (friendly, not like a form):

> "One last thing — I want to match your pace. Which mode do you want?
>
> **A) Basic** (default) — I walk you through step by step, explain what and why,
> and offer options. Good if you want to see the whole flow and understand it.
>
> **B) Advanced** — I move fast, challenge your decisions, skip explanations.
> More freedom, less hand-holding. It also unlocks the heavier stuff (a team of
> roles for bigger features, a more complex data model).
>
> If you're not sure, pick A. You can change it any time in `.participant-level`."

Wait for the answer. Map it:
- A or an unclear answer → `basic`
- B → `advanced`

Then create `.participant-level` in the project root:

```bash
echo -n "basic" > .participant-level   # or advanced
```

### 8. Create their own GitHub repo

This is the pivotal step — the participant stops working on the workshop kit and
starts working on their own repo.

**Explain what's about to happen first** (especially important for basic users):
"This kit was prepared for you — it's the skills that will guide you. I'm now
going to disconnect you from that copy and create your own repo on GitHub. From
here it's your code and you can do whatever you like with it."

**Ask for a project name:**
"What do you want to call your project? One word, lowercase, no spaces.
Examples: my-todos, habit-tracker, bookings."

Then:

**Pre-flight (before removing the origin!):**

Rule: **if anything here fails, do NOT remove the kit origin** — send the user to
the fallback instead. Removing the origin and then discovering the push doesn't
work is the worst case: the repo ends up in a broken state.

```bash
# 1. Identity — gh must be able to call /user
gh api user -q .login
```
Fails → fallback (gh isn't working or isn't logged in).

```bash
# 2. Can it actually create a repo? Dry-run the permission, don't read scopes.
gh api -X GET user/repos -f per_page=1 >/dev/null
```
If this fails, the token cannot see repos and `gh repo create` will fail too →
fallback. **Never gate on the scope string** — a working login frequently shows no
`read:user`, and blocking on that stops people who are completely fine.

```bash
# 3. Set up the HTTPS credential helper via gh — avoids SSH keys entirely
gh auth setup-git
```
This configures git to use `gh` as the credential helper for GitHub HTTPS pushes.
For **token-auth users** (corporate) this is the critical step — without it
`git push` fails with "could not read Username".

**Only after a clean pre-flight, continue:**

```bash
# Disconnect the kit remote
git remote remove origin 2>/dev/null

# Commit the starting state (CLAUDE.md, skills, .gitignore, etc.)
git add -A
git commit -m "chore: workshop kit setup"

# Create the participant's own repo (HTTPS push via the gh credential helper)
gh repo create <name> --public --source=. --push
```

**Verify the push landed** — after `gh repo create`, run
`git log --oneline -1 origin/main` and check the remote points at the same commit.
If it does:
- ✓ Say: "Your repo is live: https://github.com/<user>/<name>"

If the pre-flight or the push fails:
- ✗ **Don't remove the origin if it hasn't been removed yet.** Say: "Couldn't
  create the GitHub repo. No problem — carry on with /app-prd and we'll sort the
  repo out later." Create a `.github-pending` file so later skills know.

**If `gh` CLI doesn't exist at all:**
Skip this step. Say: "You don't have the gh CLI, so we'll create the repo manually
later. Carry on with /app-prd." Create `.github-pending`.

## Output

Finish with a summary:

```
═══ WORKSHOP SETUP CHECK ═══

 1. Node.js      ✓ v22.1.0
 2. npm          ✓ v10.2.0
 3. Git          ✓ v2.43.0
 4. GitHub CLI   ✓ v2.40.0
 5. GitHub auth  ✓ logged in as <user>
 6. Claude Code  ✓
 7. Level        ✓ basic
 8. GitHub repo  ✓ github.com/<user>/<name>  (or ⚠ deferred)

Ready: X/8 ✓

Nothing else to install — no database, no hosting account.
Your app will run on your own machine.
```

If everything is fine, say: "You're all set! Your project lives on GitHub. Start
with /app-prd — it helps you write the brief and saves it as an issue in your repo."

If something is missing, say exactly what to fix and offer: "Once that's sorted,
run /app-setup again to re-check."

## Rules

- Speak English, concise
- Run the checks one at a time, don't batch them
- Show the result of each check immediately so the user sees progress
- Install nothing automatically — just say what's missing and how to install it
- The GitHub repo is strongly recommended but not a blocker — the workshop works
  without it
