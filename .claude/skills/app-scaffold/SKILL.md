---
name: app-scaffold
description: "3. Turns your PRD into a complete Next.js app. Data lives in a JSON file on disk. Run after /app-prd."
---

You are the Scaffold skill — your job is to take an existing PRD and turn it into
a working web application.

## Local by default · Supabase optional

Data live in `data/app.json` on disk and the app runs on `localhost:3000` — no
accounts, no keys. Supabase is the supported option if someone wants a real
database; `/app-deploy` sets it up.

**What this changes for you:** this is where it matters most. You put **all** file
access into a single file, `src/lib/data.ts`. That is not tidiness for its own
sake — it is what makes the Supabase switch later a one-file change instead of a
rewrite. Keep every exported function's name and signature independent of where
the data actually lives.

## Adapting to level

Read `.participant-level` (default `basic`). Behaviour matrix lives in CLAUDE.md.

**Skill-specific effects:**

- **basic:** install, generate, and finish with 2–3 bullets on what they now have.
- **advanced:** move fast, but offer choices: "App Router (default) or Pages
  Router?", "server components or client-side?". Respect their answers. If a
  choice is non-standard for this stack, name the trade-off and accept it anyway.

If an advanced participant rejects the default data layer (wants SQLite or Drizzle
instead of a JSON file), let them — but warn: "The other skills (feature, review,
tests) assume the JSON layer. Some of their advice won't fit." And mention the one
practical trap: native modules like `better-sqlite3` can fail at compile time, so
in a 3-hour workshop that's a risk, not a saving.

## How you work

### 1. Read the PRD

Read `PRD.md` in the project root. If it doesn't exist, tell the user:
"No PRD here. Run /app-prd first to create the brief."

### 2. Check prerequisites

No accounts or keys needed — the data goes into a JSON file on disk.
Just confirm you're in a git repo and that `npm -v` works.

Tell the user in one sentence what's about to happen: "I'm going to generate the
whole app from your PRD now. It'll run for a while and produce a lot of text —
don't read it. When it's done we'll open `localhost:3000` and you'll click around."

### 3. Generate the application

Based on the PRD:

1. **Create the Next.js project in a subdirectory** — `create-next-app` needs an
   empty directory, but the repo already has CLAUDE.md, PRD.md, .claude/ and so on.
   So initialise into a temporary subdirectory and then move it:

   ```bash
   npx create-next-app@15.5.15 nextapp --yes --typescript --tailwind --eslint --app --src-dir \
     --import-alias "@/*" --use-npm --turbopack --disable-git
   ```

2. **Move the app contents into the project root:**

   ```bash
   # Move everything, including dotfiles, then drop the empty folder
   find nextapp -mindepth 1 -maxdepth 1 -exec mv -f {} . \;
   rmdir nextapp
   ```

   **Two traps worth knowing about, both already handled above:**
   `create-next-app` rejects a folder name starting with an underscore (npm naming
   rules), which is why it is `nextapp`. And do **not** use `shopt -s dotglob` for
   the move — `shopt` is a bash builtin and macOS defaults to zsh, so it fails with
   "command not found". The `find` version above works in both shells.

   Without `--yes` the installer stops on an interactive Turbopack prompt and
   appears to hang, so keep that flag.

   Our workshop README.md survives the move (create-next-app wrote its own into
   the subdirectory, not the root).

   **The `.gitignore` does NOT survive** — create-next-app ships its own and the
   move overwrites ours, silently dropping the workshop entries. Put them back:

   ```bash
   cat >> .gitignore <<'EOF'

   # Workshop
   data/uploads/
   .participant-level
   EOF
   ```

   Then check `.env.local` is listed too (create-next-app usually adds it).

   Finally set the project name in `package.json` — create-next-app names it after
   the temporary folder (`nextapp`), not what the participant called their app.

3. **Install nothing else.** The data layer is `node:fs/promises`, which ships
   with Node.js. Fewer packages means fewer things that can break.

   **Relax one ESLint rule before you generate any code.** The default config
   treats `any` as an error, so a single `any` anywhere fails `npm run build` —
   and AI-generated feature code produces them regularly. Downgrade it to a
   warning in `eslint.config.mjs`:

   ```js
   // inside the exported config array, after the Next.js presets
   {
     rules: {
       "@typescript-eslint/no-explicit-any": "warn",
     },
   },
   ```

   Keep it a warning, not "off" — participants should still see them.

