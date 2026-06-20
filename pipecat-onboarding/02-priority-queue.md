# Stage 02 — Priority-queue routing

## Where you are

```
Pipeline
  └── FrameProcessor (one box)
        │
        ├── input_queue  ◄─── YOU ARE HERE
        │     (FrameProcessorQueue — asyncio.PriorityQueue)
        │
        │     slot (1, counter, item)  ← SystemFrame   ← HIGH_PRIORITY
        │     slot (2, counter, item)  ← DataFrame     ← LOW_PRIORITY
        │     slot (2, counter, item)  ← ControlFrame  ← LOW_PRIORITY
        │     ...
        │
        │     get() always returns the lowest-numbered tuple first
        │     → SystemFrames cut the line ahead of all other frames
        │
        └── _process_frame_task  (consumes the queue)
```

## How it works (orientation — answer these before you code)

**Q: Why does a processor need a PRIORITY queue, not a plain FIFO?**

A voice AI pipeline generates a continuous stream of audio frames (one small
chunk every ~20 ms). If the user says "stop" while the bot is speaking, an
`InterruptionFrame` (a `SystemFrame`) arrives in the queue *behind* dozens of
already-queued `TTSAudioRawFrame` objects (also `DataFrame`s). With a plain FIFO
the bot keeps talking until every queued audio chunk drains — a multi-second lag.
With a priority queue the `InterruptionFrame` jumps straight to the front and the
processor acts on it immediately.

**Q: How is priority decided — what makes a frame a SystemFrame, and what values are used?**

`SystemFrame` is a plain base class in `src/pipecat/frames/frames.py` (line 100).
Any frame that inherits from it (e.g. `InterruptionFrame`, `CancelFrame`,
`StartFrame`, `StopFrame`, `FrameProcessorPauseUrgentFrame`) is a system frame.
`DataFrame` and `ControlFrame` are *not* system frames.

`FrameProcessorQueue` stores three-tuple items in the underlying
`asyncio.PriorityQueue`:

```
(priority_bucket, counter, original_item)
```

- `HIGH_PRIORITY = 1` for `SystemFrame`s
- `LOW_PRIORITY  = 2` for everything else

`asyncio.PriorityQueue` uses Python tuple comparison: it compares the first
element first, so bucket `1 < 2` always wins. Within the same bucket, items
are ordered by `counter`, an integer that strictly increases with each `put`
call. This means frames in the *same* priority bucket come out in arrival order
(FIFO within tier).

Two separate counters are kept — `__high_counter` and `__low_counter` — so
they never interfere with each other's sort order across buckets.

**Q: Where would you change which frames are treated as high-priority?**

In `FrameProcessorQueue.put` in
`src/pipecat/processors/frame_processor.py` (lines 148–154). The only test is
`isinstance(frame, SystemFrame)`. To promote a specific `ControlFrame` subclass,
you would add an `or isinstance(frame, MyUrgentControlFrame)` branch. To add a
*third* tier you would introduce a `MEDIUM_PRIORITY = 1.5` constant (floats work
in Python tuple comparison) and a matching counter.

**Predict-the-behavior questions — think before you look:**

1. Two `InterruptionFrame`s arrive back-to-back, then a `TTSAudioRawFrame`.
   What order do they come out?
2. A `ControlFrame` (LOW_PRIORITY) arrives before an `InterruptionFrame`
   (HIGH_PRIORITY). Which is dequeued first?
3. You subclass `DataFrame` *and* `SystemFrame` in the same class. What priority
   does it get?

## Why this matters

An interruption signal must reach every processor *instantly* — if it queues
behind even a half-second of buffered audio frames, the user hears the bot keep
talking after they've already started their next sentence. That latency breaks
the conversational illusion entirely. Priority routing is the mechanism that
makes barge-in feel snappy.

