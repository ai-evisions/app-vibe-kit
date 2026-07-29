# Vibe Coding Workshop — evisions

This is the project directory for the workshop "Vibe Coding: from idea to app".

## Stack

- Next.js 15 (App Router) + TypeScript + Tailwind CSS
- **Data in a JSON file on disk** (`data/app.json`) — no database
- **The app runs locally** (`npm run dev` → localhost:3000) — nothing is deployed
- Git + GitHub for issues, branches and pull requests

## Local by default, Supabase optional

Everything in this kit runs on one machine. No accounts, no keys, no cloud.
That is a deliberate choice: it takes about an hour to set up a database and a
hosting account, and that hour is better spent on the product.

**Supabase and Vercel remain a supported option.** `/app-deploy` swaps the JSON
file for a real Postgres database and puts the app on a public URL. Reach for it
when someone wants:

| They want | They need |
|---|---|
| Other people can open the app | Vercel |
| Data survives a redeploy | Supabase |
| Users log in, each sees their own data | Supabase Auth |
| Uploaded files survive a redeploy | Supabase Storage |

Until then, keep the local path. And whenever someone asks "can't we just deploy
the JSON version?" — no. Vercel's filesystem is ephemeral in production, so writes
silently disappear. Either the data moves to a database, or the app stays local.

## Rules for this repo

- We build a WEB application (Next.js) — no native/mobile apps
- The UI must be responsive (mobile-first) — Tailwind breakpoints
- English comments and UI text
- Keep the code simple — no over-engineering, this is an MVP/prototype
- When you don't know, ask the user instead of guessing

## Working with data — read this before you write a line

This is the only non-trivial part of the stack. Get it wrong and the participant
won't see time wasted — they'll just see "the app doesn't work".

1. **The only place that touches the file is `src/lib/data.ts`.**
   `readFile` and `writeFile` must not appear anywhere else in the app.
   This is also what makes a later switch to Supabase a one-file change.

2. **Every write goes through the serialized `update()` helper** in `data.ts`.
   Read → modify → write inside one function is not enough on its own: with async
   I/O two requests that arrive together both read the old file, and the second
   overwrites the first. Measured, not theoretical — three parallel writes without
   the queue produced one record. Reads may call `readAll()` directly.

3. **Pages that read data must have `export const dynamic = 'force-dynamic'`.**
   Without it Next.js caches the response, the user adds a record and nothing
   changes on screen. **This is the single most common source of confusion in the
   whole workshop** — head it off, don't wait for someone to hit it.

4. **After every data change call `revalidatePath()`** on the affected route.

5. **Read from server components, write through server actions** (`'use server'`).
   The data layer runs on the server and has no business in the browser.

6. **A missing file is not an error.** When `data/app.json` doesn't exist, return
   empty collections — the app must work immediately after a fresh clone.

7. **`id`s are numbers** (1, 2, 3…), new = highest existing + 1. No uuids.
   Keep relations as `categoryId: number`.

8. **Uploaded files** go to `data/uploads/` (in `.gitignore`). Store only the
   filename in the JSON. Always pass the filename through `path.basename()`.

If a participant explicitly wants a real database, let them — but say out loud
that the other skills assume the JSON layer, and that native modules
(`better-sqlite3`) can fail at compile time. In a three-hour workshop that's a
risk, not a saving. The supported upgrade path is `/app-deploy`.

## Git workflow — one idea, one branch, one PR

- **One idea = one branch = one PR.** Not for tidiness — so that you can tell
  which change broke the app, and roll it back without panic.
- **Flow:** pick an issue → `feat/<number>-<name>` → implement → PR with `fixes #N`
  → test locally → merge
