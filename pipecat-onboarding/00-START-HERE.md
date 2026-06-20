# Pipecat Onboarding — Start Here

Welcome. This is a **rebuild-it-to-learn-it** course for the Pipecat codebase. You
won't read 22 design docs and hope it sticks. Instead, each stage hands you a prompt
that **guts one real function** in this repo, then a task list to reimplement it from
scratch — verified by Pipecat's **own existing test suite**. You only ever break the
one thing you're working on; everything else stays green.

By the end you'll have personally rewritten the frame, the processor, the pipeline,
the interruption mechanism, the VAD state machine, the conversation aggregators, the
AI service contracts, the transport, and the runner. You will understand this system
the way you understand code you wrote.

---

## First, the orientation (answer these out loud before stage 01)

These are the "do I get the shape of it" questions. You should be able to answer all
of them at a high level after reading the files below. They come back, harder, at the
end of each stage.

**High level — how does Pipecat work?**
> A bot is a **pipeline**: a left-to-right chain of **processors**, each a small
> async node. Everything that moves through them — a chunk of mic audio, a text
> token, a "user started speaking" signal, an "end the call" command — is a typed
> **Frame**. Frames flow *downstream* (input → output) and *upstream* (output →
> input, for errors and acknowledgments). A typical voice bot is:
> `transport-in → STT → user-aggregator → LLM → assistant-aggregator → TTS → transport-out`.
> When the user barges in, an **InterruptionFrame** sweeps the chain and cancels
> in-flight work so the bot stops talking instantly.

**What language / runtime is it?**
> Python 3.10+, fully `asyncio`. No threads in the hot path except VAD inference
> (run in a `ThreadPoolExecutor` so it never blocks the event loop). Tests are
> `pytest` + `pytest-asyncio`.

**Where would I…**
> - …add a new frame type? → `src/pipecat/frames/frames.py`
> - …change how interruptions work? → `src/pipecat/processors/frame_processor.py`
> - …add a new STT/TTS/LLM provider? → subclass in `src/pipecat/services/`
> - …change when the bot decides the user stopped talking? → `src/pipecat/audio/vad/`
> - …change how conversation history is assembled? → `src/pipecat/processors/aggregators/`
> - …run the whole thing? → `src/pipecat/workers/runner.py`

---

## The five files to read first

