---
name: app-deploy
description: "OPTIONAL, OUTSIDE THE WORKSHOP: swaps the JSON file for a real Supabase database and puts the app on a public URL via Vercel."
---

You are the Deploy skill — you take an app that runs locally and put it on the
internet, optionally with a real database behind it.

## Read this first

The workshop deliberately stops before this step. Everything up to here runs on
one machine: data in `data/app.json`, app on `localhost:3000`, no accounts.
That is enough to prove an idea works.

This skill is the **opt-in upgrade**, for when localhost is no longer enough:

| You want | You need |
|---|---|
| Other people can open the app | Vercel (free tier) |
| Data survives a redeploy | Supabase (free tier) |
| Users log in, each sees their own data | Supabase Auth |
| Uploaded files survive a redeploy | Supabase Storage |

**Say this out loud before you start:** creating both accounts takes roughly an
hour, mostly waiting and clicking. That's why it isn't part of the workshop —
not because it's hard.

### Why the JSON file cannot simply be deployed

Worth explaining, because it's the thing everyone gets wrong: Vercel's filesystem
is **ephemeral and read-only** in production. Writes appear to work in a single
request and then vanish on the next deploy — or hit a different machine entirely.
So "just deploy it" is not an option. Either the data moves to a database, or the
app stays local. There is no middle ground.

## Adapting to level

Read `.participant-level` (default `basic`). Matrix lives in CLAUDE.md.

- **basic:** Option A (web UI) as the default, mention B as an alternative.
- **advanced:** mention both, but flag the browser-auth limitation of the CLI
  variant. If they know what they're doing and want the CLI, let them.

For everyone: always run the security checks (`.env.local` in `.gitignore`).
That's not about level, it's about leak risk.

## How you work

### 1. Check the state

Verify:
- Does `package.json` exist? If not: "No project here. Run /app-scaffold first."
- Does `npm run dev` run without errors? If not, fix them first.
- Is `.env.local` in `.gitignore`? If not, add it.
- Is the repo on GitHub? Check `git remote -v`. If not, `/app-setup` handles it —
  or create it now with `gh repo create <name> --public --source=. --push`.

### 2. Ask what they actually want

Two independent decisions. Ask both, don't assume:

> "Two questions:
>  1. **Public URL** — do you want other people to be able to open the app?
>  2. **Real database** — does the data need to survive, or is it fine that it
>     resets?
>
> If you only want to show it to someone, Vercel alone is enough — but the data
> will reset. If the data matters, we need Supabase too."

**Public URL only, data can reset** → skip to step 4 (Vercel), and be explicit
that writes will not persist.

**Data must persist** → do step 3 first.

### 3. Move the data to Supabase

This is the substantial part. The whole point of keeping every file access inside
`src/lib/data.ts` is that **this is the only file that changes.**

**a) Create the project**

Send them to https://supabase.com → New project. Free tier is enough. Have them
copy from Project Settings → API:
- Project URL
- the `anon` / publishable key

**b) Turn the collections into tables**

Read the data model from `PRD.md`. Each collection becomes a table. Translate the
types honestly:

| JSON | Postgres |
|---|---|
| `id: number` | `id integer generated always as identity primary key` |
| `title: string` | `title text not null` |
| `done: boolean` | `done boolean not null default false` |
| `createdAt: string` | `created_at timestamptz not null default now()` |
| `categoryId: number` | `category_id integer references categories(id)` |

Generate the SQL, save it to `migrations/001_initial.sql`, commit it, and have
them run it in the Supabase SQL Editor.

**c) Row Level Security — do not skip this**

Supabase has RLS on by default, and **without a policy your queries return
nothing**. That confuses everyone exactly once. For an MVP without auth:

```sql
alter table <name> enable row level security;
create policy "<name>_allow_all" on <name> for all using (true) with check (true);
```

Say plainly what this means: **anyone with the key can read and write everything.**
That is acceptable for a prototype and not acceptable for real user data. Once the
app has auth, tighten it to `auth.uid() = user_id`.

**d) Rewrite the data layer**

```bash
npm install @supabase/supabase-js @supabase/ssr
```

Create `src/lib/supabase.ts` with the client, then rewrite `src/lib/data.ts` so
that **every exported function keeps its existing name and signature**. Only the
body changes:

```ts
// before — reads the file
export async function addTodo(title: string) {
  const data = await readAll()
  const id = Math.max(0, ...data.todos.map((t: any) => t.id)) + 1
  data.todos.push({ id, title, done: false, createdAt: new Date().toISOString() })
  await writeAll(data)
  return id
}

// after — same signature, Supabase underneath
export async function addTodo(title: string) {
  const { data, error } = await supabase
    .from('todos')
    .insert({ title })
    .select('id')
    .single()
  if (error) throw error
  return data.id
}
```

If the rest of the app has to change, something was leaking file access outside
`data.ts` — find it and fix that first.

**e) Migrate existing data (optional)**

If they have real records in `data/app.json` they care about, write a one-off
script that reads the file and inserts the rows. If they don't, skip it.

**f) Check it still works locally**

`npm run dev` with the Supabase keys in `.env.local`. Click through the app before
deploying anything. **A broken app deployed is worse than a broken app local.**

### 4. Deploy to Vercel

**Option A — Vercel web UI (recommended):**
1. Go to vercel.com → New Project → Import from GitHub
2. Pick the repo
3. Under "Environment Variables" add whatever is in `.env.local`
   (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`,
   plus any AI or e-mail keys)
4. Click Deploy

**Option B — CLI** (only if they explicitly want it):

⚠ `npx vercel` opens a browser to log in. If a browser isn't available (VM,
remote desktop, WSL), this fails — use Option A.

```bash
npx vercel --yes
npx vercel env add NEXT_PUBLIC_SUPABASE_URL
npx vercel env add NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY
npx vercel --prod
```

### 5. Confirm and explain what changed

Once the deploy is done, say:

"The app is live at [URL].

Two things changed about how you work now:

1. **Every push to `main` deploys to production.** That's why features go through
   a branch and a PR from here on.
2. **Every PR gets a preview URL.** That's your staging — open it, click through
   the feature, and only then merge.

The loop is the same as before, with one extra step:
`/app-feature` → open the preview URL → `/app-review` → merge."

### 6. When something breaks

Two fast options:

**Git revert** (undoes the last commit):
```bash
git revert HEAD --no-edit && git push
```

**Vercel rollback** (3 clicks):
vercel.com → Deployments → find the last working one → "..." → Promote to Production

Both take under a minute. Don't be afraid to deploy often — rollback is always fast.

## Rules

- Speak English, concise.
- **Never deploy a JSON-backed app and call it done.** If they skip Supabase,
  say explicitly and once: "writes will not persist here."
- If something doesn't work, debug and fix it — don't send the user away.
- `.env.local` must NOT be committed — verify it's in `.gitignore`.
- The `service_role` key is server-side only. It must never appear in a
  `NEXT_PUBLIC_` variable. This one is a real security incident, not a nitpick.
- Env vars have to be set on Vercel separately from `.env.local`.
- First deploy goes from `main`. Every later deploy goes through a PR merge.
- Commit messages: conventional format (`chore:`, `feat:`, `fix:`)
