# Stage 04 — Processor linking

## Where you are

```
    .link(B)           .link(C)
A ──────────► B ──────────► C
  ◄──────────   ◄──────────
   B._prev = A   C._prev = B

      _next ►   downstream (frames flow this way)
      _prev ◄   upstream   (errors / acks flow this way)
```

`link(next)` is the single call that wires one edge of this chain. When
`Pipeline._link_processors` runs during construction it calls
`processors[i].link(processors[i+1])` for every adjacent pair, building the
full doubly-linked list before any frame ever moves.

---

## How it works (orientation — answer these before you code)

**Q: At a high level, what does `A.link(B)` do?**
A: It makes `B` the downstream neighbour of `A` — that is, any frame `A` pushes
downstream will be queued into `B`. It also records the reverse pointer so `B`
knows who is upstream of it.

**Q: Is it one-directional, or does it also set the other side's `_prev`?**
A: Both directions are set in a single call. `link(processor)` sets
`self._next = processor` *and* `processor._prev = self`. One call wires both
pointers.

**Q: What fields are set, and where are they declared?**
A: `_next: FrameProcessor | None` and `_prev: FrameProcessor | None`, both
initialised to `None` in `FrameProcessor.__init__` (lines 211-212 of
`src/pipecat/processors/frame_processor.py`). The public read-only views are the
`next` and `previous` properties (lines 325-340).

**Q: Where would you change how an entire pipeline is wired?**
A: `Pipeline._link_processors` in `src/pipecat/pipeline/pipeline.py` (line 207).
It iterates `self._processors` and calls `.link()` on each adjacent pair. That
is covered in stage 08.

**Predict-the-behavior questions**

1. You have `A`, `B`, `C`. You call `A.link(B)` then `B.link(C)`. What is
   `C._prev`? What is `A._next._next`?
2. What happens to frames pushed upstream (`FrameDirection.UPSTREAM`) if `_prev`
   is `None`? Trace the routing logic at line 902 of `frame_processor.py`.
3. If you call `A.link(B)` and then later call `A.link(C)`, what happens to `B`?
   Does `B._prev` still point at `A`?

---

## Why this matters

If `link()` only sets one pointer and forgets the other, upstream frames (errors,
end-of-stream signals) silently vanish because `_prev` is `None` and the routing
guard at line 902 drops them. If the pointers point to the wrong processor,
frames either skip stages or loop — both bugs are invisible until you trace the
chain by hand. When debugging "frames aren't reaching the next processor," the
first thing to check is whether `processor.next` returns what you expect; a
broken `link()` is the most common root cause.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/frame_processor.py`. Find `FrameProcessor.link`.
> Replace its body with `raise NotImplementedError("Stage 04")` but KEEP the
> signature, docstring, and decorators exactly. Don't touch any other code. Then
> show me the signature and the `_next`/`_prev` fields so I know the contract.

---

## Your tasks (in order)

- [ ] **Read and trace.** Open `src/pipecat/processors/frame_processor.py`.
  Read `__init__` (around line 210) to see where `_next` and `_prev` are
  declared. Read the `next` and `previous` properties (around line 325). Read
  `push_frame` routing logic (around line 889) so you see exactly *how* `_next`
  and `_prev` are consumed at runtime. No code yet.

- [ ] **State the contract in your own words.** Before writing a single line of
  `link`, answer: (a) which field(s) does it set? (b) on which object(s)? (c)
  what are the types? Write this as a comment above your implementation.

- [ ] **Gut check — predict the chain.** On paper (or in a comment), draw the
  state of `_next`/`_prev` after `Pipeline([A, B, C])` finishes
  `_link_processors`. Verify your prediction against `Pipeline._link_processors`
  in `src/pipecat/pipeline/pipeline.py` (line 207).

- [ ] **Implement — happy path (single processor).** Write the body of `link`:
  set `_next` on `self`, set `_prev` on the incoming processor, keep the log
  line. Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_single
  ```
  It should go green.

- [ ] **Multi-processor chain.** No code change needed — the same two-line body
  handles chains of any length because `_link_processors` just calls `link`
  repeatedly. Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_multiple
  ```
  Confirm it passes and understand *why* three processors need only two `link`
  calls.

- [ ] **Edge case — re-linking (named task).** Consider: what happens to `B._prev`
  if you call `A.link(B)` and then `A.link(C)`? `B._prev` still points at `A`,
  but `A._next` is now `C`. Is that a problem in Pipecat's use of `link`? Check
  `_link_processors` — is `link` ever called more than once on the same upstream
  processor? Write a one-sentence note on whether the current implementation
  needs a guard.

- [ ] **Reflect.** In your own words: (a) why does `link` set *both* `_next` and
  `_prev` instead of letting each processor set its own pointer? (b) what would
  break if you only set `_next` and skipped `_prev`?

---

## Stuck? (paste into Claude)

> I'm reimplementing `FrameProcessor.link`. Here's my attempt: [code]. Don't
> give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`link()` is the foundation of every frame's journey: a two-pointer handshake
(`self._next = processor; processor._prev = self`) that takes two isolated
objects and makes them neighbours. When you see frames disappearing mid-pipeline,
open a REPL, grab the processor that *should* have forwarded them, and check
`processor.next` — if it is `None`, `link` was either never called or called in
the wrong order. All the routing logic in `push_frame` is guarded by these two
pointers, so a broken link is always the first hypothesis to rule out.

---

## Next → 05-push-routing.md
