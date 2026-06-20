# Stage 14 — VADProcessor frame injection

## Where you are

```
transport-in (InputAudioRawFrame)
        │
        ▼
┌────────────────────────────────────────────────┐
│              VADProcessor  ◄── YOU ARE HERE    │
│                                                │
│  ┌─────────────────────┐                       │
│  │   VADController     │                       │
│  │  (stage 13)         │                       │
│  │                     │                       │
│  │  on_speech_started  │──► broadcast_frame    │
│  │  on_speech_stopped  │──► broadcast_frame    │
│  │  on_speech_activity │──► broadcast_frame    │
│  └─────────────────────┘        │              │
│                                 │              │
└────────────────────────────────────────────────┘
                                  │
            ┌─────────────────────┴──────────────────────┐
            │                     │                      │
            ▼                     ▼                      ▼
  VADUserStartedSpeakingFrame  UserSpeakingFrame  VADUserStoppedSpeakingFrame
            │                     │                      │
            ▼                     ▼                      ▼
     [aggregators]         [turn strategies]      [aggregators]
     [interruption]        [LLMWorker]            [interruption]
```

`VADProcessor` sits just after the transport input. It owns a `VADController` (stage 13) and
wires that controller's events to `broadcast_frame` calls so every downstream processor sees
the VAD state transitions as real frames.

---

## How it works (orientation — answer these before you code)

**Q: What is `VADProcessor`'s single job?**
A: It is the adapter between the VAD subsystem (controller + analyzer, stages 12-13) and the
Pipecat frame pipeline. The controller knows about audio states; `VADProcessor` converts state
transitions into typed frames that the rest of the pipeline understands. It is not responsible
for the detection logic — that lives in `VADController`. Its job is bridging.

**Q: How does `__init__` wire the controller's events to frame broadcasting?**
A: After constructing a `VADController`, `__init__` registers four `event_handler` closures on
it using the `@self._vad_controller.event_handler("...")` decorator pattern (same `BaseObject`
event system used across the framework). The relevant three are:

- `on_speech_started` → `await self.broadcast_frame(VADUserStartedSpeakingFrame, start_secs=...)`
- `on_speech_stopped` → `await self.broadcast_frame(VADUserStoppedSpeakingFrame, stop_secs=...)`
- `on_speech_activity` → `await self.broadcast_frame(UserSpeakingFrame)`

A fourth handler (`on_push_frame` / `on_broadcast_frame`) delegates any controller-initiated
push/broadcast calls back through the processor so they reach the pipeline correctly.

**Q: What does `process_frame` do with an `InputAudioRawFrame`?**
A: It does two things in sequence:
1. Calls `await self.push_frame(frame, direction)` — forwards the raw audio downstream
   immediately, before any VAD processing happens.
2. Calls `await self._vad_controller.process_frame(frame)` — lets the controller analyze the
   audio and, if a state transition occurs, fire the wired event handlers (which then
   broadcast the VAD frames).

This ordering matters: downstream processors receive the audio frame first, then the
VAD-triggered broadcast frames arrive. Tests confirm `InputAudioRawFrame` appears before
`VADUserStartedSpeakingFrame` in the output sequence.

**Q: Where does `VADProcessor` belong in a pipeline?**
A: Directly after the input transport, before any aggregators or turn strategy processors. It
needs to see every raw audio frame arriving from the transport. Everything that cares about
"is the user speaking?" (LLM aggregator, interruption handler, turn strategy) must be
downstream of it.

**Predict-the-behavior questions:**

1. If you send two `InputAudioRawFrame` objects where the analyzer returns `[QUIET, SPEAKING]`,
   how many frames appear in the downstream output, and in what order?

2. If a `StartFrame` arrives before any audio, what extra side effect does `process_frame`
   trigger (hint: look at the non-audio branch in `process_frame`)?

3. `broadcast_frame` (stage 05) sends to all branches of the pipeline. `push_frame` only
   sends to the next processor in the chain. Why is `broadcast_frame` the right choice for
   VAD state frames rather than `push_frame`?

---

## Why this matters

