# Stage 21 — BaseInputTransport — audio ingress

## Where you are

```
  Real world
  ──────────
  mic / WebRTC / phone
        │
        │  InputAudioRawFrame
        ▼
┌─────────────────────┐   StartFrame sets _sample_rate
│  BaseInputTransport │◄──────────────────────────────── StartFrame
│  (YOU ARE HERE)     │
│                     │◄── InputTransportStartAudioStreamingFrame
│  _audio_in_queue    │        (control: start streaming)
└────────┬────────────┘
         │  InputAudioRawFrame (via _audio_in_queue)
         ▼
      [VAD]  ← stage 14
         │
         ▼
      [STT]  ← stage 18
         │
         ▼
      [LLM / TTS / ...]
```

`BaseInputTransport` is the head of the pipeline: the first FrameProcessor to
receive real-world audio. Nothing downstream sees a single audio byte until this
class has accepted it and routed it into `_audio_in_queue`.

---

## How it works (orientation — answer these before you code)

**Q: What is an input transport's job at the highest level?**
A: It bridges the external world (a mic, a WebRTC peer, a phone call) to the
Pipecat frame pipeline. It receives raw audio samples from the transport layer,
wraps them as `InputAudioRawFrame`, and feeds them into the processing chain.
It also handles the lifecycle frames that initialise and tear down the pipeline.

**Q: What does `process_frame` do when it receives a `StartFrame`?**
A: It pushes the `StartFrame` downstream first (so every downstream processor
gets to initialise), then calls `self.start(frame)`. Inside `start()`, the
critical side-effect is:

```python
self._sample_rate = self._params.audio_in_sample_rate or frame.audio_in_sample_rate
```

The transport-level override wins; the frame's value is the fallback. After
`_sample_rate` is set, any audio filter attached to the transport is also
started with that rate. This must happen before any `InputAudioRawFrame` flows,
because downstream processors (VAD, STT) use the sample rate to interpret the
raw bytes correctly.

**Q: What happens when an `InputAudioRawFrame` arrives at `process_frame`?**
A: It is routed to `self.push_audio_frame(frame)`, which puts the frame onto
`_audio_in_queue` (if `audio_in_enabled` and not paused). A background asyncio
task (`_audio_task_handler`) drains that queue, optionally filters the audio,
and pushes it downstream — ultimately reaching VAD (stage 14). Critically, the
frame is **not** forwarded with `push_frame`; it takes the queue path instead.

**Q: What does `InputTransportStartAudioStreamingFrame` do?**
A: It is a control frame that tells the transport to begin capturing audio from
the source. When `process_frame` receives it, it calls
`self._start_audio_in_streaming()`. Concrete subclasses (Daily, LiveKit, local
mic) override this hook to actually open the audio stream. The base
implementation is a no-op — the subclass provides the I/O.

**Q: Where do concrete transports plug in?**
A: `BaseInputTransport` is abstract in the behavioural sense. The actual I/O
lives in subclasses under `src/pipecat/transports/`:
- `src/pipecat/transports/local/` — microphone via sounddevice
- `src/pipecat/transports/network/` — WebSocket-based transports
- Third-party packages (pipecat-ai/pipecat-daily, pipecat-ai/pipecat-livekit)
  contain Daily and LiveKit transports.

To add a new transport: subclass `BaseInputTransport`, override
`_start_audio_in_streaming()` to open your I/O source, and call
`await self.push_audio_frame(InputAudioRawFrame(...))` from your reading loop.
You inherit all queue management, VAD routing, filtering, and lifecycle
handling for free.

**Predict the behaviour — answer before you look at the code:**

1. An `InputAudioRawFrame` arrives while `self._paused = True`. What does
   `push_audio_frame` do? (Trace the guard condition in that method.)

2. `StartFrame` carries `audio_in_sample_rate=16000`, but `TransportParams` was
   constructed with `audio_in_sample_rate=8000`. After `start()` runs, what is
   `self._sample_rate`?

3. An `InputTransportStartAudioStreamingFrame` arrives at `process_frame`. Is
   it forwarded downstream with `push_frame`? Why or why not?

---

## Why this matters

