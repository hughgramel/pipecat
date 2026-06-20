# Stage 16 — LLMUserAggregator — collecting a user turn

## Where you are

```
STT output                         LLMUserAggregator                    LLM
──────────                    ┌──────────────────────────┐         ──────────
                              │                          │
VADUserStartedSpeakingFrame ──►  turn starts            │
                              │  _aggregation = []       │
TranscriptionFrame("Hello") ──►  append "Hello"         │
TranscriptionFrame(", how") ──►  append ", how"         │
TranscriptionFrame(" are")  ──►  append " are"          │
TranscriptionFrame(" you?") ──►  append " you?"         │
                              │                          │
VADUserStoppedSpeakingFrame ──►  turn stop strategy     │
                              │  fires after timeout     │
                              │                          │
                              │  push_aggregation():     │
                              │  "Hello, how are you?"   │
                              │  → context.add_message() │
                              │  → LLMContextFrame ──────►  LLM runs
                              │                          │
                              └──────────────────────────┘
                                   [user-agg] ◄── highlighted
```

`InterimTranscriptionFrame` and `TranslationFrame` are consumed silently (not
pushed downstream). Only finalized `TranscriptionFrame` chunks accumulate.

## How it works (orientation — answer these before you code)

**Q: What does `LLMUserAggregator` turn a stream of transcription frames into?**

A: It collects every finalized `TranscriptionFrame` that arrives while a user
turn is active into `self._aggregation` (a list of `TextPartForConcatenation`
objects). At end-of-turn it joins them into a single string, writes that string
to `LLMContext` as a `{"role": "user", "content": "..."}` message, then pushes
an `LLMContextFrame` downstream — which is the signal the LLM service watches
to start generating a response.

**Q: What brackets a user turn, and when does `LLMContextFrame` fire?**

A: Turn lifecycle is delegated to a `UserTurnController` that runs the
configured `UserTurnStrategies`:

- A **start strategy** (e.g., `VADUserTurnStartStrategy`) fires when it sees
  `VADUserStartedSpeakingFrame`. The controller emits `on_user_turn_started`,
  which broadcasts `UserStartedSpeakingFrame` (and optionally an
  `InterruptionFrame`).
- While the turn is open, every `TranscriptionFrame` is appended to
  `_aggregation` by `_handle_transcription`.
- A **stop strategy** (e.g., `SpeechTimeoutUserTurnStopStrategy`) fires some
  time after `VADUserStoppedSpeakingFrame`. The controller emits
  `on_user_turn_inference_triggered` → `push_aggregation()` → `LLMContextFrame`
  fires. Then `on_user_turn_stopped` fires and `UserStoppedSpeakingFrame` is
  broadcast.

So `LLMContextFrame` is pushed **inside `push_aggregation()`**, which is called
when the stop strategy signals that inference should start.

**Q: What is the stop-timeout, and when does it fire?**

A: `user_turn_stop_timeout` (default 5 s, set via `LLMUserAggregatorParams`)
is a safety net. If no stop strategy fires within that window after the turn
starts, the controller gives up and fires `on_user_turn_stop_timeout`. The
event handler calls `on_user_turn_stop_timeout` on the aggregator, and the
turn is still closed via `on_user_turn_stopped`. This means the bot replies
even if the STT never delivered a stop signal.

**Q: Where would you change end-of-turn behavior?**

A: Pass a `UserTurnStrategies` object in `LLMUserAggregatorParams`:

```python
params = LLMUserAggregatorParams(
    user_turn_strategies=UserTurnStrategies(
        stop=[SpeechTimeoutUserTurnStopStrategy(user_speech_timeout=0.8)]
    )
)
```

Increase `user_speech_timeout` to wait longer before cutting off; decrease it
for faster (but potentially premature) replies. You can also swap in
`FilterIncompleteUserTurnStrategies` to make the LLM itself confirm whether the
user finished speaking before committing.

**Predict-the-behavior questions — answer before you read the tests:**

1. If `VADUserStartedSpeakingFrame` arrives but no `TranscriptionFrame` ever
   comes, and then `VADUserStoppedSpeakingFrame` arrives, does an
   `LLMContextFrame` go downstream? Why or why not?

2. `LLMRunFrame` bypasses the whole turn machinery. What does `process_frame`
   do when it receives one?

3. `LLMMessagesAppendFrame` has a `run_llm` flag. What does the aggregator emit
   downstream when `run_llm=False` vs `run_llm=True`?

## Why this matters