- **Testing before a merge:** the app runs locally, so it gets tested locally —
  `git checkout <branch> && npm run dev`. Never merge something you haven't seen
  working. (With `/app-deploy` set up you'd get preview URLs on every PR instead.)
- **Commit messages** use the conventional format:
  - `feat:` — a new feature (`feat: filter by category`)
  - `fix:` — a bug fix (`fix: empty state on first load`)
  - `refactor:` — refactoring with no behaviour change
  - `style:` — purely visual changes (CSS, spacing)
  - `ci:` — CI/CD pipeline
  - `chore:` — everything else (scaffold, deps, config)
  - `docs:` — documentation, PRD
  - If a commit resolves a GitHub Issue, reference it: `feat: X (fixes #3)`
- **Never commit** `.env.local`, `data/uploads/` or anything with credentials

## Available skills

This project ships with skills in `.claude/skills/`. Type `/app` in Claude Code
and you'll get autocomplete for all of them.

### Core track (follow in this order)
- `1.` `/app-setup` — Checks your tools and creates your own GitHub repo
- `2.` `/app-prd` — Walks you through the brief → PRD.md + GitHub Issue + backlog
- `3.` `/app-scaffold` — Generates the whole app from your PRD
- `4.` `/app-feature` — Branch + implementation + PR
- `5.` `/app-review` — A second read on your changes (security, UX, PRD alignment)

### Advanced track (for participants who are ahead)
- `6.` `/app-team` — Three roles in one run: Lead directs, Builder codes, Critic reviews
- `7.` `/app-tests` — Vitest + React Testing Library, the first tests
- `8.` `/app-ci` — GitHub Actions pipeline (lint, typecheck, test, build)
- `9.` `/app-skill` — Build your own portable skill

### Outside the workshop
- `/app-deploy` — Swaps the JSON file for Supabase and puts the app on a public
  URL via Vercel. **We don't run it during the workshop** — the app stays local.
  It's here for anyone who wants to take it further at home.

### Typical flow
1. `/app-setup` → `/app-prd` → `/app-scaffold` → `npm run dev`
2. Loop: `/app-feature` → `/app-review` → merge
3. If you're ahead: `/app-tests` → `/app-ci`, `/app-team` for a bigger feature,
   or `/app-skill` for your own portable skill

## Guided mode (when the user doesn't invoke skills explicitly)

Skills are **shortcuts, not prerequisites**. If a participant starts a session
without `/app-*` and describes an idea, asks "what now?", or just writes "I want
to build an app for X" — **walk them through the flow yourself**, don't wait for
them to remember the right `/`.

### How it works

At the start of a session (or whenever the user signals a return to the workshop
flow):

1. **Detect the current phase** from the state of the repo (table below).
2. **Read the matching SKILL.md** with the Read tool — that is the source of truth
   for that step. Follow it as if the user had invoked it themselves.
3. **Don't copy the instructions in here.** Always load them from
   `.claude/skills/<name>/SKILL.md` so nothing drifts.

### Phase detection (first match wins)

| Repo state | Phase | What to load |
|---|---|---|
| No `.participant-level` | Setup | `.claude/skills/app-setup/SKILL.md` |
| No `PRD.md` | PRD | `.claude/skills/app-prd/SKILL.md` |
| No `package.json` (no scaffold yet) | Scaffold | `.claude/skills/app-scaffold/SKILL.md` |
| All of the above exist | Feature loop | `.claude/skills/app-feature/SKILL.md` (or `app-review` / `app-team` as appropriate) |

### Rules for guided mode

- **Start by saying where we are and what you suggest.** One sentence, not a
  paragraph. ("You have a PRD and a working app. I'd add the first feature. I can
  walk you through it, or run `/app-feature` yourself.")
- **Always mention the equivalent skill** as an alternative. The goal is that the
  participant gradually discovers the skills exist, without being pushed at them.
- **Respect `.participant-level`** exactly as you would on an explicit invocation
  (pace, explanation, default choices).
- **Don't trigger guided mode on every message.** If the user just wants to chat,
  debug one specific thing, or asks about something outside the flow, answer
  normally. Only switch it on when it's clear they want to progress.

### Examples

**A — first message from a new participant:**
> user: "Hi, I'd like to try building an app for planning holidays."
>
> claude: no `.participant-level` → reads `app-setup/SKILL.md`, runs the check.
> Then flows straight into the PRD phase (`app-prd/SKILL.md`).

**B — participant returning mid-session:**
> user: "What now?"
>
> claude: checks state. `PRD.md` exists, `package.json` too → "You have a PRD and
> the app. I'd add the first feature. I can walk you through it, or run
> `/app-feature` yourself."

**C — feature loop:**
> user: "I'd like to filter trips by price."
>
> claude: everything is in place → feature loop. Reads `app-feature/SKILL.md` and
> proceeds. When done, suggests `/app-review`.

## Participant level (shared by all skills)

Every skill in this repo adapts to the participant's level. The current level
lives in `.participant-level` in the repo root. Values: `basic` (default),
`advanced`. The file is created by `/app-setup`; every other skill reads it at the
start of its session.

If the file doesn't exist or is empty → behave as **basic**.

### Behaviour matrix

| Dimension | basic (default) | advanced |
|-----------|-----------------|----------|
| Question pace | 1 question with 2–3 examples | 2–3 questions at once, no suggestions |
| Explanation | Briefly what and why | Only what you're doing, no rationale |
| Default choices | Offer 2–3 options | Ask openly ("what do you want?") |
| Scope | Propose an MVP, let them decide | Respect their proposal, challenge edge cases and trade-offs |
| Reaction to a mistake | "The problem is X, try Y" — matter-of-fact | "Why do you think…? What happens when…?" — Socratic |
| Encouragement | Neutral feedback | Direct, no praise for obvious things |
| Tone | Friendly and concise | Matter-of-fact, efficient |

### Dynamic adaptation

Even with a stored level, watch the signals and adapt within a session.
**Don't change the level in the file** — just adjust behaviour temporarily.

**Signals for more hand-holding:**
- Asks "what should I type?" instead of describing intent
- Fear of breaking things ("I don't want to break it")
- "I don't understand" / "What does that mean?"

**Signals for less hand-holding, more challenge:**
- Asks "why not X" / "couldn't we do it via Y?"
- Uses technical terminology (SSR, server action, hydration, ADR)
- Mentions previous projects or production experience
- Proposes their own architectural decisions

**Explicit override (apply immediately):**
- "Explain in more detail" → adjust tone
- "You don't need to explain" / "Just do it" → skip rationale, go to action
- "Can you skip that?" → respect it

### Rules

- **Never write about the level explicitly.** Don't say "I see you're advanced" —
  just behave differently.
- **Read** the level from the file. Don't change it at runtime (only `/app-setup`
  may rewrite it, and only if the participant asks).
- When the signals contradict the stored level, adapt gradually — someone can be
  a senior in Java and new to Next.js.
