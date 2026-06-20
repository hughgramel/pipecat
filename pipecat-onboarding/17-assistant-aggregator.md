# Stage 17 — LLMAssistantAggregator — collecting the LLM's reply

## Where you are

```
   LLM service
       |
       | LLMFullResponseStartFrame  ← turn opens; assistant turn started event fires
       v
  ┌─────────────────────────────────┐
  │      LLMAssistantAggregator     │  ◄── YOU ARE HERE
  │                                 │
  │  LLMTextFrame("Hello ")         │
  │  LLMTextFrame("from ")          │  ← tokens accumulate in _aggregation[]
  │  LLMTextFrame("Pipecat!")       │
  │       |                         │
  │  LLMFullResponseEndFrame        │  ← turn closes; push_aggregation() fires
  │       |                         │
  │  context.add_message(           │
  │    {"role":"assistant",         │  ← ONE message written to LLMContext
  │     "content":"Hello from ..."})|
  └─────────────────────────────────┘
       |
       | LLMContextFrame (downstream)
       v
      TTS
       |
       | (or, on barge-in:)
       |
  InterruptionFrame → _handle_interruptions()
       │  _trigger_assistant_turn_stopped(interrupted=True)
       │      → push_aggregation() commits whatever was collected, if any
       │  reset() clears in-memory buffer for next turn
       └──► partial text enters context tagged interrupted=True
```

`LLMAssistantAggregator` is the mirror of `LLMUserAggregator` (stage 16). Where the user half collects speech tokens into a user message, this half collects streamed LLM output tokens into an assistant message. Both sides write into the same shared `LLMContext`, so the full conversation history stays coherent.

---

## How it works (orientation — answer these before you code)

**Q: What does LLMAssistantAggregator accumulate, and when does it commit?**

A: It accumulates `LLMTextFrame` tokens (and other `TextFrame` subtypes with `append_to_context=True`) into a list called `_aggregation`. The list is committed — turned into a single `{"role": "assistant", "content": "..."}` message in `LLMContext` — when `LLMFullResponseEndFrame` arrives. The commit happens inside `push_aggregation()`, which is called by `_trigger_assistant_turn_stopped()`, which is called by `_handle_llm_end()`.

**Q: Walk me through the Start/End bracketing in `process_frame`.**

A:
1. `LLMFullResponseStartFrame` → `_handle_llm_start()` → records a start timestamp and fires the `on_assistant_turn_started` event. In realtime mode it also flushes the paired user aggregator.
2. `LLMTextFrame` (and other `TextFrame` subtypes) → `_handle_text()` → appends a `TextPartForConcatenation` to `self._aggregation`.
3. `LLMFullResponseEndFrame` → `_handle_llm_end()` → calls `_trigger_assistant_turn_stopped()` → calls `push_aggregation()` → concatenates `_aggregation`, writes the assistant message to context, pushes an `LLMContextFrame` downstream, pushes an `LLMContextAssistantTimestampFrame`, and clears `_aggregation`.

**Q: What happens on interruption — does the partial reply poison the context?**

A: No, but it *is* committed. `_handle_interruptions()` calls `_trigger_assistant_turn_stopped(interrupted=True)`, which in turn calls `push_aggregation()`. If tokens had accumulated, they are written to context as a complete assistant message — but the `on_assistant_turn_stopped` event fires with `message.interrupted = True`, so downstream code knows the turn was cut short. After `push_aggregation()` returns, `reset()` clears the now-empty buffer. The design choice is deliberate: the bot really did say those words before the user interrupted, so the context should reflect that.

**Q: Where would you change how partial replies are handled?**

A: In `_handle_interruptions()` (around line 1609). If you wanted to *discard* partials instead of committing them, you would call `await self.reset()` *before* calling `_trigger_assistant_turn_stopped(interrupted=True)` — that would cause `push_aggregation()` to find `_aggregation` empty and return early. Alternatively, you could check `message.interrupted` inside an `on_assistant_turn_stopped` handler and remove the last message from context after the fact.

**Predict the behavior — answer before running:**

1. You send `[LLMFullResponseStartFrame, LLMTextFrame("Hi"), LLMTextFrame(" there"), LLMFullResponseEndFrame]`. How many messages end up in `context.messages`? What is the content?
2. You send `[LLMFullResponseStartFrame, LLMTextFrame("Partial"), SleepFrame, InterruptionFrame]`. After the interruption frame processes, is `_aggregation` empty or does it still contain "Partial"? Is there a message in `context.messages`?
3. You send `LLMRunFrame` upstream. What frame does the aggregator push back upstream, and why?

---

## Why this matters

If a barge-in left a half-finished assistant message unrecorded, the LLM's next call would have a gap in the conversation history — it would have no memory of starting a reply before being cut off. Conversely, if the partial tokens stayed in `_aggregation` and leaked into the *next* turn's message, the assistant's reply would be prefixed with stale text, producing gibberish that is very hard to debug because it looks like an LLM hallucination. Understanding that interruptions commit the partial (flagged `interrupted=True`) is the key to diagnosing "the bot thinks it said something it didn't" bugs — the answer is almost always that the interrupted turn wrote correctly but a downstream handler didn't check the flag.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/aggregators/llm_response_universal.py`. Find `LLMAssistantAggregator.process_frame` (around line 1439). Replace its entire body with `raise NotImplementedError("Stage 17")` but keep the method signature (`async def process_frame(self, frame: Frame, direction: FrameDirection):`), its docstring, and any decorators exactly as they are. Do not touch any other method. Then show me:
> 1. The full method signature.
> 2. The three frame types that bracket and populate the assistant turn (`LLMFullResponseStartFrame`, `LLMFullResponseEndFrame`, `LLMTextFrame`).
> 3. The call chain from `LLMFullResponseEndFrame` to the line that writes to `self._context`.
> 4. What `_handle_interruptions` does with `_aggregation` after an `InterruptionFrame`.

