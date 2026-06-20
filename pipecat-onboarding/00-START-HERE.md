# Pipecat Internals — A Reconstruction Practicum

## Course Syllabus

### Course description

This practicum teaches the internals of the Pipecat framework through reconstruction.
Rather than studying documentation and hoping it transfers, you will rebuild the
framework one function at a time. Each assignment isolates a single function in the
live codebase, removes its implementation, and asks you to reconstruct it until
Pipecat's own test suite passes. Because only one function is removed per assignment,
the rest of the system remains intact and operational throughout — you are always
completing a working machine, never assembling one from nothing.

On completing all twenty-two assignments you will have personally reconstructed the
frame, the processor, the pipeline, the interruption mechanism, the voice-activity
state machine, the conversation aggregators, the speech and language service
contracts, the input transport, and the runner.

### Prerequisites

- Working knowledge of Python and `asyncio`.
- Basic familiarity with `git` (branching, `restore`, `diff`).
- The repository available locally with `uv` installed. All work occurs on the
  `onboarding` branch; the `main` branch holds the unmodified original for reference.

### Learning objectives

Upon completion, the student will be able to:

1. Trace how a unit of data (a *frame*) flows through a chain of processors, in both
   the downstream and upstream directions.
2. Explain and reconstruct the interruption ("barge-in") mechanism that makes a voice
   agent feel responsive.
3. Describe the voice-activity-detection pipeline that determines when a user is
   speaking.
4. Account for how a conversation turn is assembled from streamed fragments.
5. Implement the service contract shared by all speech and language providers.
6. Articulate the principal design decisions of the framework and the trade-offs
   behind them (see the Learning Outcomes handout).

### Orientation assessment (complete before Assignment 01)

You should be able to answer the following at a high level before beginning. Revisit
them at the end of each assignment, where the same questions recur in greater depth.

**1. How does Pipecat work, in broad terms?**
A bot is a *pipeline*: a left-to-right chain of *processors*, each a small asynchronous
node. Every unit of data that moves through them — a chunk of microphone audio, a text
token, a "user started speaking" signal, an "end the call" command — is a typed
*frame*. Frames travel *downstream* (input → output) and *upstream* (output → input,
for errors and acknowledgments). A representative voice bot is
`transport-in → STT → user-aggregator → LLM → assistant-aggregator → TTS → transport-out`.
When the user interrupts, an `InterruptionFrame` sweeps the chain and cancels in-flight
work so the bot stops speaking immediately.

**2. What language and runtime does it use?**
Python 3.10+, built entirely on `asyncio`. No threads run in the hot path except
voice-activity inference, which is offloaded to a `ThreadPoolExecutor` so it never
blocks the event loop. The test suite uses `pytest` with `pytest-asyncio`.

**3. Where would you go to modify each subsystem?**

| Task | Location |
|------|----------|
| Add a new frame type | `src/pipecat/frames/frames.py` |
| Change interruption behavior | `src/pipecat/processors/frame_processor.py` |
| Add an STT/TTS/LLM provider | subclass under `src/pipecat/services/` |
| Change end-of-turn detection | `src/pipecat/audio/vad/` |
| Change how history is assembled | `src/pipecat/processors/aggregators/` |
| Run a complete bot | `src/pipecat/workers/runner.py` |

### Required reading

Review the following five source files before Assignment 01 (skim for structure; do not
attempt to memorize). Every assignment builds on them.

1. `src/pipecat/frames/frames.py` — the frame types and their taxonomy
   (`SystemFrame` / `DataFrame` / `ControlFrame`, and the `UninterruptibleFrame` mixin).
2. `src/pipecat/processors/frame_processor.py` — the `FrameProcessor` base class: the
   two-task model, `queue_frame`, `push_frame`, `link`, `_start_interruption`.
3. `src/pipecat/pipeline/pipeline.py` — `Pipeline`, `PipelineSource`, `PipelineSink`.
4. `src/pipecat/pipeline/worker.py` — `PipelineWorker`, its lifecycle, and heartbeats.
5. `src/pipecat/processors/aggregators/llm_response_universal.py` — how a conversation
   turn is collected.

### System overview

Every datum is a *frame*. Frames travel through a chain of *frame processors*, each
owning two internal asyncio tasks and two queues: a *priority queue* that allows
`SystemFrame`s (such as interruptions) to advance ahead of ordinary frames, and a
*FIFO frame queue* that protects terminal frames (such as `EndFrame`) from being
discarded during an interruption. A *pipeline* is itself a frame processor that links a
list of processors between a `PipelineSource` and a `PipelineSink`. A *pipeline worker*
wraps the pipeline, injects the opening `StartFrame`, and monitors heartbeats. The
*worker runner* is the outermost loop: it starts the message bus, runs the workers, and
terminates when they finish.

Built upon the frame-processor abstraction: *voice-activity detection*
(analyzer → controller → processor) determines when the user is speaking;
*aggregators* (context, user, assistant) assemble conversation turns; *services*
(STT, TTS, LLM) form the AI edges; *transports* form the audio and network I/O edges.

