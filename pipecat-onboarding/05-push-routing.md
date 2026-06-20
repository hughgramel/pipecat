# Stage 05 — Push-frame routing

## Where you are

```
┌──────────┐   _next   ┌──────────┐   _next   ┌──────────┐
│    A     │ ────────► │    B     │ ────────► │    C     │
│          │ ◄──────── │          │ ◄──────── │          │
└──────────┘   _prev   └──────────┘   _prev   └──────────┘

B wants to push a frame DOWNSTREAM (toward C):
  B.__internal_push_frame(frame, DOWNSTREAM)
      └─► B._next.queue_frame(frame, DOWNSTREAM)   # i.e. C.queue_frame(...)

B wants to push a frame UPSTREAM (toward A):
  B.__internal_push_frame(frame, UPSTREAM)
      └─► B._prev.queue_frame(frame, UPSTREAM)     # i.e. A.queue_frame(...)

Decision point: direction == DOWNSTREAM? → use _next
                direction == UPSTREAM?   → use _prev
```

Stage 04 established `_next`/`_prev` by linking processors in the pipeline.
Stage 02 showed that `queue_frame` puts the frame onto the priority input queue
of the *receiving* processor, where it will be dequeued and processed.
This stage is the bridge between those two: the moment a frame actually crosses
the boundary between processors.

---

## How it works (orientation — answer these before you code)

**Q: At a high level, what does `__internal_push_frame` do?**

A: It reads the direction, picks the right neighbor (`_next` for DOWNSTREAM,
`_prev` for UPSTREAM), optionally notifies an observer, then calls
`neighbor.queue_frame(frame, direction)` to hand the frame off.

**Q: Walk through the runtime branches.**

- `direction == DOWNSTREAM and self._next is not None`
  → builds a `FramePushed` observer payload (if `_observer` exists), calls
  `await self._observer.on_push_frame(data)`, then
  `await self._next.queue_frame(frame, direction)`.
- `direction == UPSTREAM and self._prev is not None`
  → same observer dance, then `await self._prev.queue_frame(frame, direction)`.
- If the matching neighbor is `None` (end of chain), both branches are skipped
  silently — the frame is dropped.

**Q: Where do the `on_before_push_frame` / `on_after_push_frame` events fire?**

A: In the *caller*, `push_frame()`, which wraps `__internal_push_frame`:

```python
await self._call_event_handler("on_before_push_frame", frame)
await self.__internal_push_frame(frame, direction)
await self._call_event_handler("on_after_push_frame", frame)
```

So the hooks are synchronous event handlers on the pushing processor, not inside
the internal method itself.

**Q: Where would you add a new observer or change routing?**

A: `__internal_push_frame` is the single choke point. To intercept every
cross-processor handoff — for logging, filtering, or rerouting — this is where
you'd add logic.

**Predict-the-behavior questions**

1. Processor B is the last in a three-processor chain (`_next` is `None`).
   B calls `await self.push_frame(frame, FrameDirection.DOWNSTREAM)`.
   What happens?
2. You set `_observer` on processor B. Which method on the observer is called
   when B *processes* a frame vs. when B *pushes* a frame?
3. Processor B calls `push_frame(frame, UPSTREAM)` but `_prev` is `None`
   (B is the head of the chain). Is an exception raised?

---

## Why this matters

`__internal_push_frame` is the literal crossing point — the line of code that
moves a frame from one processor's output to the next processor's input queue.
Every "my processor handled the frame but nothing arrived downstream" bug traces
back here: either the direction was wrong, the neighbor link was never set
(stage 04), or the neighbor's queue was not accepting frames (stage 02). Reading
this method is always step one when debugging a silent pipeline.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/frame_processor.py`. Find the private
> `__internal_push_frame` method on `FrameProcessor`. Replace its body with
> `raise NotImplementedError("Stage 05")` but KEEP the signature, docstring,
> and decorators exactly. Don't touch any other code. Then show me the
> signature, the `FrameDirection` enum values, and what `_next`/`_prev` expose
> (their type and how they are set) so I know the contract before I write
> anything.

---

## Your tasks (in order)

- [ ] **Read and trace.** Read `__internal_push_frame` in full (lines 880–915).
  Then trace a single `TextFrame` pushed DOWNSTREAM through a three-processor
  pipeline: which lines in `push_frame`, `__internal_push_frame`, and
  `queue_frame` execute, in order?

- [ ] **State the contract.** Before writing any code, write down:
  - Inputs: `frame` (a `Frame` instance) and `direction` (a `FrameDirection`
    enum value — `DOWNSTREAM = 1` or `UPSTREAM = 2`).
  - Side effects: calls `queue_frame` on the appropriate neighbor; optionally
    calls `_observer.on_push_frame`.
  - Return value: `None` (coroutine, no return).
  - Silent no-op condition: neighbor in the requested direction is `None`.

- [ ] **Gut check.** Answer the three predict-the-behavior questions above
  before running any tests.

- [ ] **Implement downstream routing.** Write the DOWNSTREAM branch only:
  check `direction == FrameDirection.DOWNSTREAM and self._next`, then call
  `await self._next.queue_frame(frame, direction)`. Ignore observers for now.
  Run:

  ```bash
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_frame -v
  ```

  The test sends a frame through a three-processor pipeline and verifies a frame
  arrives downstream. It should pass once routing is correct.

- [ ] **Add upstream routing.** Add the UPSTREAM branch (`self._prev`). Run
  the same test again to confirm both directions work — `test_broadcast_frame`
  exercises both.

- [ ] **Add observer notification.** Before each `queue_frame` call, check
  `self._observer`; if set, build a `FramePushed` object and call
  `await self._observer.on_push_frame(data)`. Then run:

  ```bash
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_before_after_events -v
  ```

  This test registers `on_before_push_frame` and `on_after_push_frame` handlers
  on a processor and asserts they were called. Those handlers live in
  `push_frame`, not here, but the test only passes if the full `push_frame` →
  `__internal_push_frame` chain works end-to-end.

- [ ] **Name and handle the end-of-chain edge case.** The chain ends when
  `_next` is `None` (tail going DOWNSTREAM) or `_prev` is `None` (head going
  UPSTREAM). Confirm your implementation silently does nothing in these cases
  rather than raising. Write a one-sentence comment in your code naming this
  "end of chain — frame is dropped silently."

- [ ] **Reflect.** Without looking at the source, describe in two sentences:
  (1) what happens to a frame when `push_frame(frame, UPSTREAM)` is called on
  the first processor in the chain, and (2) which method you would set a
  breakpoint on to intercept every inter-processor handoff.

---

## Stuck? (paste into Claude)

> I'm reimplementing `__internal_push_frame`. Here's my attempt: [code]. Don't
> give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

Every "frames not flowing" bug starts here. If a processor's `process_frame`
runs but nothing arrives at the next processor, the first place to look is
`__internal_push_frame`: Was the direction correct? Was `_next`/`_prev` set by
the pipeline linker (stage 04)? Did the neighbor's `queue_frame` (stage 02)
actually enqueue the frame, or was it in cancel/pause state? This method is the
handoff; everything else is setup for it.

---

## Next → 06-interruption-broadcast.md