4. **Create the data layer** (`src/lib/data.ts`) — the only place in the entire
   app that touches the file. Follow this pattern exactly:

   ```ts
   import { readFile, writeFile, mkdir } from 'node:fs/promises'
   import path from 'node:path'

   // One type per collection from the PRD. Keep them typed — the default ESLint
   // config treats `any` as an ERROR and it will fail `npm run build`.
   export type Todo = {
     id: number
     title: string
     done: boolean
     createdAt: string
   }

   type Data = { todos: Todo[] }

   const FILE = path.join(process.cwd(), 'data', 'app.json')
   const EMPTY: Data = { todos: [] }

   async function readAll(): Promise<Data> {
     try {
       return JSON.parse(await readFile(FILE, 'utf8')) as Data
     } catch {
       return structuredClone(EMPTY)   // file doesn't exist yet → empty data
     }
   }

   async function writeAll(data: Data) {
     await mkdir(path.dirname(FILE), { recursive: true })
     await writeFile(FILE, JSON.stringify(data, null, 2), 'utf8')
   }

   // Every mutation goes through here, and they run strictly one after another.
   // Without this queue, two requests arriving together BOTH read the old file and
   // the second one overwrites the first. Tested: 3 parallel writes without it
   // produced 1 record. Do not remove it.
   let queue: Promise<unknown> = Promise.resolve()
   function update<T>(fn: (data: Data) => T): Promise<T> {
     const run = queue.then(async () => {
       const data = await readAll()
       const result = fn(data)      // mutate the object in place
       await writeAll(data)
       return result
     })
     queue = run.catch(() => {})    // a failed write must not block the queue
     return run
   }

   export async function listTodos(): Promise<Todo[]> {
     return (await readAll()).todos
   }

   export function addTodo(title: string): Promise<number> {
     return update((data) => {
       const id = Math.max(0, ...data.todos.map((t) => t.id)) + 1
       data.todos.push({ id, title, done: false, createdAt: new Date().toISOString() })
       return id
     })
   }
   ```

   **Every function that changes data must go through `update()`.** Reads may call
   `readAll()` directly. This queue only protects a single process — which is all
   we need locally, and one more reason a deployed app needs a real database.

   Derive function names from the user stories in the PRD (`addTodo`,
   `toggleTodo`, `deleteTodo`, `listTodos`…).

5. **Create `data/app.json`** with the empty collections from the PRD and commit
   it — so the app works immediately after a fresh clone.

6. **If the PRD has an "External services" section**, ask for the keys and create
   `.env.local` + `.env.example`:
   - AI feature → `GEMINI_API_KEY` or `GROQ_API_KEY`
   - E-mail → `BREVO_API_KEY`

   Say: "Your PRD has an [AI/e-mail] feature. I'll need an API key. Do you have
   one? If not: [link from the PRD]." If the PRD has no external services, don't
   create `.env.local` at all — there's nothing to put in it.

   `.env.example` is a template with no real values and **belongs in git**;
   `.env.local` with actual keys must never be committed.

7. Implement the CRUD UI from the user stories in the PRD:
   - A list of items with the ability to add
   - A form for creating a new item
   - Edit and delete
   - A basic layout with navigation

   Read data from **server components**, write through **server actions**
   (`'use server'`) — the data layer runs on the server and has no business in
   the browser.

### 4. Commit and push

Once the app generates successfully, commit and push to GitHub (if a remote exists):

```bash
git add -A
git commit -m "feat: scaffold from PRD"
git push
```

If the push fails (no remote), that's fine — `/app-setup` can sort it out.

## Rules

- TypeScript. `any` is tolerated (the rule is a warning), but type the data layer
  properly — that is the file everything else depends on
- Tailwind CSS for all styling
- **All data access goes through `src/lib/data.ts`.** `readFile` and `writeFile`
  must not appear anywhere else in the app
- **Pages that read data must have `export const dynamic = 'force-dynamic'`.**
  Without it Next.js caches the response, the user adds a record and nothing
  changes on screen — **this is the single most common source of confusion in the
  whole workshop**, and it produces no error message at all
- After every data change call `revalidatePath('/')` (or the affected path)
- Use the App Router (server and client components)
- Keep the code simple and readable — no extra abstractions
- English UI text
- `.env.local` must not be in git — verify it's in `.gitignore`
- Commit messages: conventional format
- When you're done say: "Your app is ready! Run `npm run dev` and open
  http://localhost:3000. Your data lives in `data/app.json` — open it in an editor
  and you'll see your own records. The code is pushed to GitHub.
  Don't read the generated code line by line — **click around in the app**.
  Next: `/app-review` reads the code and finds weak spots, or go straight to
  `/app-feature` if you want to add something."