Read these (skim, don't memorize) before stage 01. Everything else builds on them.

1. `src/pipecat/frames/frames.py` — every Frame type and the taxonomy
   (`SystemFrame` / `DataFrame` / `ControlFrame`, the `UninterruptibleFrame` mixin).
2. `src/pipecat/processors/frame_processor.py` — the `FrameProcessor` base: the
   two-task model, `queue_frame`, `push_frame`, `link`, `_start_interruption`.
3. `src/pipecat/pipeline/pipeline.py` — `Pipeline`, `PipelineSource`, `PipelineSink`.
4. `src/pipecat/pipeline/worker.py` — `PipelineWorker`, lifecycle, heartbeats.
5. `src/pipecat/processors/aggregators/llm_response_universal.py` — how a turn of
   conversation is collected.

---

## The architecture in one breath

Every datum is a **Frame**. Frames travel through a chain of **FrameProcessor**
nodes, each owning two internal asyncio tasks and two queues — a **priority queue**
that lets `SystemFrame`s (like interruptions) jump ahead, and a **FIFO frame queue**
that protects terminal frames (`EndFrame`) from being flushed on interruption. A
**Pipeline** is itself a FrameProcessor that `link()`s a list of processors between a
`PipelineSource` and `PipelineSink`. A **PipelineWorker** wraps the pipeline, injects
the opening `StartFrame`, and monitors heartbeats. **WorkerRunner** is the outermost
loop that starts the bus, runs the workers, and ends when they finish.

Hanging off the FrameProcessor abstraction: **VAD** (analyzer → controller →
processor) decides when the user is speaking; **aggregators** (context, user,
assistant) assemble conversation turns; **services** (STT, TTS, LLM) are the AI edges;
**transports** are the audio/network I/O edges.

```
                          WorkerRunner  (stage 22)
                               │ runs
                          PipelineWorker (stages 10–11)
                               │ wraps
   ┌───────────────────────  Pipeline  (stages 8–9) ───────────────────────┐
   │                                                                        │
 transport-in → [VAD] → STT → user-agg → LLM → assistant-agg → TTS → transport-out
 (stage 21)   (12-14)  (18)   (16)      (20)    (17)          (19)
   │                                                                        │
   └── each box is a FrameProcessor (stages 4–7) moving Frames (stage 1)    │
       through its priority queue (stage 2) + frame queue (stage 3) ────────┘
                  interruptions sweep the whole chain (stages 6–7)
                  conversation state lives in LLMContext (stage 15)
```

---

## The 22 stages

Work them **in order** — each stage's function depends only on functions from
earlier stages, never later. Open stage 01 and go.

| # | Page | Guts | What it teaches | Depends on |
|---|------|------|-----------------|------------|
| 1 | `01-frame-base.md` | `Frame.__post_init__` | frame IDs, names, the 3-tier taxonomy | — |
| 2 | `02-priority-queue.md` | `FrameProcessorQueue.put` | SystemFrames jump the line | 1 |
| 3 | `03-frame-queue.md` | `FrameQueue.reset` | terminal frames survive interruption | 1 |
| 4 | `04-processor-linking.md` | `FrameProcessor.link` | the doubly-linked chain | 1 |
| 5 | `05-push-routing.md` | `FrameProcessor.__internal_push_frame` | upstream vs downstream routing | 2,4 |
| 6 | `06-interruption-broadcast.md` | `FrameProcessor.broadcast_interruption` | how barge-in is triggered | 5 |
| 7 | `07-interruption-handling.md` | `FrameProcessor._start_interruption` | cancel current work + flush | 3,6 |
| 8 | `08-pipeline-wiring.md` | `Pipeline._link_processors` | source/sink + linking the list | 4 |
| 9 | `09-pipeline-routing.md` | `Pipeline.process_frame` | Pipeline as a passthrough node | 8 |
| 10 | `10-worker-startup.md` | `PipelineWorker` startup/`run()` | StartFrame from params | 9 |
| 11 | `11-heartbeats.md` | heartbeat task handler | stuck-pipeline detection | 10 |
| 12 | `12-vad-state-machine.md` | `VADAnalyzer._run_analyzer` | QUIET→SPEAKING→STOPPING | 1 |
| 13 | `13-vad-controller.md` | `VADController` event loop | firing speech events | 12 |
| 14 | `14-vad-processor.md` | `VADProcessor` | injecting speaking-frames | 5,13 |
| 15 | `15-llm-context.md` | `LLMContext` | conversation state object | 1 |
| 16 | `16-user-aggregator.md` | `LLMUserAggregator.process_frame` | collecting a user turn | 7,15 |
| 17 | `17-assistant-aggregator.md` | `LLMAssistantAggregator.process_frame` | collecting LLM tokens | 7,15 |
| 18 | `18-stt-service.md` | `STTService` (audio→text) | the STT service contract | 5 |
| 19 | `19-tts-service.md` | `TTSService` (text→audio) | the TTS service contract | 5 |
| 20 | `20-llm-function-calls.md` | `LLMService._run_function_call` | tool/function dispatch | 15 |
| 21 | `21-input-transport.md` | `BaseInputTransport.process_frame` | audio ingress | 5 |
| 22 | `22-worker-runner.md` | `WorkerRunner.run` | the outermost loop | 10 |

> **Stages 1, 2, 3, 18, 19** cover infrastructure / abstract methods that have **no
> isolated unit test**. Those pages tell you so and give you a thin-mock checkpoint
> instead of pointing at a test that doesn't exist. Every other stage maps each task
> to a real `pytest` invocation.

---

## How to run the tests

```bash
cd /Users/hughgramelspacher/repos/pipecat
uv run pytest                                   # everything
uv run pytest tests/test_frame_processor.py     # one file
uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption  # one test
```

`pyproject.toml` sets `pythonpath = ["src"]`, so imports just work. There's no
top-level `conftest.py`; the test helper `run_test(...)` in
`src/pipecat/tests/utils.py` wires processors into a real pipeline and reports the
frames it saw — read it once, you'll see it everywhere.

---

## How each stage works

1. **Read** the real implementation and trace it (always task 1).
2. **State the contract** — inputs, outputs, invariants — before writing code.
3. **Paste the setup prompt** into Claude to gut the one function.
4. **Reimplement** it in small steps, each lighting up a specific existing test.
5. **Hit the edge cases** (interruption, empty input, cancellation) as named tasks.
6. **Reflect** — tie it back to a real "why isn't this working" debugging skill.

Stuck on any stage? Each page has a "Stuck?" prompt that asks Claude to question you
toward the answer instead of handing it over.

Open **`01-frame-base.md`** and begin.