`VADProcessor` is the seam where "the audio analyzer detected speech" becomes a typed frame
that every other processor in the pipeline can react to, including aggregators that accumulate
user utterances and interruption handlers that cut off bot speech. Without this bridge, the
VAD subsystem is isolated — it can detect speech but nothing downstream would know. When you
are debugging a situation where "VAD fires but the LLM never receives a user turn," the first
place to look is whether `VADProcessor` is present in the pipeline and whether its broadcast
events are actually reaching downstream processors, which you can verify by checking the frame
sequence the tests assert here.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/audio/vad_processor.py`. Find `VADProcessor.__init__` (the
> event-handler closures that call `broadcast_frame`) and `VADProcessor.process_frame`. Replace
> the bodies of the three speech-event handlers (`on_speech_started`, `on_speech_stopped`,
> `on_speech_activity`) and the body of `process_frame` (everything after
> `await super().process_frame(frame, direction)`) with `raise NotImplementedError("Stage 14")`.
> Keep all signatures, decorators, docstrings, and the `VADController` construction exactly as
> they are. Then show me: (1) the three event-handler signatures, (2) the `process_frame`
> signature, (3) the `VADController` parameter it wraps, and (4) the three frame types it is
> supposed to broadcast so I know the contract before I implement anything.

---

## Your tasks (in order)

- [ ] **Read and trace the contract.** Read `vad_processor.py` and `vad_controller.py`
  (stage 13). Identify: which controller events fire, what arguments they receive, and what
  frame types must be broadcast for each. Write down the mapping before touching any code.

- [ ] **Confirm the frame types.** Find `VADUserStartedSpeakingFrame`,
  `VADUserStoppedSpeakingFrame`, and `UserSpeakingFrame` in
  `src/pipecat/frames/frames.py`. Note any constructor arguments (e.g. `start_secs`,
  `stop_secs`) that `broadcast_frame` must receive as keyword arguments.

- [ ] **Implement `on_speech_started`.** Wire the `on_speech_started` event to
  `broadcast_frame(VADUserStartedSpeakingFrame, start_secs=...)`. Pull `start_secs` from
  `_controller._vad_analyzer.params.start_secs`. Run:
  ```
  uv run pytest tests/test_vad_processor.py::TestVADProcessor::test_pushes_started_speaking_frame
  ```

- [ ] **Implement `on_speech_stopped`.** Wire `on_speech_stopped` to
  `broadcast_frame(VADUserStoppedSpeakingFrame, stop_secs=...)`. Run:
  ```
  uv run pytest tests/test_vad_processor.py::TestVADProcessor::test_pushes_stopped_speaking_frame
  ```

- [ ] **Implement `on_speech_activity`.** Wire `on_speech_activity` to
  `broadcast_frame(UserSpeakingFrame)` (no extra kwargs). Run:
  ```
  uv run pytest tests/test_vad_processor.py::TestVADProcessor::test_pushes_user_speaking_frame
  ```

- [ ] **Edge case — audio passthrough.** Implement `process_frame` so that
  `InputAudioRawFrame` is pushed downstream (via `push_frame`) before the controller
  processes it. Confirm the ordering assertion in:
  ```
  uv run pytest tests/test_vad_processor.py::TestVADProcessor::test_forwards_audio_frames
  ```
  The expected downstream sequence starts with `SpeechControlParamsFrame` then
  `InputAudioRawFrame` — audio must flow through even when no VAD event fires.

- [ ] **Run the full suite.** Verify all seven tests pass:
  ```
  uv run pytest tests/test_vad_processor.py -v
  ```

- [ ] **Reflect.** You have now built the complete VAD subsystem across stages 12-14:
  analyzer (raw detection) → controller (state machine + events) → processor (frame
  injection). Write one sentence describing what would break in a real voice pipeline if
  `process_frame` called `self._vad_controller.process_frame(frame)` but forgot to call
  `self.push_frame(frame, direction)` first.

---

## Stuck? (paste into Claude)

> I'm reimplementing `VADProcessor`. Here's my attempt: [code]. Don't give me the answer —
> ask me one question that points at what I'm missing.

---

## What you learned

`VADProcessor` shows a clean pattern for bridging an event-emitting subsystem into a
frame-based pipeline: construct the subsystem, wire its events to `broadcast_frame` calls
in `__init__`, and in `process_frame` forward the raw frame downstream first and then hand
it off to the subsystem for analysis. The ordering — push audio, then run VAD — is what
makes the downstream frame sequence deterministic and testable. When debugging a real voice
pipeline where VAD fires but nothing downstream reacts, start by inspecting the frame
sequence at the output of `VADProcessor`: if `VADUserStartedSpeakingFrame` is missing or
arrives out of order, the broadcast wiring in `__init__` is the culprit; if audio frames
are missing entirely, check the `push_frame` call in `process_frame`.

---

## Next → 15-llm-context.md