Nothing in the pipeline happens until audio enters as frames — the bot cannot
hear the user, VAD never fires, and STT never transcribes. When you debug "no
audio reaching the bot", `BaseInputTransport` is the first place to check:
is the audio task running, is `audio_in_enabled` true, is the transport paused?
A wrong `_sample_rate` (set by `StartFrame`) is equally silent but more
insidious: audio reaches VAD but sounds like noise at the wrong speed, garbling
every transcription without any visible error.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/transports/base_input.py`. Find `BaseInputTransport.process_frame`.
> Replace its body with `raise NotImplementedError("Stage 21")` but KEEP the
> signature, docstring, and decorators exactly. Don't touch any other code.
> Then show me:
> - The full method signature (including `self`, `frame`, `direction`).
> - The three frame types it branches on (`StartFrame`, `InputAudioRawFrame`,
>   `InputTransportStartAudioStreamingFrame`) and what it does for each.
> - The fields `_sample_rate`, `_audio_task`, and `_audio_in_queue` so I
>   understand the state the method reads and writes.

---

## Your tasks (in order)

- [ ] **Read the source.** Open `src/pipecat/transports/base_input.py` and read
  the entire file. Trace `push_audio_frame` → `_audio_in_queue` → `_audio_task_handler`
  so you understand the full path an `InputAudioRawFrame` travels after
  `process_frame` hands it off.

- [ ] **State the contract.** Before writing a line of implementation, write
  down (in comments or a scratch pad) the mapping:
  - `StartFrame` → push downstream first, then call `start(frame)` to configure `_sample_rate`.
  - `InputAudioRawFrame` → call `push_audio_frame(frame)`, do NOT call `push_frame`.
  - `InputTransportStartAudioStreamingFrame` → call `_start_audio_in_streaming()`, do NOT push.
  - All other `SystemFrame` subtypes → `push_frame(frame, direction)`.
  - `EndFrame`, `CancelFrame`, `StopFrame` → see their dedicated branches in the original.

- [ ] **Gut check your model.** Answer the three predict-the-behaviour questions
  above before running any tests. Write your expected answers in a comment.

- [ ] **Implement the `InputAudioRawFrame` branch** and verify the happy path:
  ```
  uv run pytest tests/test_base_input_transport.py::TestBaseInputTransportFrameAudio::test_incoming_audio_frame_routed_to_push_audio_frame
  ```
  This test mocks `push_audio_frame` and asserts it was called with the frame
  (not `push_frame`). Green here means your routing logic is correct.

- [ ] **Implement the `InputTransportStartAudioStreamingFrame` branch** and verify:
  ```
  uv run pytest tests/test_base_input_transport.py::TestBaseInputTransportFrameAudio::test_start_audio_streaming_frame_triggers_streaming
  ```
  This test mocks `_start_audio_in_streaming` and asserts it was called once.

- [ ] **Name the sample-rate edge case.** Add the `StartFrame` branch. Then
  write a small inline test (or a mental trace) confirming: if `TransportParams`
  sets `audio_in_sample_rate=8000` and `StartFrame` carries `audio_in_sample_rate=16000`,
  `self._sample_rate` is `8000` after `start()`. The params value wins.

- [ ] **Run the full test file** to make sure you haven't broken the deprecation
  test for the old `start_audio_in_streaming` method:
  ```
  uv run pytest tests/test_base_input_transport.py
  ```

- [ ] **Reflect.** In one or two sentences: what would happen if you
  accidentally called `push_frame(frame)` for an `InputAudioRawFrame` instead
  of `push_audio_frame(frame)`? What would break downstream, and why?

---

## Stuck? (paste into Claude)

> I'm reimplementing `BaseInputTransport.process_frame`. Here's my attempt:
> [code]. Don't give me the answer — ask me one question that points at what
> I'm missing.

---

## What you learned

`BaseInputTransport.process_frame` is the gatekeeper between the physical world
and the frame pipeline: it configures the sample rate on `StartFrame`, routes
raw audio into the queue-backed VAD path (not straight to `push_frame`), and
hands control-frame `InputTransportStartAudioStreamingFrame` to the subclass
hook that opens the actual I/O source. When you debug transport issues —
"the bot never hears audio", "wrong sample rate garbles STT", "streaming never
starts" — this method and the three fields it touches (`_sample_rate`,
`_audio_task`, `_paused`) are the first place to look.

---

## Next → 22-worker-runner.md
