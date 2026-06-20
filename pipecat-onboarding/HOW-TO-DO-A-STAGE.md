# How to do a stage (the ritual)

Every stage follows the same loop. Do these in order. It takes ~30 seconds of setup
and saves you from flailing.

```
0. ORIENT   → 1. SNAPSHOT → 2. GUT → 3. GO RED → 4. REBUILD → 5. GO GREEN → 6. CLOSE
   know the      git is        break       confirm      one sub-      checkpoints      reflect +
   targets       your reset    one fn       the spec     behavior      pass in order    check off
                 button                     is the       at a time
                                            failing test
```

---

## 0. Orient — know what "done" means *before* you touch code

- Read the stage's **`## Where you are`** diagram and **`## How it works`** Q&A. Answer
  the *predict-the-behavior* questions out loud. If you can't, read the real code first.
- Read this stage's block in **[`MASTERY.md`](MASTERY.md)** — those are the statements
  you should be able to say by the end. They're your real target; the passing test is
  just the proof.

## 1. Snapshot — make your reset button

You're on the `onboarding` branch; `main` holds the pristine original. Before gutting,
make sure the file you're about to break is clean so you can always get back:

```bash
git status --short          # should be clean for the file you're touching
```

If you need the original back at any point: `git restore <the-file>` (from `main`).

## 2. Gut — break exactly one thing

Paste the stage's **`## Step 1 — Set up this stage`** prompt to me. It replaces *one
function's body* with `raise NotImplementedError("Stage NN")` and keeps the signature,
docstring, and decorators. Nothing else in the repo changes — this is the whole point.
You then ask me to show you the signature + the fields/collaborators so you know the
contract.

## 3. Go red — find the failing test (it's your spec)

Run the stage's covering test and **confirm it fails**:

```bash
uv run pytest <the path from the stage> -x
```

A red test here is good — it proves the test actually exercises the code you gutted.
Read the failure. That test is the specification: it already encodes what "correct"
means. (See [`TESTING.md`](TESTING.md) → "How to read a failing test as a spec.")

> If gutting the function does **not** turn a test red, you gutted the wrong thing, or
> the function is only covered end-to-end — check the stage's notes for the thin-mock
> path (stages 01, 02, 03, 18, 19).

## 4. Rebuild — one sub-behavior at a time

Work the **`## Your tasks`** list top to bottom. Task 1 is always *read and trace the
real implementation*. Then implement the happy path, then each named edge case. Don't
write the whole function and hope — implement the smallest piece that turns the next
checkpoint green.

## 5. Go green — checkpoints in order

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

## 6. Close — reflect, check off, optionally commit

- Do the **`## What you learned`** reflection and re-answer the MASTERY statements.
  If any still feels shaky, that's the signal to reread — not the test, the *code*.
- Tick the stage in **[`PROGRESS.md`](PROGRESS.md)**.
- Optional: `git commit -am "stage NN done"` and `git push` to back up your fork.
- Move to the **`## Next →`** link.

---

## Hints & "where do I get the other functions?"

A real question, and the answer is reassuring:

- **You only ever gut ONE function.** Everything that function *calls* — helpers,
  earlier-stage methods, frame classes — still exists, untouched. You are filling one
  hole in a working machine, not building from an empty file. So you rarely "need to
  get" another function; you just *call the one that's already there*.

- **What a stage leans on is listed for you.** The `Depends on` column in
  **[`00-START-HERE.md`](00-START-HERE.md)** names the earlier stages whose concepts
  this one builds on. Example: stage 07 (interruption handling) leans on stage 03's
  `FrameQueue.reset` and stage 06's `broadcast_interruption` — both already implemented
  (by the repo, or by you if you did those stages). If a call in your rebuilt function
  looks unfamiliar, it's almost always one of those — go reread that stage's page.

- **The setup prompt hands you the contract.** When you gut a function, ask me to show
  the signature *and* the fields/collaborators it uses (the stage prompts already say
  this). That list IS your hint sheet — those are the exact things you're allowed to
  call.

- **Stuck on the logic itself?** Use the page's `## Stuck?` prompt — it makes me ask
  you a pointed question instead of handing over the answer. And `git restore <file>`
  lets you peek at the real implementation, then re-gut and try again. Reading the real
  code is allowed and encouraged (task 1 of every stage); *copying it blind* is what
  wastes the exercise.

How much to lean on hints is up to you. First try: orient, gut, and rebuild from the
test alone. Only open the real code when you're genuinely stuck — that struggle is
where the learning is.

→ Back to [`00-START-HERE.md`](00-START-HERE.md)
