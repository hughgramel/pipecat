# Stage 10 — PipelineWorker startup

## Where you are

```
WorkerRunner
└── PipelineWorker.run()         ◄── YOU ARE HERE
      │
      ├── _setup()               sets up clock, task manager, processors
      │
      ├── _create_tasks()        spawns _process_push_queue coroutine
      │
      └── _process_push_queue()
            │
            ├── start clock
            ├── _maybe_start_idle_task()
            │
            ├── *** build StartFrame from PipelineParams ***
            │         │
            │         ▼
            │   [PipelineSource] ──► [YourProcessor1] ──► ... ──► [PipelineSink]
            │                                                            │
            │   heartbeats registered ◄── on_pipeline_started ◄─────────┘
            │   idle task registered                                (StartFrame exits here)
            │
            └── main loop: drain _push_queue until EndFrame/CancelFrame/StopFrame
```

The `StartFrame` is the first frame injected into the pipeline. Every processor
receives it before any user data, and uses it to configure itself (sample rate,
metrics switches, tracing). Heartbeat tasks are started **after** `StartFrame`
exits the sink, not before — so processors are always configured before
monitoring begins.

> Naming note: older Pipecat called this class `PipelineTask`. It was renamed to
> `PipelineWorker` in 1.3.0. The tests in `tests/test_pipeline.py` keep the
> `test_task_*` prefix from the old name. `PipelineTask` still exists as a
> deprecated alias.

---

## How it works (orientation — answer these before you code)

**Q: What is `PipelineWorker` at a high level?**

`PipelineWorker` is the glue between a `WorkerRunner` and a user-defined
`Pipeline`. It owns the "outer" control loop: it injects the first `StartFrame`,
routes frames queued by user code (via `queue_frame`) into the pipeline, and
waits for a terminal frame (`EndFrame`, `StopFrame`, or `CancelFrame`) to reach
the sink before tearing down. It wraps the user pipeline with a `PipelineSource`
and `PipelineSink` so it can intercept every frame at both ends.

**Q: What does `run()` actually kick off?**

`run()` calls `_setup()` (wires up the clock, `TaskManager`, and all processor
`setup()` hooks), then `_create_tasks()` (spawns `_process_push_queue` as an
asyncio task), then blocks in `_wait_for_pipeline_finished()`. The real work is
in `_process_push_queue`.

**Q: How is `StartFrame` built and queued?**

Inside `_process_push_queue` (around line 1077 in `worker.py`):

```python
start_frame = StartFrame(
    audio_in_sample_rate=self._params.audio_in_sample_rate,
    audio_out_sample_rate=self._params.audio_out_sample_rate,
    enable_metrics=self._params.enable_metrics,
    enable_tracing=self._enable_tracing,
    enable_usage_metrics=self._params.enable_usage_metrics,
    report_only_initial_ttfb=self._params.report_only_initial_ttfb,
    tracing_context=self._tracing_context,
)
start_frame.metadata = self._create_start_metadata()
await self._pipeline.queue_frame(start_frame)
```

It pulls every field directly from `self._params` (a `PipelineParams` instance
set in `__init__`). The frame is injected directly into `self._pipeline` —
bypassing the user-facing `queue_frame` / `_push_queue` so nothing can jump
ahead of it.

**Q: What gets registered / started after `StartFrame` is queued?**

`_process_push_queue` immediately waits for the `StartFrame` to traverse the
entire pipeline (`_wait_for_pipeline_start`). When the `StartFrame` exits the
`PipelineSink`, `_sink_push_frame` fires three things:

1. The `on_pipeline_started` event handler.
2. `self._observer.on_pipeline_started()` — notifies all observers.
3. `self._maybe_start_heartbeat_tasks()` — conditionally starts the heartbeat
   push loop and the heartbeat monitor loop (only if
   `params.enable_heartbeats=True`).

