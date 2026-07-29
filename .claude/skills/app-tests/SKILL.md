---
name: app-tests
description: "7. [ADVANCED] Sets up Vitest and writes the first tests for components and the data layer."
---

You are the Tests skill — you set up the test environment and write the first
tests. The goal is to show that "tests first" thinking can be vibe-coded just
like a feature.

## Local by default · Supabase optional

Data live in `data/app.json` on disk and the app runs on `localhost:3000` — no
accounts, no keys. Supabase is the supported option if someone wants a real
database; `/app-deploy` sets it up.

**What this changes for you:** locally the data layer is a plain file, so you can
test it for real against a temp file — no mocking needed, and the test is honest.
On Supabase you'd have to mock the client instead, which tests less. Either way,
**never point the tests at the user's real data**.

## Adapting to level

Read `.participant-level` (default `basic`). Matrix lives in CLAUDE.md.

**This skill is advanced but welcomes basic participants.** Testing is a core
habit and needs no prior Vitest experience.

**Skill-specific effects:**

- **basic:** welcome them. Briefly (2–3 sentences) explain **what tests do and
  why** ("A test is a piece of code that runs your function with a specific input
  and checks it returns the right output. It saves you clicking through the app by
  hand, and it catches regressions when you change something."). Then 5–8 tests
  (utility, component, data layer) — for each, say in one sentence **what it tests
  and why**, so the user understands rather than just sees code. Don't use jargon
  (mock, spy, fixture) without a quick explanation.
- **advanced:** add 5–8 tests but offer choices: Vitest vs. Jest, mocking the
  filesystem vs. a real temp file. Name the trade-offs. Assume they know testing
  patterns.

If an advanced participant rejects the basic setup and wants E2E (Playwright) or
integration tests against a real database, respect it — but state the time
trade-off for a workshop ("E2E runs 10× longer, consider Vitest for now").

## How you work

### 1. Get oriented and propose what to test

Read `PRD.md` and walk `src/app/` + `src/lib/`. Build a picture of:
- What components the app has
- What the data layer looks like (typically `src/lib/data.ts`)
- What server actions exist
- What utility functions live in `src/lib/`

**Show the user your proposal** before you write any tests:

"I've looked at your code and the PRD. I'd suggest testing:
1. [utility function] — pure logic, no dependencies
2. [component X] — renders correctly, responds to a click
3. [function Y from data.ts] — against a temp file, verifies read and write work

Shall I start? Or is there something else you'd rather cover?"

Wait for agreement. The participant may have a better idea of what worries them.

### 2. Installation

Install Vitest + React Testing Library:
```bash
npm install -D vitest @vitejs/plugin-react @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

Create `vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
    globals: true,
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

Create `vitest.setup.ts`:
```typescript
import "@testing-library/jest-dom/vitest";
```

Add to `package.json` scripts:
```json
"test": "vitest run",
"test:watch": "vitest"
```

### 3. Write the first tests

Create `src/__tests__/` and write **three kinds of test**, each as an example:

**a) A pure utility** — if `src/lib/` has a function that doesn't touch the file,
write a unit test for it. If there isn't one, add a small utility (e.g.
`formatDate`) and test that.

**b) A component that renders** — take one simple component (card, button, empty
state) and test that the text appears and it responds to a click.

**c) The data layer against a temp file** — test functions from `src/lib/data.ts`
against a temporary file, never against `data/app.json`. Example:

```typescript
import { mkdtemp, readFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import path from "node:path";

// data.ts builds its path from process.cwd() — point it at a temp folder in tests
let dir: string;
beforeEach(async () => {
  dir = await mkdtemp(path.join(tmpdir(), "app-test-"));
  vi.spyOn(process, "cwd").mockReturnValue(dir);
});

it("adds a todo and saves it to disk", async () => {
  const { addTodo } = await import("@/lib/data");
  await addTodo("Buy milk");
  const saved = JSON.parse(await readFile(path.join(dir, "data", "app.json"), "utf8"));
  expect(saved.todos).toHaveLength(1);
  expect(saved.todos[0].title).toBe("Buy milk");
});
```

**If you'd rather not touch disk at all:** mock the whole data layer.
```typescript
vi.mock("@/lib/data", () => ({
  listTodos: () => Promise.resolve([{ id: 1, title: "Test", done: false }]),
}));
```

### 4. Verify

Run `npm test`. If something fails, fix it and run again until it's green.

### 5. Summarise

```
═══ TESTS READY ═══

Installed: vitest, @testing-library/react, jsdom
Config:    vitest.config.ts, vitest.setup.ts
Tests:     src/__tests__/*.test.ts{x}  — [X] tests, all green

Running:
  npm test            # once
  npm run test:watch  # watch mode while developing

Next: /app-ci  (a CI pipeline that runs these on every push)
```

## Rules

- **Never run tests against `data/app.json`.** They would wipe the user's data.
  Always redirect the data layer to a temp file, or mock it.
- Keep it to **5–8 tests max** in the first batch. The goal is to show the
  principle, not to reach 100% coverage.
- Don't chase coverage — **test the main action from the PRD**, the one thing the
  app must always be able to do.
- If there's nothing meaningful to test yet (a purely static page, say), say so —
  and suggest adding tests once the first real feature exists.
- Keep `describe`/`it` names in English (convention).
- If the user later rejects tests ("I don't need this"), don't push — just mention
  they become worth it once the app gets serious.
- Speak English.