---

## Your tasks (in order)

- [ ] **Read the source.** Read `LLMAssistantAggregator.process_frame` in full (lines 1439–1527). Then read `push_aggregation` (1563–1580), `_handle_llm_start` / `_handle_llm_end` (1848–1879), `_handle_text` (1894–1915), `_handle_interruptions` (1609–1611), and `_trigger_assistant_turn_stopped` (2014–2036). Trace the complete path from `LLMFullResponseStartFrame` to the assistant message appearing in `context.messages`.

- [ ] **State the contract before writing a line of code.** In your own words: (a) what three frame types define an assistant turn's lifecycle, (b) where tokens are stored between start and end, (c) what happens in `push_aggregation()` when `_aggregation` is empty versus non-empty, and (d) exactly what is committed on an interruption and how the caller knows the turn was interrupted.

- [ ] **Gut check.** Answer the three predict-the-behavior questions in the orientation section above. Write your answers down before running any tests.

  > **Note — expected failure mode (HANG, not crash).** Gutting
  > `LLMAssistantAggregator.process_frame` does not produce a clean `FAILED`
  > line. The covering tests **hang indefinitely** because Pipecat's
  > `TaskManager` swallows exceptions raised inside background asyncio tasks:
  > the `NotImplementedError` is caught internally, and the outer `run()` loop
  > waits forever for a terminal frame that never arrives. The "red light" for
  > this stage is a test that does not return — interrupt with `Ctrl-C` or set
  > a pytest timeout. A hanging test runner confirms the function is correctly
  > gutted; an ordinary `FAILED` assertion would not.

- [ ] **Implement: accumulate tokens and commit on End.** Restore `process_frame`
  so that `LLMFullResponseStartFrame` calls `_handle_llm_start`, `LLMTextFrame`
  (and general `TextFrame`) calls `_handle_text`, and `LLMFullResponseEndFrame`
  calls `_handle_llm_end`. Keep the `super().process_frame` call at the top and
  the pass-through `else` branch at the bottom. Run the happy-path test:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMAssistantAggregator::test_simple -v
  ```
  This test sends `LLMFullResponseStartFrame → LLMTextFrame("Hello from ") →
  LLMTextFrame("Pipecat!") → LLMFullResponseEndFrame` and asserts one assistant
  message with content `"Hello from Pipecat!"` appears in context.

  > **Note on `tests/test_llm_response.py`.** That file contains a
  > `TestLLMFullResponseAggregator` class that tests an older helper class
  > (`LLMFullResponseAggregator` from `pipecat.processors.aggregators.llm_response`),
  > not `LLMAssistantAggregator`. Those tests do not exercise the method being
  > reconstructed in this stage and will not turn red when `process_frame` is
  > gutted. Use the `TestLLMAssistantAggregator` tests in
  > `tests/test_context_aggregators_universal.py` for all verification here.

- [ ] **Full context-frame run test.** Verify the aggregator pushes an
  `LLMContextFrame` upstream when `LLMRunFrame` arrives:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMAssistantAggregator::test_llm_run -v
  ```

- [ ] **Messages update and transform.** Confirm the aggregator correctly replaces
  and transforms messages in the shared context without triggering an LLM run
  unless `run_llm=True`:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMAssistantAggregator::test_llm_messages_update tests/test_context_aggregators_universal.py::TestLLMAssistantAggregator::test_llm_messages_transform -v
  ```

- [ ] **Edge case: interruption commits the partial turn, flagged correctly.**
  Send `LLMFullResponseStartFrame → LLMTextFrame("Hello ") → InterruptionFrame →
  LLMFullResponseStartFrame → LLMTextFrame("Hello ") → LLMTextFrame("there!") →
  LLMFullResponseEndFrame`. Confirm `stop_messages[0].interrupted` is `True`,
  `stop_messages[0].content == "Hello "`, and `stop_messages[1].interrupted` is
  `False`:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMAssistantAggregator::test_interruption -v
  ```

- [ ] **Reflect.** Why does the design commit the partial on interruption rather
  than discard it? What would break in a real voice conversation if you discarded
  it instead? What downstream code would need to change if you wanted to suppress
  partial replies from ever entering context?

---

## Stuck? (paste into Claude)

> I'm reimplementing `LLMAssistantAggregator.process_frame`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`LLMAssistantAggregator` gives you a single, reliable assistant message in `LLMContext` for every turn the bot completes or is interrupted during. The key invariant is that `_aggregation` is always empty when a new `LLMFullResponseStartFrame` arrives — either because `push_aggregation()` cleared it on a normal end, or because `reset()` cleared it after an interruption's commit. When debugging context corruption after a barge-in ("the bot thinks it said something longer than it actually said"), the first place to look is whether `InterruptionFrame` reached `process_frame` before the next `LLMFullResponseStartFrame`, and whether `on_assistant_turn_stopped` was handled correctly with `message.interrupted` checked. Context corruption after barge-in is almost always a missing or misordered `InterruptionFrame`, not a bug in the aggregator itself.

---

## Next → 18-stt-service.md