The **idle monitor** task (`_maybe_start_idle_task`) is started slightly
earlier, right at the top of `_process_push_queue`, before the `StartFrame` is
built. This means the idle timer starts the moment the worker begins running,
not after `StartFrame` has been processed.

**Q: Where do I set audio sample rate or enable metrics?**

Pass a `PipelineParams` instance to the `PipelineWorker` constructor:

```python
worker = PipelineWorker(
    pipeline,
    params=PipelineParams(
        audio_in_sample_rate=16000,
        audio_out_sample_rate=24000,
        enable_metrics=True,
        enable_heartbeats=True,
        heartbeats_period_secs=0.5,
    ),
)
```

`PipelineParams` fields (all with defaults):

| Field | Default | Carried in StartFrame? |
|---|---|---|
| `audio_in_sample_rate` | 16000 | yes |
| `audio_out_sample_rate` | 24000 | yes |
| `enable_metrics` | False | yes |
| `enable_usage_metrics` | False | yes |
| `report_only_initial_ttfb` | False | yes |
| `enable_heartbeats` | False | no (controls task spawning) |
| `heartbeats_period_secs` | 1.0 | no |
| `heartbeats_monitor_secs` | 10.0 | no |
| `send_initial_empty_metrics` | True | no |
| `start_metadata` | {} | merged into `StartFrame.metadata` |

**Predict-the-behavior questions (answer before reading on):**

1. You construct a `PipelineWorker` with default params, then call `queue_frame(TextFrame(...))` before calling `run()`. Does the `TextFrame` get lost, or does it eventually make it into the pipeline? Why?

2. You register an `on_pipeline_started` handler and an `on_pipeline_finished` handler. You only queue an `EndFrame` before calling `run()`. Which handler fires first, and what frame type does `on_pipeline_finished` receive?

3. You set `params=PipelineParams(enable_heartbeats=True)`. At exactly what moment do heartbeats begin — when `run()` is called, when `StartFrame` is queued, or when `StartFrame` exits the sink?

---

## Why this matters

`StartFrame` is the single configuration broadcast that reaches every processor
before any user data flows. If it carries the wrong sample rate, every STT and
TTS service will be misconfigured from the start — and there is no mechanism to
re-send it mid-call. Similarly, metrics and tracing are gated on `StartFrame`
flags; if you forget `enable_metrics=True` in `PipelineParams`, no metrics will
ever appear no matter what you set on individual services. When a bot seems to
be ignoring your params, trace back to the `PipelineParams` passed to
`PipelineWorker.__init__` and verify the `StartFrame` your pipeline actually
receives.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/pipeline/worker.py`. Find `_process_push_queue` — this is
> the method that builds and sends the `StartFrame`. Replace only its body
> (from the `self._clock.start()` line through `await self._cleanup(...)`) with
> `raise NotImplementedError("Stage 10")`. Keep the method signature, the
> docstring, and the `async def` line exactly as they are. Do not touch
> `__init__`, `run`, `_setup`, `_create_tasks`, `_sink_push_frame`, or any
> other method.
>
> Then show me:
> 1. The `_process_push_queue` signature and its docstring.
> 2. The fields of `PipelineParams` with their defaults.
> 3. The fields of `StartFrame.__init__` that map to `PipelineParams`.
>
> I want to see the contract clearly before I implement anything.

---

## Your tasks (in order)

- [ ] **Read and trace.** In `worker.py`, read `__init__`, `run`, `_setup`,
  `_create_tasks`, and `_process_push_queue` top to bottom. In
  `src/pipecat/frames/frames.py`, find `StartFrame` and list every field it
  accepts. Answer the three predict-the-behavior questions above in your own
  words before touching any code.

- [ ] **State the contract.** Without writing code, write two or three sentences
  that describe the exact pre- and post-conditions of `_process_push_queue`:
  what must be true when it starts, what it must do before the main loop, and
  what signals it uses to sequence the startup (the `asyncio.Event` that gates
  the main loop). Include the heartbeat and idle task lifecycle.

- [ ] **Gut check.** Before implementing, predict: if `_process_push_queue`
  starts the heartbeat tasks *before* awaiting `_wait_for_pipeline_start`, what
  could go wrong? Write your answer, then check the real code to see how it
  avoids this.

- [ ] **Implement: StartFrame construction and pipeline bootstrap.** Rebuild the
  top of `_process_push_queue`: start the clock, start the idle task, build
  `StartFrame` from `self._params`, set its metadata, queue it into
  `self._pipeline`, and wait for the pipeline start event. Then run:

  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_task_single
  ```

  The test constructs a bare `PipelineWorker`, queues two `TextFrame`s and an
  `EndFrame`, then calls `run()` and asserts `worker.has_finished()`. It passes
  when your `StartFrame` reaches the sink and the main loop drains the queue.

