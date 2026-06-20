# How to do a stage (the ritual)

Every stage follows the same seven steps, in order. The setup takes about 30 seconds
and saves you from flailing.

| # | Step | What you do |
|---|------|-------------|
| 0 | Orient | Learn what the function is supposed to do |
| 1 | Baseline | Confirm the file is clean — this is your reset point |
| 2 | Clear the function | Empty out the one function's body |
| 3 | Confirm the failure | Run its test and watch it fail — that test is your spec |
| 4 | Implement | Fill the function back in, one behavior at a time |
| 5 | Pass the checkpoints | Make each test pass, in order |
| 6 | Wrap up | Reflect, then check the stage off |

---

## 0. Orient — know what "done" means *before* you touch code

- Read the stage's **`## Where you are`** diagram and **`## How it works`** Q&A. Answer
  the *predict-the-behavior* questions out loud. If you can't, read the real code first.
- Read this stage's block in **[`MASTERY.md`](MASTERY.md)** — those are the statements
  you should be able to say by the end. They're your real target; the passing test is
  just the proof.

## 1. Baseline — set your reset point

You're on the `onboarding` branch; `main` holds the pristine original. Before you
change anything, make sure the file you're about to edit is clean so you can always get
back to it:

```bash
git status --short          # should be clean for the file you're touching
```

If you need the original back at any point: `git restore <the-file>` (from `main`).

## 2. Clear the function — empty exactly one thing

Paste the stage's **`## Step 1 — Set up this stage`** prompt to me. It replaces *one
function's body* with `raise NotImplementedError("Stage NN")` and keeps the signature,
docstring, and decorators. Nothing else in the repo changes — that's the whole point.
You then ask me to show you the signature plus the fields and collaborators it uses, so
you know the contract you're implementing.

## 3. Confirm the failure — find your spec

Run the stage's covering test and **confirm it fails**:

```bash
uv run pytest <the path from the stage> -x
```

A failure here is what you want — it proves the test actually exercises the code you
just emptied. Read it closely: that test is the specification: it already encodes what
"correct" means. (See [`TESTING.md`](TESTING.md) → "How to read a failing test as a
spec.")

> If emptying the function does **not** make a test fail, you cleared the wrong thing,
> or the function is only covered end-to-end — check the stage's notes for the
> thin-mock path (stages 01, 02, 03, 18, 19).

## 4. Implement — one behavior at a time

Work the **`## Your tasks`** list top to bottom. Task 1 is always *read and trace the
real implementation*. Then implement the happy path, then each named edge case. Don't
write the whole function and hope — implement the smallest piece that turns the next
checkpoint green.

## 5. Pass the checkpoints — in order

Each task maps to a specific `pytest` invocation. Run it after each sub-behavior:

```bash
uv run pytest <task's test> -q          # one checkpoint
uv run pytest <stage file> --lf         # rerun just what failed last
```

When every checkpoint passes, run the stage's whole file once to be sure you didn't
break a sibling test:

```bash
uv run pytest tests/test_<stage>.py -q
```

## 6. Wrap up — reflect, check off, optionally commit

- Do the **`## What you learned`** reflection and re-answer the MASTERY statements.
  If any still feels shaky, that's the signal to reread — not the test, the *code*.
- Tick the stage in **[`PROGRESS.md`](PROGRESS.md)**.
- Optional: `git commit -am "stage NN done"` and `git push` to back up your fork.
- Move to the **`## Next →`** link.

---

## Hints & "where do I get the other functions?"

A real question, and the answer is reassuring:

- **You only ever empty ONE function.** Everything that function *calls* — helpers,
  earlier-stage methods, frame classes — still exists, untouched. You are filling one
  hole in a working machine, not building from an empty file. So you rarely "need to
  get" another function; you just *call the one that's already there*.

- **What a stage leans on is listed for you.** The `Depends on` column in
  **[`00-START-HERE.md`](00-START-HERE.md)** names the earlier stages whose concepts
  this one builds on. Example: stage 07 (interruption handling) leans on stage 03's
  `FrameQueue.reset` and stage 06's `broadcast_interruption` — both already implemented
  (by the repo, or by you if you did those stages). If a call in your rebuilt function
  looks unfamiliar, it's almost always one of those — go reread that stage's page.

- **The setup prompt hands you the contract.** When you empty a function, ask me to
  show the signature *and* the fields/collaborators it uses (the stage prompts already
  say this). That list IS your hint sheet — those are the exact things you're allowed to
  call.

- **Stuck on the logic itself?** Use the page's `## Stuck?` prompt — it makes me ask
  you a pointed question instead of handing over the answer. And `git restore <file>`
  lets you peek at the real implementation, then clear it again and retry. Reading the
  real code is allowed and encouraged (task 1 of every stage); *copying it blind* is
  what wastes the exercise.

How much to lean on hints is up to you. First try: orient, clear the function, and
rebuild from the test alone. Only open the real code when you're genuinely stuck — that
struggle is
where the learning is.

→ Back to [`00-START-HERE.md`](00-START-HERE.md)