If you ever debug "the bot won't stop talking" and processor logs show
`InterruptionFrame` arriving well after `TTSAudioRawFrame`s, check whether the
queue is actually a `FrameProcessorQueue` — a plain `asyncio.Queue` substituted
somewhere upstream would silently degrade to FIFO and swallow the priority
guarantee.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/frame_processor.py`. Find `FrameProcessorQueue.put`.
> Replace its body with `raise NotImplementedError("Stage 02")` but KEEP the
> signature, docstring, and decorators exactly. Don't touch any other code.
> Then show me:
> 1. The full method signature and its `item` type annotation.
> 2. The class-level constants (`HIGH_PRIORITY`, `LOW_PRIORITY`) and both
>    instance counters (`__high_counter`, `__low_counter`).
> 3. What the *parent* class's `put` expects as its argument (i.e. what tuple
>    shape does `asyncio.PriorityQueue.put` ultimately receive?).

## Your tasks (in order)

- [ ] **Task 1 — Read and trace.** Open `src/pipecat/processors/frame_processor.py`
  and read the full `FrameProcessorQueue` class (lines 119–167). Then open
  `src/pipecat/frames/frames.py` and read `SystemFrame`, `DataFrame`,
  `ControlFrame`, and `UninterruptibleFrame` (lines 60–152). Write in your own
  words: what is the difference between `SystemFrame` and `UninterruptibleFrame`?
  (One affects *queue priority*; the other affects *interruption survival*.)

- [ ] **Task 2 — State the contract.** Before writing a single line, write out
  the observable contract of `put` as bullet points:
  - Input type and shape of `item`.
  - What tuple does it store in the parent `PriorityQueue`?
  - How does the tuple differ for a `SystemFrame` vs any other frame?
  - What happens to the counter on each call?
  - What happens if `item[0]` is not a `Frame` at all (the docstring mentions
    a "watchdog cancellation sentinel")?

- [ ] **Task 3 — Gut check.** Before implementing, answer the predict-the-behavior
  questions from the orientation section. Write your predictions down. You will
  verify them after the checkpoint.

- [ ] **Task 4 — Implement priority assignment and tiebreaker.**
  Restore the `put` body. You need:
  1. Unpack `frame` from `item`.
  2. `isinstance(frame, SystemFrame)` → `HIGH_PRIORITY`, increment
     `__high_counter`, wrap in `(HIGH_PRIORITY, counter, item)`.
  3. Otherwise → `LOW_PRIORITY`, increment `__low_counter`, wrap similarly.
  4. Call `super().put(wrapped_tuple)`.

  **Checkpoint (thin mock — run this in a scratch file):**

  ```python
  import asyncio
  from dataclasses import dataclass
  from pipecat.processors.frame_processor import FrameProcessorQueue
  from pipecat.frames.frames import DataFrame, SystemFrame, FrameDirection

  @dataclass
  class MyDataFrame(DataFrame): pass
  @dataclass
  class MySystemFrame(SystemFrame): pass

  async def test():
      q = FrameProcessorQueue()
      data_frame   = MyDataFrame()
      system_frame = MySystemFrame()
      await q.put((data_frame,   FrameDirection.DOWNSTREAM, None))
      await q.put((system_frame, FrameDirection.DOWNSTREAM, None))
      first  = await q.get()
      second = await q.get()
      assert isinstance(first[0],  MySystemFrame), f"Expected SystemFrame first, got {type(first[0])}"
      assert isinstance(second[0], MyDataFrame),   f"Expected DataFrame second, got {type(second[0])}"
      print("Checkpoint passed: SystemFrame came out before DataFrame")

  asyncio.run(test())
  ```

  Run it: `uv run python scratch_02.py` from the repo root.

- [ ] **Task 5 — Integration green-light.** Run the two integration tests that
  exercise priority routing end-to-end through a real pipeline:

  ```bash
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_interruptible_frames -v
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_uninterruptible_frames -v
  ```

  `test_interruptible_frames` checks that an `InterruptionFrame` (SystemFrame)
  overtakes a queued `TestInterruptibleFrame` (DataFrame) even when the processor
  is sleeping mid-frame. `test_uninterruptible_frames` checks that a
  `TestUninterruptibleFrame(DataFrame, UninterruptibleFrame)` *survives* the
  interruption — it still comes out, just after the `InterruptionFrame`. Both
  must be green.

- [ ] **Task 6 — Edge case: equal-priority FIFO ordering.** Add a second
  assertion to your scratch file: put three `DataFrame`s in order A, B, C and
  assert they come out in the same order A, B, C. This verifies the tiebreaker
  counter provides FIFO within the LOW_PRIORITY bucket. Then do the same for two
  `SystemFrame`s (S1, S2) and assert S1 comes out before S2.

- [ ] **Task 7 — Reflect.** Go back to your Task 3 predictions. Were they all
  correct? If not, which one surprised you and why? Write two sentences connecting
  the tiebreaker counter to real-world barge-in latency: what would happen if
  there were *no* tiebreaker and the queue had to fall back on comparing Frame
  objects directly?

## Stuck? (paste into Claude)

> I'm reimplementing `FrameProcessorQueue.put`. Here's my attempt: [code].
> Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`FrameProcessorQueue` decouples *ordering policy* from *processing logic*: the
processor's consumer loop stays a simple `await queue.get()` loop, yet the queue
guarantees that steering signals (interruptions, stop, cancel) always arrive
before data payloads regardless of arrival order. That is the mechanism behind
barge-in latency — the gap between when a user starts speaking and when the bot
actually stops. Shrinking that gap comes down to ensuring `InterruptionFrame`
propagates without touching a backlog, which this queue enforces at the
per-processor level throughout the entire pipeline chain.

## Next → 03-frame-queue.md
