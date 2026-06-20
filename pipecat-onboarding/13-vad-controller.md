# Stage 13 — VADController event dispatch

## Where you are

```
mic audio
    │
    ▼
┌─────────────────────┐
│    VADAnalyzer      │  ← stage 12: returns VADState per chunk
│  (QUIET/SPEAKING/…) │
└──────────┬──────────┘
           │ VADState
           ▼
┌─────────────────────┐
│   VADController     │  ← YOU ARE HERE
│                     │
│  QUIET→SPEAKING  ──►│── on_speech_started  ◄── fires once on leading edge
│  SPEAKING→QUIET  ──►│── on_speech_stopped  ◄── fires once on trailing edge
│  every SPEAKING  ──►│── on_speech_activity ◄── fires every audio chunk
│  audio goes idle ──►│── on_speech_stopped  ◄── safety: mic-mute detection
└──────────┬──────────┘
           │ events
           ▼
┌─────────────────────┐
│   VADProcessor      │  ← stage 14: turns events into pipeline Frames
└─────────────────────┘
```

## How it works (orientation — answer these before you code)

**Q: What does VADController add on top of VADAnalyzer?**

A: `VADAnalyzer` (stage 12) returns a raw `VADState` for every audio chunk. It knows nothing about history. `VADController` wraps the analyzer and owns *state memory*: it remembers the previous `VADState` so it can detect *transitions* — the edges where the state changes from one stable value to another — and fire discrete async events on those edges. It also tracks wall-clock time to detect when audio stops arriving entirely.

**Q: How does the controller detect a transition and decide which event to fire?**

A: In `_handle_vad`, the controller calls `self._vad_analyzer.analyze_audio(audio)` to get a `new_vad_state`, then compares it to the saved `vad_state`. A transition is only acted on when:

1. `new_vad_state != vad_state` (something changed), AND
2. `new_vad_state` is not a *transient* state — `STARTING` and `STOPPING` are filtered out; only `SPEAKING` and `QUIET` are stable edges.

When those conditions hold:
- `new_vad_state == SPEAKING` → fire `on_speech_started`
- `new_vad_state == QUIET`    → fire `on_speech_stopped`

Then `vad_state` is updated to `new_vad_state` so the next chunk has the right baseline.

**Q: What is `on_speech_activity` for, and when does it fire?**

A: It fires on *every* audio chunk that arrives while `_vad_state == SPEAKING` — after the transition logic has already run. It is a continuous heartbeat: callers can use it to measure speaking energy, update UI, or drive barge-in confidence. Contrast with `on_speech_started`, which fires only once per speech segment.

**Q: How does the audio-idle timeout work?**

A: `setup()` spawns `_audio_idle_handler()` as a long-running asyncio task. That loop records `_last_audio_time` and sleeps until `_last_audio_time + _audio_idle_timeout`. If the deadline has passed *and* `_vad_state == SPEAKING`, it logs a warning, forces `_vad_state` to `QUIET`, and fires `on_speech_stopped`. This handles the case where the user mutes their microphone mid-sentence: the analyzer never gets new audio to analyze, so the normal state-transition path would never fire the stop event.

Set `audio_idle_timeout=0` at construction time to disable this safety entirely.

**Q: Where would you hook in your own callback or change idle behavior?**

A: Register event handlers with the decorator pattern shown in the class docstring:

```python
@vad_controller.event_handler("on_speech_started")
async def handle_started(controller):
    ...
```

To change idle behavior: pass a different `audio_idle_timeout` (seconds) to the constructor. To inspect the current VAD state from outside the controller, read `controller._vad_state` — though you should treat it as internal and prefer reacting to the events instead.

**Predict the behavior:**

1. The analyzer returns `STARTING → STARTING → SPEAKING`. How many times does `on_speech_started` fire?
   > *Once* — `STARTING` is filtered; only the `QUIET→SPEAKING` edge (when `SPEAKING` finally differs from the saved `QUIET`) fires the event.

2. The user speaks for three audio chunks then stops. How many `on_speech_activity` events fire?
   > Three — one per chunk while `_vad_state == SPEAKING`.

3. Audio stops arriving entirely while the user is mid-sentence. Which code path fires `on_speech_stopped`?
   > `_audio_idle_handler()`, not `_handle_vad()`. The idle task detects the elapsed wall-clock time and forces the state to `QUIET`.

## Why this matters

