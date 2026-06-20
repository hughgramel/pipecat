# Stage 12 — VAD confidence state machine

## Where you are

```
  raw audio bytes
        |
        v
  VADAnalyzer._run_analyzer          <-- YOU ARE HERE
        |
  (per-frame confidence float)
        |
        v
  ┌─────────────────────────────────────────────────────────┐
  │                    State machine                        │
  │                                                         │
  │   confidence >= threshold      _vad_starting_count      │
  │      AND volume >= min_volume  >= _vad_start_frames     │
  │              ┌──────────────────────────────────┐       │
  │              │                                  │       │
  │  ┌───────────v──┐   (each high-confidence  ┌───v──────┐ │
  │  │    QUIET     │   frame increments count) │ STARTING │ │
  │  │              <──────────────────────────┤          │ │
  │  └──────────────┘   (any low frame resets) └──────────┘ │
  │                                                         │
  │  ┌──────────────┐   _vad_stopping_count    ┌──────────┐ │
  │  │   STOPPING   │   >= _vad_stop_frames    │ SPEAKING │ │
  │  │              ├────────────────────────> │          │ │
  │  └──────┬───────┘                          └──────────┘ │
  │         │  (each low-confidence frame                    │
  │         │   increments count; high frame                │
  │         │   reverts to SPEAKING)                        │
  └─────────────────────────────────────────────────────────┘
        |
        v
  VADController (stage 13) — fires on_speech_started / on_speech_stopped
  only on SPEAKING / QUIET transitions (not on STARTING or STOPPING)
        |
        v
  VADProcessor (stage 14) — pushes UserStartedSpeakingFrame / UserStoppedSpeakingFrame
```

`_run_analyzer` is the pure synchronous state logic. It is called from `analyze_audio`, which offloads it to a `ThreadPoolExecutor` so the CPU-bound model inference never blocks the asyncio event loop.

---

## How it works (orientation — answer these before you code)

**Q: What does `_run_analyzer` do at the highest level?**

A: It accumulates raw audio bytes into an internal buffer, drains it in fixed-size chunks (`_vad_frames_num_bytes`), and for each chunk calls the abstract `voice_confidence()` (implemented by the concrete provider — e.g. Silero) plus `_get_smoothed_volume()`. It then advances a four-state machine and returns the current `VADState`.

**Q: What are the four states and how do they move?**

| From | Condition | To | Side effect |
|---|---|---|---|
| QUIET | `speaking` is True | STARTING | `_vad_starting_count = 1` |
| STARTING | `speaking` is True | STARTING | `_vad_starting_count += 1` |
| STARTING | `speaking` is False | QUIET | `_vad_starting_count = 0` |
| STARTING (post-loop) | count >= `_vad_start_frames` | SPEAKING | count reset |
| SPEAKING | `speaking` is False | STOPPING | `_vad_stopping_count = 1` |
| STOPPING | `speaking` is True | SPEAKING | `_vad_stopping_count = 0` |
| STOPPING | `speaking` is False | STOPPING | `_vad_stopping_count += 1` |
| STOPPING (post-loop) | count >= `_vad_stop_frames` | QUIET | count reset |

The `speaking` boolean is `confidence >= params.confidence AND volume >= params.min_volume`. Both thresholds must hold simultaneously.

**Q: Runtime detail — why a thread pool?**

`analyze_audio` (the async entry point) calls `loop.run_in_executor(self._executor, self._run_analyzer, buffer)`. The executor holds a single worker thread per analyzer. Running the ONNX/Silero model is CPU-bound; keeping it off the event loop prevents audio processing from blocking downstream frame delivery.

**Q: Where would you tune sensitivity?**

| Parameter | Default | Effect |
|---|---|---|
| `params.confidence` | 0.7 | Minimum per-frame confidence score to count as speech |
| `params.min_volume` | 0.6 | Volume gate — high confidence but quiet audio is ignored |
| `params.start_secs` | 0.2 s | How long confidence must stay high before SPEAKING is declared |
| `params.stop_secs` | 0.2 s | How long silence must persist before reverting to QUIET |

`set_params()` converts `start_secs` / `stop_secs` into frame counts (`_vad_start_frames`, `_vad_stop_frames`) based on `num_frames_required()` and `sample_rate`.

**Predict-the-behavior questions (answer before you code):**

1. A single loud frame arrives with confidence 0.95. What state does the machine return? Why doesn't it jump straight to SPEAKING?
2. The user is SPEAKING and then goes quiet for one frame, then loud again. What state does the machine return after the loud frame?
3. Volume is 0.9 and confidence is 0.95 but `min_volume` is set to 1.0. What does `speaking` evaluate to, and why does this protect against distant keyboard noise?