`LLMUserAggregator` is the gate between speech and inference: it decides *when*
the bot starts thinking. Tune `user_speech_timeout` too short and the bot
interrupts the user mid-sentence; leave `user_turn_stop_timeout` too long and
the bot freezes when STT stalls. When debugging "bot replies before I finish"
look at the stop strategy timeout; when debugging "bot waits forever" look at
whether a `VADUserStoppedSpeakingFrame` is actually reaching the aggregator.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/aggregators/llm_response_universal.py`. Find
> `LLMUserAggregator.process_frame`. Replace its body with
> `raise NotImplementedError("Stage 16")` but KEEP the signature, docstring,
> and decorators exactly. Don't touch any other code. Then show me the
> signature, the frame types it handles, and the LLMContext/LLMContextFrame so
> I know the contract.

## Your tasks (in order)

- [ ] **Read and trace** — Read `LLMUserAggregator.process_frame` (line 765)
  and `push_aggregation` (line 838) in full. Identify: which frame types are
  *consumed* (not pushed downstream), which delegate to helpers, and which fall
  through to `push_frame`.

- [ ] **Write the contract** — Before writing any code, write a short (5–8
  line) comment block at the top of `process_frame` describing: frames in →
  what accumulates → what fires at turn end. Include the frame type that
  triggers the LLM. Get this reviewed by Claude before proceeding.

- [ ] **Gut check** — Answer the three predict-the-behavior questions in the
  orientation section above. Check your answers against the test at line 324
  (`test_user_turn_stop_timeout_no_transcription`) and the test at line 91
  (`test_llm_run`).

  > **Note — expected failure mode (HANG, not crash).** Gutting
  > `LLMUserAggregator.process_frame` does not produce a clean `FAILED` line.
  > The covering tests **hang indefinitely** because Pipecat's `TaskManager`
  > swallows exceptions raised inside background asyncio tasks: the
  > `NotImplementedError` is caught internally, and the pipeline's internal
  > `run()` loop waits forever for a terminal frame that never arrives. The
  > "red light" for this stage is a test that does not return — interrupt with
  > `Ctrl-C` or set a pytest timeout. A hanging test runner confirms the
  > function is correctly gutted; an ordinary `FAILED` assertion would not.

- [ ] **Implement transcription accumulation + message append**, then run:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_llm_messages_append
  ```
  This confirms that `LLMMessagesAppendFrame` writes to context without
  emitting `LLMContextFrame` when `run_llm=False`.

- [ ] **Implement the full turn flow** (VAD started → transcription → VAD
  stopped → LLMContextFrame), then run:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_llm_run
  ```
  This confirms that `LLMRunFrame` bypasses the turn machinery and immediately
  emits `LLMContextFrame`.

- [ ] **Verify default turn strategies**, then run:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_default_user_turn_strategies
  ```
  This sends `VADUserStartedSpeakingFrame` → `TranscriptionFrame("Hello!")` →
  `VADUserStoppedSpeakingFrame` and checks that `UserStartedSpeakingFrame`,
  `InterruptionFrame`, `LLMContextFrame`, and `UserStoppedSpeakingFrame` all
  arrive downstream in order, and that `stop_message.content == "Hello!"`.

- [ ] **Edge case: stop with no transcription (timeout)**, then run:
  ```
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_user_turn_stop_timeout_no_transcription
  ```
  This sends `VADUserStartedSpeakingFrame` → `VADUserStoppedSpeakingFrame`
  (no `TranscriptionFrame` in between) and waits for `user_turn_stop_timeout`.
  Confirms `on_user_turn_started`, `on_user_turn_stopped`, and
  `on_user_turn_stop_timeout` all fire even when the user said nothing.

- [ ] **Reflect** — After all four tests pass: what would break first if you
  removed the `user_turn_stop_timeout` safety net entirely? Write one sentence
  in a comment at the top of your implementation.

## Stuck? (paste into Claude)

> I'm reimplementing `LLMUserAggregator.process_frame`. Here's my attempt:
> [code]. Don't give me the answer — ask me one question that points at what
> I'm missing.

## What you learned

`LLMUserAggregator` is the gatekeeper for LLM inference: scattered
`TranscriptionFrame` chunks are buffered between VAD-derived start/stop signals,
then flushed as one user message the moment the stop strategy fires. To tune
end-of-turn latency change `user_speech_timeout` on the stop strategy; to
debug "bot replies too early" add a longer timeout or switch to
`FilterIncompleteUserTurnStrategies`; to debug "bot never replies" check
whether `VADUserStoppedSpeakingFrame` is reaching the aggregator, and whether
`user_turn_stop_timeout` is large enough to be your fallback.

## Next → 17-assistant-aggregator.md
