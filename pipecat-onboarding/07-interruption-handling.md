# Stage 07 — Interruption handling in a processor

## Where you are

```
         ┌──────────────────────────────────────────────────────────┐
         │              FrameProcessor (receiving side)             │
         │                                                          │
  ──►  queue_frame()  ──►  __process_queue  ──►  process_frame()   │
         │                      │                      ▲            │
         │             ┌────────┘                      │            │
         │             │  FrameQueue.reset()        currently       │
         │             │  (Stage 03)               processing       │
         │             │                        __process_current_  │
         │             │                             frame          │
         │             │                              │             │
         │    ◄── InterruptionFrame arrives           │             │
         │             │                              │             │
         │         _start_interruption() ◄────────────┘            │
         │          ┌──┴────────────────────────────┐               │
         │          │  Is __process_current_frame   │               │
         │          │  an UninterruptibleFrame?     │               │
         │          └──┬────────────────────────────┘               │
         │       YES   │         NO                                 │
         │      flush  │    cancel task  ──► spawn fresh task       │
         │      only   │    (also flushes via __reset_process_queue)│
         └─────────────┴──────────────────────────────────────────--┘
```

`_start_interruption` is the decision gate that separates "barge-in works" from "barge-in silently fails".

## How it works (orientation — answer these before you code)

**Q: What does `_start_interruption` do at a high level?**
A: When an `InterruptionFrame` arrives at a processor it calls `_start_interruption`. The method's job is to tear down whatever in-flight work the processor is doing so the pipeline can start fresh — while guaranteeing that certain privileged frames are never lost.

**Q: When does it cancel the process task, and when does it only flush?**
A: It inspects `self.__process_current_frame` (the frame being actively processed right now). If that frame is an instance of `UninterruptibleFrame`, the task must not be cancelled — the current work must finish. In that case the method only calls `__reset_process_queue()`, which flushes interruptible frames from the queue while preserving any queued `UninterruptibleFrame` items (Stage 03). If the current frame is ordinary (interruptible), the method cancels the process task with `__cancel_process_task()` and then spawns a replacement with `__create_process_task()`. The `__create_process_task` call internally calls `__reset_process_task`, which resets the event and calls `__reset_process_queue` — so the flush always happens, regardless of which branch is taken.

**Q: How does `FrameQueue.reset` from Stage 03 fit in?**
A: `FrameQueue.reset` (Stage 03) removes all interruptible frames from the queue but keeps any `UninterruptibleFrame` entries intact. Both branches of `_start_interruption` ultimately call it — directly in the uninterruptible branch, and indirectly (via `__reset_process_task`) in the interruptible branch.

**Q: Does it spawn a fresh task?**
A: Yes, but only in the interruptible branch. After cancelling the old task, `__create_process_task()` creates a new `asyncio.Task` running `__process_frame_task_handler`. In the uninterruptible branch, no new task is spawned; the existing task keeps running until the current frame finishes.

**Q: Where did the `InterruptionFrame` come from?**
A: Stage 06 — `broadcast_interruption()` (or `broadcast_frame(InterruptionFrame)`) fires an `InterruptionFrame` both upstream and downstream through the pipeline. Every `FrameProcessor` receives it via its `_handle_system_frame` logic, which calls `_start_interruption` before the frame is further propagated.

**Predict-the-behavior questions — answer before running tests:**

1. A processor is sleeping 400 ms inside `process_frame` on an ordinary `TextFrame`. An `InterruptionFrame` arrives. What do you expect to see come out of the pipeline?
2. A processor is sleeping 400 ms on an `UninterruptibleFrame`. An `InterruptionFrame` arrives. Does the uninterruptible frame make it through? Does the interruption frame make it through?
3. An `EndFrame` is sitting in the process queue when an `InterruptionFrame` arrives. What happens to it?

## Why this matters

This is the mechanism that makes a voice bot actually stop speaking mid-sentence when the user speaks: without it, the TTS audio frames already queued would keep flowing even after the interruption signal arrived.

If you ever debug a bot that "stops but then resumes the old reply," the first place to look is `_start_interruption` — either the queue flush is not happening, or the process task is not being cancelled and recreated, so old frames drain through after the new turn begins.

Getting the uninterruptible / terminal-frame logic right prevents a subtler failure: a pipeline that hangs because an `EndFrame` or `StopFrame` was discarded during an interruption, leaving the pipeline waiting for a shutdown signal that never comes.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/frame_processor.py`. Find `FrameProcessor._start_interruption`. Replace its body with `raise NotImplementedError("Stage 07")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the process-task field, and how it checks the current frame so I know the contract.

## Your tasks (in order)

- [ ] **Read the target.** Open `src/pipecat/processors/frame_processor.py`. Read `_start_interruption` (line ~853) in full. Then trace the two helpers it calls: `__reset_process_queue` and `__cancel_process_task` / `__create_process_task`. Note what each does and what field each touches.

- [ ] **State the contract.** Before writing any code, write down in your own words: (a) what input triggers this method, (b) what the two branches do and why they differ, (c) which Stage 03 method does the actual queue flush, and (d) what field holds the currently-executing frame.

- [ ] **Gut check — predict test outcomes.** For each of the three "predict" questions in the orientation section, write your expected output. Then look at the expected frames in the three covering tests to verify your mental model.

- [ ] **Implement the happy path (interruptible frame).** Write the interruptible branch: cancel the current process task, then create a fresh one. Run the first test:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_interruptible_frames
  ```

- [ ] **Implement the uninterruptible edge case.** Add the guard that detects `UninterruptibleFrame` and only flushes instead of cancelling. Run:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_uninterruptible_frames
  ```

- [ ] **Verify terminal frames survive (named edge case).** `EndFrame` and `StopFrame` are `UninterruptibleFrame` subclasses, so they must survive a queue flush. Run:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_terminal_frames_survive_interruption
  ```

- [ ] **Run all three together.** Confirm no regressions:
  ```
  uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_interruptible_frames tests/test_frame_processor.py::TestFrameProcessor::test_uninterruptible_frames tests/test_frame_processor.py::TestFrameProcessor::test_terminal_frames_survive_interruption
  ```

- [ ] **Reflect.** Answer: why does `__create_process_task` implicitly flush the queue even though you didn't call `__reset_process_queue` directly in the interruptible branch? What would break if you called `__cancel_process_task` but forgot `__create_process_task`?

## Stuck? (paste into Claude)

> I'm reimplementing `_start_interruption`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`_start_interruption` is the receiving side of barge-in: it is the exact point where "user interrupted the bot" turns into "in-flight frames are discarded." When debugging barge-in correctness, the question to ask is: did `_start_interruption` fire, did it take the right branch (cancel vs. flush-only), and did `FrameQueue.reset` leave `UninterruptibleFrame` items intact? All three must be true for interruption to work correctly without losing terminal frames that would stall the pipeline.

## Next → 08-pipeline-wiring.md