---

## Why this matters

VAD tuning directly controls the user experience of barge-in: `start_secs` and `confidence` together determine how quickly the bot treats speech as real, while `stop_secs` and `min_volume` control how long a pause must last before the bot considers the user done. A too-aggressive `start_secs` causes the bot to interrupt itself on every breath; a too-conservative `stop_secs` makes it seem unresponsive. When you debug "the bot cuts me off" or "the bot never stops listening", the state machine here is the first place to instrument.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/audio/vad/vad_analyzer.py`. Find `VADAnalyzer._run_analyzer`. Replace its body with `raise NotImplementedError("Stage 12")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the `VADState` enum values, and the following fields so I know the contract: `_vad_buffer`, `_vad_frames_num_bytes`, `_vad_starting_count`, `_vad_stopping_count`, `_vad_start_frames`, `_vad_stop_frames`, `_vad_state`, `_params.confidence`, `_params.min_volume`.

---

## Your tasks (in order)

- [ ] **Read and trace the state machine.** Read `_run_analyzer` in full. Draw the four-state diagram on paper (or in a comment). Identify the two post-loop threshold checks and explain why they happen after the while loop rather than inside it.

- [ ] **State the contract.** Before writing any code, write a one-paragraph comment above your implementation: input is a `bytes` audio buffer; output is a `VADState`; the machine is stateful (mutates `self._vad_state` and the two counters); re-entrant calls accumulate into `_vad_buffer`; the return value is always the state after processing all complete chunks.

- [ ] **Gut check: predict behavior.** Answer the three predict-the-behavior questions from the orientation section out loud (or in a comment). Do this before writing a single line of implementation.

- [ ] **Implement QUIET → STARTING → SPEAKING on rising confidence.** Wire up the `speaking` boolean, the QUIET-to-STARTING transition, the STARTING count increment, and the post-loop check that promotes STARTING → SPEAKING when `_vad_starting_count >= _vad_start_frames`. Run:
  ```
  uv run pytest tests/test_silero_vad.py::TestSileroVAD::test_voice_confidence_conversion_matches_expected
  ```
  This test confirms the `voice_confidence()` path (int16→float32 conversion) that feeds your confidence variable. It should pass as long as you call `self.voice_confidence(audio_frames)` correctly and do not mutate the buffer before passing it.

- [ ] **Silence stays QUIET.** Add the low-confidence branches (STARTING → QUIET reset, SPEAKING → STOPPING, STOPPING count increment, STOPPING → QUIET post-loop check). Run:
  ```
  uv run pytest tests/test_silero_vad.py::TestSileroVAD::test_voice_confidence_silence
  ```
  Silence audio produces a confidence well below 0.5, so a fresh analyzer fed silence must return QUIET.

- [ ] **Edge case: transitional states produce no event (named test).** STARTING and STOPPING are internal hysteresis states; the VADController (stage 13) only fires events on SPEAKING and QUIET. Confirm your implementation passes the named test:
  ```
  uv run pytest tests/test_vad_controller.py::TestVADController::test_no_event_on_transitional_states
  ```
  If this fails, check that your STARTING and STOPPING transitions do not prematurely advance to SPEAKING or QUIET within the same frame.

- [ ] **Edge case: min_volume gate (named).** Add a targeted unit test (or a manual trace) for the case where `confidence >= params.confidence` but `volume < params.min_volume`. The `speaking` boolean must be `False`, so the machine must not advance from QUIET toward STARTING. Write a comment naming this case: "high-confidence low-volume gate".

- [ ] **Reflect.** Write 2-3 sentences in a comment or your notes: what would you change about the default `start_secs` / `stop_secs` / `confidence` values for a noisy call-center environment vs. a quiet studio? What metric would you add to `_run_analyzer` to make tuning easier?

---

## Stuck? (paste into Claude)

> I'm reimplementing `VADAnalyzer._run_analyzer`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`_run_analyzer` turns a raw per-frame confidence float into a debounced state by requiring `_vad_start_frames` consecutive high-confidence, high-volume frames before declaring SPEAKING, and `_vad_stop_frames` consecutive silent frames before reverting to QUIET. This hysteresis is the primary tuning surface for "bot interrupts on pause" vs. "bot is slow to respond": shortening `start_secs` makes the bot more reactive but more prone to false barge-ins on breath sounds; raising `min_volume` prevents distant noise from counting as speech even when the model is briefly confident. When debugging interruption behavior, instrument `_vad_state`, `_vad_starting_count`, and `_vad_stopping_count` on every frame — the pattern of counts will immediately reveal whether the problem is the confidence threshold, the volume gate, or the frame-count window.

---

## Next → 13-vad-controller.md
