# Stage 03 — Frame queue with uninterruptible tracking

## Where you are

```
FrameProcessor (internals)
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [Stage 02] _push_frame_task                                │
│  Priority Queue  ──────────────────────────────────────►    │
│  (SystemFrames first)                                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ★ THIS STAGE: _frame_queue  (FrameQueue / FIFO)   │   │
│  │                                                     │   │
│  │  TextFrame  →  AudioFrame  →  ★ EndFrame            │   │
│  │  [normal]      [normal]       [MUST SURVIVE barge] │   │
│  │                                                     │   │
│  │  has_uninterruptible: True  (O(1) flag)             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  When a barge-in arrives → FrameQueue.reset() is called    │
│  Normal frames are discarded; EndFrame stays in queue       │
└─────────────────────────────────────────────────────────────┘
```

The stage-02 priority queue dispatches incoming frames to `process_frame`.
`FrameQueue` is the *outgoing* FIFO — it holds frames that have been processed
and are waiting to be pushed to the next processor.  "EndFrame must survive a
barge-in" is the invariant that keeps a voice call from hanging after the user
speaks over the bot.

---

## How it works (orientation — answer these before you code)

**Q: What does `reset()` do at a high level, and when is it called?**

`reset()` drains every item whose underlying `Frame` is *not* an
`UninterruptibleFrame`, then re-enqueues the survivors in the same order.
It is called by `FrameProcessor` when an `InterruptionFrame` is processed —
the bot is being "barged in on" and the pending output (speech audio,
text, etc.) should be discarded so the bot can stop talking immediately.

**Q: What is `has_uninterruptible` and why is it O(1)?**

It is a simple integer counter `_uninterruptible_count` exposed as a boolean
property.  Every time `_put` is called with an `UninterruptibleFrame`, the
counter increments; every time `_get` removes one, it decrements.  The flag
tells the interruption handler whether it is even worth calling `reset()`:
if `has_uninterruptible` is `False` there are no survivors, so the queue can
be cleared in a single pass without keeping a `kept` list.

**Q: What counts as uninterruptible?**

Anything that is an instance of `UninterruptibleFrame` — the mixin introduced
in stage 01 (`src/pipecat/frames/frames.py`, line 142).  Concrete examples in
the codebase: `EndFrame`, `StopFrame`, `PipelineFlushFrame`,
`FunctionCallResultFrame`, `ServiceUpdateSettingsFrame`.  None of these should
ever be lost to a barge-in.

**Q: Where would you change which frames survive a flush?**

Add `UninterruptibleFrame` as a base class (or mixin) to the frame class in
`src/pipecat/frames/frames.py`.  No changes to `FrameQueue` itself are needed —
the queue's logic is driven entirely by `isinstance(frame, UninterruptibleFrame)`.

**Predict-the-behavior questions (think before you run the tests):**

1. You enqueue `[TextFrame, EndFrame, AudioFrame]` then call `reset()`.
   What is in the queue afterward?  What is `has_uninterruptible`?

2. You enqueue `[EndFrame, StopFrame]` and call `reset()`.
   Are both still in the queue?  Does the order change?

3. You enqueue only `[TextFrame, AudioFrame]` and call `reset()`.
   What does `has_uninterruptible` return before and after the call?

---

## Why this matters