- [ ] **Implement: lifecycle events and main loop.** Add the optional initial
  metrics frame (gated on `params.enable_metrics and params.send_initial_empty_metrics`),
  then implement the main `while running` loop that drains `self._push_queue`,
  queues each frame into `self._pipeline`, waits for a terminal frame when
  needed, and calls `self._cleanup`. Then run:

  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_task_started_ended_event_handler
  ```

  The test registers `on_pipeline_started` and `on_pipeline_finished` handlers
  and verifies both fire in the correct order with the correct frame types.
  `on_pipeline_started` is triggered from `_sink_push_frame` when `StartFrame`
  exits; `on_pipeline_finished` fires when `EndFrame` exits. (Those handlers
  live in `_sink_push_frame`, not in `_process_push_queue` — but they cannot
  fire unless your implementation correctly queues `StartFrame` first and then
  drains the queue for `EndFrame`.)

- [ ] **Implement: heartbeats.** Verify that `_maybe_start_heartbeat_tasks` is
  called from the right place — it must be called from `_sink_push_frame` when
  `StartFrame` exits (that code is already there; confirm you haven't broken it).
  Then run:

  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_task_heartbeats
  ```

  The test constructs a `PipelineWorker` with `PipelineParams(enable_heartbeats=True,
  heartbeats_period_secs=0.2)` and counts `HeartbeatFrame`s via an observer.
  It expects at least 5 heartbeats with elapsed time ≥ 4 × 0.2 s.

- [ ] **Edge case: idempotent run / double start.** Read `run()` — what is the
  very first check it performs? What happens if you call `run()` a second time
  on a worker that has already finished? What happens if you call `run()` on a
  worker that was given a `name=` vs one without? Write a brief explanation (no
  code required). Then verify `test_task_single` still passes; it implicitly
  tests the single-run path.

- [ ] **Reflect.** Close the file and write from memory: (1) the sequence of
  events from `run()` being called to the first user frame being processed,
  (2) which two `asyncio.Event`s gate the startup and shutdown sequences,
  (3) where you would look first if a bot reported "wrong sample rate" or
  "metrics not appearing".

---

## Stuck? (paste into Claude)

> I'm reimplementing `_process_push_queue` in `src/pipecat/pipeline/worker.py`.
> Here's my attempt:
>
> ```python
> [your code here]
> ```
>
> Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`PipelineWorker` bootstraps every pipeline the same way: a single `StartFrame`
built from `PipelineParams` is the first thing every processor sees, carrying
all the configuration that cannot be changed mid-call (sample rates, metrics
flags, tracing context). The startup is sequenced with two `asyncio.Event`s —
one that gates the main push loop until `StartFrame` exits the sink, and one
that gates the worker's outer `run()` loop until a terminal frame exits — so
there is never a race between configuration and processing. When debugging a bot
that ignores params, start at the `PipelineParams` passed to `__init__`: if the
wrong values are there, `StartFrame` is wrong, and every downstream processor is
misconfigured from frame zero.

---

## Next → 11-heartbeats.md