`VADController` is the bridge between raw audio analysis and the high-level events that drive conversation turn management. `on_speech_started` is the signal that typically triggers barge-in (interrupting the bot), and `on_speech_stopped` is the signal that triggers end-of-turn detection and kicks off STT. When debugging "the bot never realizes I stopped talking," the first place to look is whether `on_speech_stopped` is firing at all — if it is not, the issue is here in the controller (analyzer not reaching `QUIET`, or the idle timeout not being set up), rather than in the processors downstream.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/audio/vad/vad_controller.py`. Find the `_handle_vad` method — it calls `analyze_audio` and fires `on_speech_started` / `on_speech_stopped`. Replace its body with `raise NotImplementedError("Stage 13")` but keep the signature, docstring, and type annotations exactly as they are. Then also replace the body of `_handle_audio` with `raise NotImplementedError("Stage 13")` and keep its signature and docstring. Do not touch `_audio_idle_handler`, `_start`, `push_frame`, `broadcast_frame`, `setup`, or `cleanup`. Show me:
> 1. The signatures of `_handle_vad` and `_handle_audio`.
> 2. The five event names registered in `__init__`.
> 3. The `_audio_idle_timeout` field and how `setup()` uses it.

## Your tasks (in order)

- [ ] **Read and trace the real implementation.** Before you stub anything, read `_handle_vad` and `_handle_audio` in full. Trace through: what happens when the analyzer returns `STARTING`? When it returns `SPEAKING` for the second time in a row? Confirm your answers match the "Predict the behavior" questions above.

- [ ] **Write the contract on paper.** Draw the state machine: the two stable states (`QUIET`, `SPEAKING`) and the two transient ones (`STARTING`, `STOPPING`). Write down, in pseudocode, the three things `_handle_vad` must do: (1) call the analyzer, (2) check for a stable edge, (3) update saved state. Predict which test each rule is meant to satisfy.

- [ ] **Gut check.** Look at `_handle_audio`: it sets `_last_audio_time`, calls `_handle_vad`, then checks `_vad_state`. Notice the order matters — `_handle_vad` updates `_vad_state` in-place via return value before the activity check. Make sure your implementation returns the new state rather than mutating it directly inside `_handle_vad`.

- [ ] **Implement the QUIET→SPEAKING edge and run the first test.**
  Fire `on_speech_started` when `new_vad_state == SPEAKING` and it differs from the current stable state (filtering out `STARTING`/`STOPPING`). Update `vad_state` and return it.
  ```
  uv run pytest tests/test_vad_controller.py::TestVADController::test_speech_started_event
  ```

- [ ] **Add the SPEAKING→QUIET edge and run the second test.**
  Fire `on_speech_stopped` when `new_vad_state == QUIET` and it differs from the current stable state.
  ```
  uv run pytest tests/test_vad_controller.py::TestVADController::test_speech_stopped_event
  ```

- [ ] **Implement the continuous activity event and run the third test.**
  In `_handle_audio`, after `_handle_vad` returns the new state, check if `_vad_state == VADState.SPEAKING` and call `on_speech_activity` unconditionally (the test sends two frames and expects exactly two events, so no throttle logic is needed here — `_maybe_speech_activity` exists but is not called by `_handle_audio`).
  ```
  uv run pytest tests/test_vad_controller.py::TestVADController::test_speech_activity_event
  ```

- [ ] **Verify the audio-idle forced stop (edge case: mic mute).** This test lives in `TestVADControllerAudioIdle` and requires `setup()` to be called (which starts the idle task). Read `_audio_idle_handler` to understand how it uses `_last_audio_time` and `_audio_idle_timeout`. You are not stubbing that method — just confirm your `_handle_audio` keeps `_last_audio_time` updated so the idle task has accurate data.
  ```
  uv run pytest tests/test_vad_controller.py::TestVADControllerAudioIdle::test_audio_idle_forces_speech_stop
  ```

- [ ] **Verify the StartFrame VAD-params broadcast (edge case: params propagation).** `_start()` calls `broadcast_frame(SpeechControlParamsFrame, vad_params=...)`. Confirm `process_frame` routes `StartFrame` to `_start`. This test ensures downstream services (e.g., STT) receive the initial VAD configuration.
  ```
  uv run pytest tests/test_vad_controller.py::TestVADController::test_start_frame_broadcasts_vad_params
  ```

- [ ] **Reflect.** Run the full test file and confirm everything passes:
  ```
  uv run pytest tests/test_vad_controller.py -v
  ```
  Then answer: If a user speaks very fast and the analyzer never leaves `STARTING` before jumping to `QUIET`, which event fires? Which does not? How would you reproduce this in a unit test?

## Stuck? (paste into Claude)

> I'm reimplementing the `VADController` event loop — specifically `_handle_vad` and `_handle_audio`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`VADController` converts the continuous stream of `VADState` values from the analyzer into three discrete signals: `on_speech_started` (leading edge), `on_speech_stopped` (trailing edge), and `on_speech_activity` (heartbeat while speaking). These are the events that every turn-management and barge-in mechanism in Pipecat ultimately listens to. When debugging "the bot never realizes I stopped talking," check whether `on_speech_stopped` fires at all: if it doesn't, the VAD model may never be reaching `QUIET` (a sensitivity/threshold problem in the analyzer, stage 12), or audio stopped arriving but the idle timeout was disabled or never set up (a wiring problem here in the controller). The idle timeout also teaches an important lesson about asyncio: not all events come from data — some must come from the *absence* of data, which requires a separate monitoring task rather than inline logic.

## Next → 14-vad-processor.md
