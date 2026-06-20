# Mastery — what you should be able to say

This is the answer key to "did I actually learn it." Two parts:

1. **Per-stage statements** — at the *start* of a stage, read its block: these are your
   targets. At the *end*, say each one out loud in your own words. If you can't, reread
   the **code** (not the test) before moving on.
2. **Cross-cutting "why" questions** — the design decisions that span the whole repo
   (why asyncio, why a priority queue, why dataclasses, why pytest). These are the ones
   an interviewer — or your own future debugging self — actually asks.

A statement you can't explain is a stage worth redoing (`git restore <file>`).

---

## Per-stage: "By the end I can explain…"

### Foundations — the atom and the queues
- **01 Frame** — why every datum in Pipecat is a `Frame`; what `__post_init__` adds
  (unique id, name, empty metadata) and why a unique id matters when reading logs; the
  three-tier taxonomy (`SystemFrame` / `DataFrame` / `ControlFrame`) and what the
  `UninterruptibleFrame` mixin marks.
- **02 Priority queue** — why a processor needs a *priority* queue, not a plain FIFO;
  what makes a frame high-priority (it's a `SystemFrame`) and why an interruption must
  never wait behind a backlog of audio frames.
- **03 Frame queue** — what `reset()` drains and what it preserves on interruption; why
  `EndFrame`/`StopFrame` must survive a barge-in; what the O(1) `has_uninterruptible`
  flag buys you.

### The node — FrameProcessor
- **04 Linking** — how `_next`/`_prev` form a doubly-linked chain and why both
  directions exist (downstream data, upstream errors/acks).
- **05 Push routing** — how `push_frame(direction)` decides whether to call
  `_next.queue_frame()` or `_prev.queue_frame()`; what happens at the end of the chain.
- **06 Interruption broadcast** — how *any* processor triggers barge-in by broadcasting
  an `InterruptionFrame` both ways in one call; why it resets its own task first.
- **07 Interruption handling** — the receiving side: when to cancel the in-flight task
  vs. only flush the queue (current frame uninterruptible); why this is what makes TTS
  actually stop mid-sentence.

### The chain — Pipeline & Worker
- **08 Pipeline wiring** — how a flat list of processors becomes a linked chain
  bracketed by `PipelineSource`/`PipelineSink`; where upstream frames exit.
- **09 Pipeline routing** — why a `Pipeline` is *itself* a `FrameProcessor`, and how
  that makes pipelines nestable.
- **10 Worker startup** — what the opening `StartFrame` carries (params like sample
  rate) and why processors can't start before they receive it; the Worker-vs-Task
  naming history.
- **11 Heartbeats** — why a pipeline needs a liveness signal; how a missed heartbeat
  surfaces a blocked processor that would otherwise silently kill a call.

### VAD — when is the user speaking
- **12 VAD state machine** — the QUIET→STARTING→SPEAKING→STOPPING hysteresis and why
  the counters exist (so one noisy frame doesn't flip the state); the dual
  confidence+volume gate; why inference runs in a thread pool.
- **13 VAD controller** — how state *transitions* (edges) become discrete
  `on_speech_started`/`on_speech_stopped` events; what the audio-idle timeout protects
  against (a muted mic that just stops sending audio).
- **14 VAD processor** — how controller events become real
  `VADUserStartedSpeakingFrame`/`...Stopped...` frames the rest of the chain reacts to;
  why audio still passes through.

### Conversation — context & aggregation
- **15 LLMContext** — what the context holds (OpenAI-format messages + tools) and why
  it must not mutate the caller's original list; that it's the bot's *memory*.
- **16 User aggregator** — how scattered transcription frames between "user started"
  and "user stopped" become one user message that triggers the LLM; what the stop
  timeout handles.
- **17 Assistant aggregator** — how streamed `LLMTextFrame` tokens between
  Start/End-response frames become one assistant message; why a barge-in must *discard*
  the partial reply (else the context lies about what the bot said).

### The AI & I/O edges
- **18 STT service** — the service pattern: a base `FrameProcessor` + an abstract
  `run_stt` the provider implements; audio-in → `TranscriptionFrame`-out; the mute
  protocol so STT doesn't transcribe the bot's own voice.
- **19 TTS service** — the mirror: text-in → ordered `TTSAudioRawFrame`-out via abstract
  `run_tts`; why chunk ordering matters; what TTFB metrics measure.
- **20 LLM function calls** — how a model's function-call request is dispatched to
  registered Python code and the result fed back; what happens on an unknown function.
- **21 Input transport** — how real-world audio (mic/WebRTC/phone) enters as
  `InputAudioRawFrame`; what `StartFrame` configures; the head of the chain.
- **22 Worker runner** — what `WorkerRunner.run` starts (bus + workers), how signals are
  handled, and why `auto_end=True` ends the bot when its workers finish. This is
  `python my_bot.py`.

---

## Cross-cutting "why" — the decisions behind the whole repo

These span many stages. You should be able to answer each after finishing the course.

**Why `asyncio` and not threads?**
A voice bot is almost all *waiting* — on the network, on the model, on the mic. That's
IO-bound, where one event loop juggling thousands of awaits beats thread-per-task
(no GIL contention, no lock soup, cheap context switches). The one CPU-bound piece —
VAD model inference (stage 12) — is deliberately pushed to a `ThreadPoolExecutor` so it
can't block the loop. That split *is* the architecture.

**Why a priority queue inside every processor (stage 02), not just a FIFO?**
Latency-critical signals (interruptions, start/stop) are `SystemFrame`s and must jump
ahead of a backlog of audio `DataFrame`s. If barge-in waited its turn behind buffered
audio, the bot would talk over the user for hundreds of ms. Priority routing is how
"stop talking" beats "here's more audio."

**Why `@dataclass` for frames but Pydantic `BaseModel` for params/config?**
Frames are created at very high frequency and carry trusted internal data — they want
to be *cheap*, so plain dataclasses (no validation overhead). Params, service configs,
metrics, and external API payloads are low-frequency and cross trust boundaries — they
want *validation and serialization*, so Pydantic. (This rule is in the repo's own
`AGENTS.md`.) Knowing which to reach for is a real contribution skill.

**Why the doubly-linked chain (stages 04–05) instead of one central router?**
Each processor only needs to know its immediate neighbors. Frames flow downstream;
errors and acknowledgments flow upstream. No global dispatcher means processors compose
freely and a `Pipeline` can itself be a node in a bigger pipeline (stage 09).

**Why mark some frames "uninterruptible" (stages 03, 07)?**
Barge-in flushes pending work — but if it flushed `EndFrame`/`StopFrame`, the call would
never cleanly end. The mixin lets the flush be aggressive about audio while protecting
the few terminal frames that must always arrive.

**Why split conversation collection into two aggregators (stages 16–17)?**
The user side and assistant side have opposite shapes: user text arrives as
transcriptions bracketed by speaking events; assistant text streams as tokens bracketed
by response events; and only the assistant side must discard partials on interruption.
Two focused processors beat one branchy mega-processor.

**Why the abstract-method service pattern (stages 18–20)?**
60+ providers (Deepgram, ElevenLabs, OpenAI, …) share one contract: the base
`STTService`/`TTSService`/`LLMService` owns the frame plumbing, metrics, and muting;
each provider only implements `run_stt`/`run_tts`. New integration = one method, not a
new processor. That's why the base classes have no isolated test — they're abstract;
the concrete providers carry the coverage.

**Why pytest + `uv` + the `run_test()` helper?**
pytest is the Python standard — plain `assert`, fixtures, and `::test_name` selection
so each rebuild step has exactly one green light. `uv` is the repo's fast resolver/runner
(`uv run pytest` needs no manual venv; `pythonpath=["src"]` in `pyproject.toml` finds
the source). `run_test()` exists because testing a processor in isolation is
meaningless — it only behaves correctly *inside* a running pipeline, so the helper wires
up a real one and asserts on the frames that come out each direction. (Full detail in
[`TESTING.md`](TESTING.md).)

**Why "Worker" everywhere the docs used to say "Task" (stages 10, 22)?**
Pipecat folded in multi-agent support; "task" now means only an asyncio task, while the
top-level runnable unit is a "Worker." `PipelineTask`/`PipelineRunner` still exist as
deprecated aliases. Recognizing this saves you confusion when test names say `test_task_*`
but the classes say `Worker`.

---

→ Back to [`00-START-HERE.md`](00-START-HERE.md) · ritual in [`HOW-TO-DO-A-STAGE.md`](HOW-TO-DO-A-STAGE.md)
