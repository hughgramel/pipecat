# Pipecat Internals — A Reconstruction Practicum

## Assignment Procedure

Read this document once before beginning Assignment 01. Every assignment in this
practicum follows the same seven-step procedure. The procedure exists to keep the work
disciplined: you establish what correct behavior is *before* writing code, and you
verify each piece independently rather than writing the whole function and hoping.

### Procedure at a glance

| Step | Phase | Action |
|------|-------|--------|
| 0 | Orientation | Determine what the function is required to do |
| 1 | Baseline | Confirm the source file is unmodified — your point of return |
| 2 | Isolation | Remove the body of the single target function |
| 3 | Specification | Run the covering test and confirm it fails; the failing test is your specification |
| 4 | Implementation | Reconstruct the function, one behavior at a time |
| 5 | Verification | Make each checkpoint test pass, in order |
| 6 | Review | Reflect on the outcomes and record completion |

---

### Step 0 — Orientation

Determine the requirements before modifying anything.

- Read the assignment's **"Where you are"** diagram and **"How it works"** discussion.
  Attempt the *predict-the-behavior* questions; if you cannot answer them, study the
  source before proceeding.
- Read this assignment's entry in the Learning Outcomes handout
  ([`MASTERY.md`](MASTERY.md)). Those statements are the objective of the assignment;
  the passing test is merely evidence that you have met it.

### Step 1 — Baseline

All work occurs on the `onboarding` branch; the `main` branch holds the unmodified
original. Before making changes, confirm that the file you intend to modify is clean,
so that you can always return to a known-good state:

```bash
git status --short          # the target file should be unmodified
```

To restore the original implementation at any point: `git restore <file>`.

### Step 2 — Isolation

Provide the assignment's **"Set up this assignment"** prompt to the assistant. It
replaces the body of *one* function with `raise NotImplementedError("Stage NN")`,
preserving the signature, docstring, and decorators. No other code in the repository
changes. Then request the function's signature together with the fields and
collaborators it relies upon, so that you understand the contract you are implementing.

### Step 3 — Specification

Run the assignment's covering test and **confirm that it fails**:

```bash
uv run pytest <path given in the assignment> -x
```

A failure at this point is expected and desirable: it demonstrates that the test
genuinely exercises the function you removed. Study the failure. The test is the
specification — it already encodes the definition of correct behavior. (See the
Laboratory Reference, "Reading a failing test as a specification.")

> If removing the function does **not** cause a test to fail, you have removed the wrong
> function, or the function is covered only by end-to-end tests. Consult the
> assignment's notes for the lightweight-verification procedure (Assignments 01, 02, 03,
> 18, 19).

### Step 4 — Implementation

Work through the assignment's task list in order. The first task is always to read and
trace the original implementation. Implement the principal (happy-path) behavior first,
then each named edge case. Do not implement the entire function at once; implement the
smallest piece that satisfies the next checkpoint.

### Step 5 — Verification

Each task corresponds to a specific `pytest` invocation. Run it after completing each
behavior:

```bash
uv run pytest <the task's test> -q          # a single checkpoint
uv run pytest <the assignment's file> --lf  # re-run only what failed last
```

When all checkpoints pass, run the assignment's complete test file once to confirm that
no neighboring test has regressed:

```bash
uv run pytest tests/test_<assignment>.py -q
```

### Step 6 — Review

- Complete the assignment's **"What you learned"** reflection and re-state the outcomes
  from the Learning Outcomes handout. Any statement that remains unclear indicates
  material to revisit — in the source, not merely the test.
- Record completion in the Completion Record ([`PROGRESS.md`](PROGRESS.md)).
- Optionally, commit and push your work: `git commit -am "Assignment NN complete"`.
- Continue to the assignment named under **"Next →"**.

---

### Permitted resources and academic integrity

This practicum depends on productive struggle. The following policy governs how much
assistance to draw on.

- **Only one function is ever removed.** Everything that function calls — helpers,
  methods reconstructed in earlier assignments, frame classes — remains present and
  unmodified. You are completing one absent piece of a working system, not building from
  an empty file. You therefore rarely need to "obtain" another function; you call the
  one already present.

- **The dependencies of each assignment are stated.** The "Prereq." column of the
  assignment schedule in [`00-START-HERE.md`](00-START-HERE.md) names the earlier
  assignments on which each one builds. For example, Assignment 07 (interruption
  handling) relies on Assignment 03's `FrameQueue.reset` and Assignment 06's
  `broadcast_interruption`, both of which already exist. If a call in your
  reconstruction appears unfamiliar, it almost certainly belongs to one of these; revisit
  that assignment.

- **The setup prompt supplies the contract.** When the function is isolated, request its
  signature together with the fields and collaborators it uses. That list constitutes
  the permitted means by which your implementation may accomplish its task.

- **Consulting the original is permitted; copying it is not.** Reading and tracing the
  original implementation is the first task of every assignment and is encouraged.
  Transcribing it without understanding defeats the purpose of the exercise. When you are
  genuinely stuck, the assignment's "Stuck?" prompt directs the assistant to pose a
  guiding question rather than supply the answer; `git restore <file>` allows you to
  examine the original, after which you should isolate the function again and retry from
  your own understanding.

The recommended practice is to attempt each assignment from the specification (the
failing test) alone, and to consult the original source only when genuinely blocked.

→ Return to [`00-START-HERE.md`](00-START-HERE.md).