```
                          WorkerRunner  (Assignment 22)
                               │ runs
                          PipelineWorker (Assignments 10–11)
                               │ wraps
   ┌───────────────────────  Pipeline  (Assignments 8–9) ──────────────────────┐
   │                                                                            │
 transport-in → [VAD] → STT → user-agg → LLM → assistant-agg → TTS → transport-out
 (Asgn 21)    (12–14)  (18)   (16)      (20)    (17)          (19)
   │                                                                            │
   └── each box is a FrameProcessor (Assignments 4–7) moving Frames (Asgn 1)    │
       through its priority queue (Asgn 2) and frame queue (Asgn 3) ────────────┘
                  interruptions sweep the whole chain (Assignments 6–7)
                  conversation state lives in LLMContext (Assignment 15)
```

### Assignment schedule

Complete the assignments **in order**. Each function under reconstruction depends only
on functions from earlier assignments, never later ones.

| # | Assignment | Function under reconstruction | Concept | Prereq. |
|---|------------|-------------------------------|---------|---------|
| 1 | `01-frame-base.md` | `Frame.__post_init__` | frame identity and the three-tier taxonomy | — |
| 2 | `02-priority-queue.md` | `FrameProcessorQueue.put` | system frames advance ahead of data frames | 1 |
| 3 | `03-frame-queue.md` | `FrameQueue.reset` | terminal frames survive interruption | 1 |
| 4 | `04-processor-linking.md` | `FrameProcessor.link` | the doubly-linked chain | 1 |
| 5 | `05-push-routing.md` | `FrameProcessor.__internal_push_frame` | upstream vs. downstream routing | 2, 4 |
| 6 | `06-interruption-broadcast.md` | `FrameProcessor.broadcast_interruption` | triggering an interruption | 5 |
| 7 | `07-interruption-handling.md` | `FrameProcessor._start_interruption` | cancelling in-flight work and flushing | 3, 6 |
| 8 | `08-pipeline-wiring.md` | `Pipeline._link_processors` | source/sink and linking the list | 4 |
| 9 | `09-pipeline-routing.md` | `Pipeline.process_frame` | a pipeline as a composable node | 8 |
| 10 | `10-worker-startup.md` | `PipelineWorker` startup / `run()` | configuration via `StartFrame` | 9 |
| 11 | `11-heartbeats.md` | the heartbeat task handler | detecting a stalled pipeline | 10 |
| 12 | `12-vad-state-machine.md` | `VADAnalyzer._run_analyzer` | the QUIET→SPEAKING→STOPPING machine | 1 |
| 13 | `13-vad-controller.md` | `VADController` event loop | emitting speech events | 12 |
| 14 | `14-vad-processor.md` | `VADProcessor` | injecting speaking frames | 5, 13 |
| 15 | `15-llm-context.md` | `LLMContext` | the conversation-state object | 1 |
| 16 | `16-user-aggregator.md` | `LLMUserAggregator.process_frame` | assembling a user turn | 7, 15 |
| 17 | `17-assistant-aggregator.md` | `LLMAssistantAggregator.process_frame` | assembling an assistant turn | 7, 15 |
| 18 | `18-stt-service.md` | `STTService` (audio → text) | the STT service contract | 5 |
| 19 | `19-tts-service.md` | `TTSService` (text → audio) | the TTS service contract | 5 |
| 20 | `20-llm-function-calls.md` | `LLMService._run_function_call` | tool / function dispatch | 15 |
| 21 | `21-input-transport.md` | `BaseInputTransport.process_frame` | audio ingress | 5 |
| 22 | `22-worker-runner.md` | `WorkerRunner.run` | the outermost loop | 10 |

> **Note.** Assignments 1, 2, 3, 12, 18, and 19 concern infrastructure or abstract
> methods that have **no dedicated unit test**. Those assignments state this explicitly
> and provide a lightweight verification harness in place of a test that does not exist.
> Every other assignment maps each task to a specific `pytest` invocation.

### Assessment

An assignment is complete when (a) every `pytest` checkpoint listed in it passes, and
(b) you can articulate the corresponding outcomes in the Learning Outcomes handout
([`MASTERY.md`](MASTERY.md)). Run the suite as follows:

```bash
cd /Users/hughgramelspacher/repos/pipecat
uv run pytest                                   # the full suite
uv run pytest tests/test_frame_processor.py     # a single file
uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption  # a single test
```

`pyproject.toml` sets `pythonpath = ["src"]`, so imports resolve without installing the
package. The full procedure for testing appears in the Laboratory Reference
([`TESTING.md`](TESTING.md)) and should be read before Assignment 01.

### Course handbook

| Document | Purpose |
|----------|---------|
| [`HOW-TO-DO-A-STAGE.md`](HOW-TO-DO-A-STAGE.md) | **Assignment Procedure** — the standard procedure for completing any assignment, including permitted resources |
| [`TESTING.md`](TESTING.md) | **Laboratory Reference** — running the suite, reading a failing test as a specification, the `run_test` harness, verification without a dedicated test |
| [`MASTERY.md`](MASTERY.md) | **Learning Outcomes & Self-Assessment** — per-assignment outcome statements and the conceptual examination |
| [`PROGRESS.md`](PROGRESS.md) | **Completion Record** — a record of assignments completed |

### Beginning the course

Read the Assignment Procedure ([`HOW-TO-DO-A-STAGE.md`](HOW-TO-DO-A-STAGE.md)) and the
Laboratory Reference ([`TESTING.md`](TESTING.md)), then proceed to **Assignment 01**
(`01-frame-base.md`).
