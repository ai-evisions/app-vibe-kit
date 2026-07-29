# App Vibe Kit — evisions

A ready-made kit of skills for the **Vibe Coding: from idea to app** workshop.

No code, no app — just skills and rules. You generate everything yourself from
your own brief.

## Quick start

```bash
# 1. Clone the kit under your own name
gh repo clone ai-evisions/app-vibe-kit my-app

# 2. Start Claude Code
cd my-app
claude
# or Claude Desktop → Code tab → Working dir → my-app

# 3. Check it works — type /app and you should get autocomplete
/app-setup
```

## What's in the kit

| File | What it does |
|------|--------------|
| `CLAUDE.md` | Rules for Claude — stack, working with data, git workflow |
| `.claude/skills/` | 10 skills — brief, app, feature, review… |
| `README.md` | You're reading it |
| `.nvmrc` | Node.js version |

## Available skills

Type `/app` in Claude Code and you'll see autocomplete.

| Skill | What it does | Output |
|-------|--------------|--------|
| `/app-setup` | Checks your tools, picks your level, creates your GitHub repo | GitHub repo + `.participant-level` |
| `/app-prd` | The brief step by step — problem, user, scope, data | `PRD.md` + GitHub Issue + backlog |
| `/app-scaffold` | Generates the whole app from your PRD | Next.js app on `localhost:3000` |
| `/app-feature` | Idea → branch → implementation → PR | Feature branch + pull request |
| `/app-review` | A second read on your changes | Report (blockers, warnings, nitpicks) |
| `/app-team` | Three roles: Lead / Builder / Critic, max 2 rounds | A bigger feature with visible collaboration |
| `/app-tests` | Vitest + React Testing Library | 5–8 tests + a green `npm test` |
| `/app-ci` | GitHub Actions pipeline | CI on every push and PR |
| `/app-skill` | Build a portable skill (worked example: `prd-critic`) | A new skill in `.claude/skills/` |
| `/app-deploy` | **Optional** — real database + public URL | A live URL |

## Two modes — same skills, different tone

`/app-setup` asks for your level: **basic** or **advanced**.

- **Basic** — the skill explains, offers examples, asks one question at a time, keeps the scope small
- **Advanced** — the skill moves fast, challenges decisions, skips explanations, leaves you room

All 10 skills are available to everyone. Switch any time in `.participant-level`.

## Prerequisites

- [Node.js 20+](https://nodejs.org)
- [Git](https://git-scm.com) + [GitHub CLI (`gh`)](https://cli.github.com) — then `gh auth login`
- [Claude Code](https://docs.claude.com/en/docs/claude-code) (Claude Pro/Max) or Claude Desktop (Code tab)

That's everything. **No database or hosting accounts** — the app runs on your own
machine.

### Optional (depending on your app)

- [Gemini API](https://aistudio.google.com) — AI feature (free tier)
- [Brevo](https://www.brevo.com) — sending e-mail (free tier, 300/day)

## Stack

- **Next.js 15** — App Router, TypeScript, Tailwind CSS
- **Data in a JSON file** — `data/app.json`, nothing to install, open it in an editor
- **Local** — `npm run dev` → `localhost:3000`
- **Git + GitHub** — issues, branches, pull requests

## Flow

```
/app-setup      →  Tools + GitHub repo
/app-prd        →  Brief → PRD.md + GitHub Issue + backlog
/app-scaffold   →  App from the PRD
                   npm run dev → localhost:3000
```

Then you iterate:

```
Issue/idea → branch → /app-feature → /app-review → PR → test locally → merge → repeat
```

## Habits you take away

- **The brief before the code** — half an hour in the PRD pays for itself several times over
- **One idea = one branch = one PR** — so you know what broke the app
- **Code review** — `/app-review` as a second pair of eyes
- **Data changes in the PR description** — so it's visible what changed in `data/app.json`
- **`.env.example` in git**, secrets in `.env.local`
- **A skill is markdown you write yourself** — see `/app-skill`

## Recipes in /app-feature

The skill has ready-made recipes for:

- **AI feature** — Gemini or Groq, an API route handler, a client call
- **Sending e-mail** — Brevo REST API, server-side route
- **File upload** — server action + `data/uploads/`, route handler to serve them

## When localhost isn't enough any more

The kit also includes `/app-deploy`, which swaps the JSON file for a real Postgres
database (Supabase) and puts the app on a public URL (Vercel) — both free tier.
We don't run it during the workshop, because setting up the accounts eats an hour
of the three. Run it at home.

It matters more than it looks: **a JSON-backed app cannot simply be deployed.**
Vercel's filesystem is ephemeral in production, so writes silently disappear.
`/app-deploy` handles the whole migration — and because all file access lives in
`src/lib/data.ts`, it's a one-file change.
