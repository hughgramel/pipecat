# Stage 06 — Interruption broadcast

## Where you are

```
                    ← InterruptionFrame (upstream)
[InputTransport] ←←←←←←←←←←[YourProcessor]→→→→→→→→→→ [TTS] → [OutputTransport]
                              broadcast_interruption()
                    → InterruptionFrame (downstream)
```

A processor in the middle of the chain calls `broadcast_interruption()`. Two
`InterruptionFrame` instances fly out simultaneously — one heading upstream
toward the input transport, one heading downstream toward the output. Every
processor in the chain receives it and reacts (stage 07).

---

## How it works (orientation — answer these before you code)

**Q: What does `broadcast_interruption` do at a high level, and who calls it?**

A: It is the signal that says "stop everything — the user is speaking." Any
processor can call it, but the most common caller is a VAD-based turn strategy
(stage 14) that detects voice activity mid-bot-response. It clears the caller's
own processing state and then sends `InterruptionFrame` in both directions so
every processor in the pipeline can react.

**Q: Does it push upstream AND downstream? Does it reset its own task first?**

A: Yes to both. The exact sequence is:
1. `self.__reset_process_task()` — drops the caller's own non-system frame
   queue and resets its processing event, so no leftover frames resume
   afterwards.
2. `await self.stop_all_metrics()` — cancels any in-flight timing metrics.
3. `await self.broadcast_frame(InterruptionFrame)` — creates two
   `InterruptionFrame` instances (one per direction), links them as siblings
   via `broadcast_sibling_id`, then pushes one downstream and one upstream.

The task reset happens *before* the push, so the interruption cannot be
blocked by the processor's own queue.

**Q: Is `InterruptionFrame` a `SystemFrame`? Why does that matter?**

A: Yes. `InterruptionFrame` inherits from `SystemFrame` (see
`src/pipecat/frames/frames.py:1018`). System frames bypass the normal FIFO
processing queue and go into the priority queue introduced in stage 02. That
means the interruption jumps ahead of any `DataFrame` already waiting — it
cannot be backpressured by a full queue of audio or text frames.

**Q: Where would you trigger an interruption in a real bot (other than inside a processor)?**

A: The `VADUserTurnStartStrategy` (stage 14) is the typical trigger. It detects
the user speaking via voice-activity detection and calls
`broadcast_interruption()` so the bot stops mid-sentence. Any processor can
also call it directly — for example, a "stop" button on the UI, a timeout
watchdog, or an LLM-tool result that cancels an earlier response.

**Q: Does `broadcast_interruption` block until every downstream processor has
handled the frame?**

A: No. It `await`s the *push* (enqueue into the next processor's queue), not
the *processing*. Execution returns to the caller as soon as the two frames are
queued, which is why code after the call still runs (the second test covers
this).

**Predict the behavior:**

1. A processor calls `broadcast_interruption()` while it has 10 `AudioRawFrame`
   items sitting in its process queue. After the call returns, are those frames
   still there?
2. You call `broadcast_interruption()` and then immediately push a
   `OutputTransportMessageUrgentFrame`. In what order do downstream processors
   receive them?
3. If you forget the `await` on `broadcast_interruption()`, what happens?

---

## Why this matters

Barge-in — the ability for the user to interrupt the bot mid-sentence — is the
single most important real-time UX property of a voice agent. Without it the
bot feels deaf and robotic. `broadcast_interruption` is the one call that makes
barge-in work: it resets the caller's state and notifies the entire pipeline in
a single `await`. When debugging "the bot keeps talking over me," the first
question is always: did `broadcast_interruption` actually get called, and did it
reach the output transport?

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/frame_processor.py`. Find
> `FrameProcessor.broadcast_interruption`. Replace its body with
> `raise NotImplementedError("Stage 06")` but KEEP the signature, docstring,
> and decorators exactly. Don't touch any other code. Then show me the
> signature and the `InterruptionFrame` type so I know the contract.

---

## Your tasks (in order)

- [ ] **Read the source.** Open `src/pipecat/processors/frame_processor.py`
      and read `broadcast_interruption` (line ~735) and `broadcast_frame`
      (line ~756). Then read `InterruptionFrame` in
      `src/pipecat/frames/frames.py` (line ~1018). Note its inheritance chain:
      `InterruptionFrame → SystemFrame → Frame`.

- [ ] **State the contract before writing code.** Answer in a comment or
      scratch pad:
      - What is the return type of `broadcast_interruption`?
      - Which method does the actual two-direction push, and what are its args?
      - What happens to the caller's process queue when the task is reset?
      - Does anything block after the push?

- [ ] **Gut check.** With the body replaced by `raise NotImplementedError`,
      run:
      ```
      uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption -x
      ```
      Confirm you get `NotImplementedError`. This proves the test is actually
      calling your code.

- [ ] **Implement — happy path.** Restore the body: reset the process task,
      stop metrics, broadcast the frame. Run the happy-path test:
      ```
      uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption -x
      ```
      Expected: `InterruptionFrame` appears in both `expected_down_frames` and
      `expected_up_frames`, followed downstream by the urgent frame the
      processor pushed after the broadcast.

- [ ] **Edge case — code after the call still runs.** Run:
      ```
      uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption_allows_subsequent_code -x
      ```
      This test asserts that `code_after_ran` is `True` and that the
      `OutputTransportMessageUrgentFrame` appears downstream. If this fails,
      you have made `broadcast_interruption` block or swallow the subsequent
      push — re-read `broadcast_frame` and confirm it returns after enqueueing,
      not after delivery.

- [ ] **Run both tests together** to confirm neither regresses:
      ```
      uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption_allows_subsequent_code
      ```

- [ ] **Reflect.** In one or two sentences: why does the task reset happen
      *before* the push rather than after? What would break if you reversed
      the order?

---

## Stuck? (paste into Claude)

> I'm reimplementing `broadcast_interruption`. Here's my attempt: [code].
> Don't give me the answer — ask me one question that points at what I'm
> missing.

---

## What you learned

`broadcast_interruption` is a three-step protocol: clear your own queue, stop
your metrics, then broadcast. Understanding this order is the key to debugging
barge-in issues: if the bot keeps talking, either the interruption was never
broadcast (VAD/trigger problem), it was broadcast but the queue wasn't cleared
first (ordering bug), or downstream processors ignored the frame (stage 07
problem). Knowing which step failed tells you exactly where to look.

---

## Next → 07-interruption-handling.md