If `EndFrame` (which signals clean shutdown to every downstream processor) were
flushed during a barge-in, no processor would ever receive it and the pipeline
would hang indefinitely — the call would never end.  The same applies to
`StopFrame` used for mid-call resets.  Understanding this flush-but-preserve
pattern is essential when debugging "the bot hangs after the user interrupts":
the first place to look is whether the terminal frame was accidentally made
interruptible, or whether `reset()` is being called at the wrong moment.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/utils/frame_queue.py`. Find `FrameQueue.reset`. Replace its
> body with `raise NotImplementedError("Stage 03")` but KEEP the signature,
> docstring, and decorators exactly. Don't touch any other code. Then show me
> the full signature, the queue's internal fields (`_frame_getter`,
> `_uninterruptible_count`), and explain what the `has_uninterruptible` property
> returns so I know the contract before I write anything.

---

## Your tasks (in order)

- [ ] **Read and trace** — read `src/pipecat/utils/frame_queue.py` in full.
  Trace the path of a single `EndFrame` from `put_nowait` → `_put` →
  `_uninterruptible_count` → `get_nowait` → `_get` → counter decrement.
  Write the sequence in a comment before your implementation.

- [ ] **State the contract in words** — before touching code, write out:
  (a) what the queue contains after `reset()` if it had
  `[TextFrame, EndFrame, AudioFrame]`; (b) what `has_uninterruptible` is
  after that `reset()`; (c) what happens if the queue was already empty.

- [ ] **Identify the 3-5 sub-behaviors** — list them:
  1. Normal frames are removed.
  2. `UninterruptibleFrame` instances are preserved.
  3. Their relative order is preserved.
  4. `_uninterruptible_count` is correct after reset (check via `has_uninterruptible`).
  5. An empty queue is a no-op.

- [ ] **Implement the happy path first** — write a `reset()` that drains the
  queue into a temporary list, filters out non-uninterruptible items, and
  re-enqueues the rest.  Use `get_nowait` / `put_nowait` so the `_put`/`_get`
  hooks keep `_uninterruptible_count` consistent.

- [ ] **Thin-mock checkpoint** — paste and run this snippet in a Python REPL
  (no pytest needed):

  ```python
  import asyncio
  from pipecat.utils.frame_queue import FrameQueue
  from pipecat.frames.frames import TextFrame, EndFrame

  q = FrameQueue()
  q.put_nowait(TextFrame(text="hello"))
  q.put_nowait(EndFrame())

  assert q.qsize() == 2
  assert q.has_uninterruptible is True

  q.reset()

  assert q.qsize() == 1, f"expected 1, got {q.qsize()}"
  surviving = q.get_nowait()
  from pipecat.frames.frames import UninterruptibleFrame
  assert isinstance(surviving, UninterruptibleFrame), f"got {type(surviving)}"
  assert q.has_uninterruptible is False
  print("checkpoint passed")
  ```

- [ ] **Integration test — terminal frames survive** — run:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_terminal_frames_survive_interruption -v
  ```
  Read the test first (line 342 in `tests/test_frame_processor.py`): a
  `DelayAndInterruptProcessor` sleeps while `EndFrame` lands in the queue, then
  fires `InterruptionFrame`.  `EndFrame` must still reach `CaptureFrameProcessor`.

- [ ] **Integration test — StopFrame survives** — run:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_stop_frame_survives_interruption -v
  ```
  Same pattern but with `StopFrame` (line 394).

- [ ] **Edge case — the flag** — after your `reset()`, manually verify that
  `_uninterruptible_count` is still correct.  The tricky case: what if you
  enqueue two `EndFrame`s, call `reset()`, then `get_nowait()` both?  The
  counter should reach zero, not go negative.  Write a one-liner assertion for
  this and confirm it passes.

- [ ] **Reflect** — in 2-3 sentences, explain why using `get_nowait`/`put_nowait`
  (which go through `_get`/`_put`) is safer than directly manipulating
  `self._queue` (the internal deque), even though direct manipulation would be
  faster.

---

## Stuck? (paste into Claude)

> I'm reimplementing `FrameQueue.reset`. Here's my attempt: [code]. Don't give
> me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`FrameQueue.reset` is the mechanism that makes barge-in safe: it flushes
pending output without discarding terminal control frames.  When debugging a
pipeline that hangs after an interruption, the first question is "did `EndFrame`
survive `reset()`?" — check `has_uninterruptible` before and after, and confirm
the frame's class actually inherits `UninterruptibleFrame`.  When debugging a
call that *doesn't* stop cleanly even without interruption, the same flag tells
you whether an `EndFrame` is stuck waiting in the queue.

---

## Next → 04-processor-linking.md
