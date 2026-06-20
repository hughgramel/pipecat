# Stage 22 — WorkerRunner — the outermost loop

## Where you are

```
python my_bot.py
        │
        ▼
  WorkerRunner.run()
        │
        ├─ starts WorkerBus (pub/sub backbone)
        │
        ├─ installs SIGINT / SIGTERM handlers
        │
        ├─ starts each registered root worker as an asyncio task
        │         │
        │         └─ PipelineWorker
        │                   │
        │                   └─ Pipeline
        │                         ├─ InputTransport  (stage 20)
        │                         ├─ STTService      (stage 18)
        │                         ├─ LLMService      (stage 19)
        │                         ├─ TTSService      (stage 17)
        │                         └─ OutputTransport (stage 20)
        │
        └─ waits on _shutdown_event
               │
               └─ fires when: last root worker exits  (auto_end=True)
                           or: BusEndMessage arrives on the bus
                           or: end() / cancel() called directly
                           or: SIGINT / SIGTERM received

  "You built everything inside — this is the hand on the ignition."
```

## How it works (orientation — answer these before you code)

**Q: What does `WorkerRunner.run()` do at the highest level?**
A: It sets up the session (bus, signal handlers, asyncio task manager), starts every registered root worker as a background asyncio task, fires the `on_ready` event, then blocks on a `_shutdown_event` until something signals it to stop. When the event fires it cancels any remaining worker tasks, cleans up, and stops the bus.

**Q: How does the bus get started, and why does it matter?**
A: `_setup_session()` calls `await self._bus.start()` before any worker task is created. The bus must be running first because workers subscribe to it on attachment, and messages (including the `StartFrame` a `PipelineWorker` sends itself) must be deliverable from the first moment a worker runs.

**Q: How are root workers registered and started?**
A: Before `run()` is called, you register workers with `await runner.add_workers(worker)`. Each call attaches the worker to the bus and registry and queues it in `self._entries`. During `_setup_session()`, every entry in `_entries` gets handed to `_start_worker()`, which wraps it in `_run_worker()` and creates an asyncio task for it. Workers added *after* `run()` has started are handed to `_start_worker()` immediately.

**Q: What is `auto_end` and when would you set it to `False`?**
A: With `auto_end=True` (the default), `_run_worker()` checks after each root worker finishes whether any *other* root workers are still running; if none are, it sets `_shutdown_event`, causing `run()` to return. This is exactly right for a single-pipeline bot: when the pipeline is done, the process exits. Set `auto_end=False` when the runner is hosted inside a long-lived server (e.g. FastAPI) that adds and removes workers across many sessions — you then call `end()` or `cancel()` explicitly when you want the runner to stop.

**Q: How are signals handled?**
A: In `_setup_session()`, if `handle_sigint=True` the runner calls `loop.add_signal_handler(SIGINT, ...)` to hook into the asyncio event loop. The handler schedules a `_sig_cancel()` coroutine as an asyncio task, which calls `cancel(reason="interrupt signal")`. `handle_sigterm` works the same way. This means Ctrl-C in a terminal triggers a clean shutdown through the same `cancel()` path as everything else.

**Q: Why are `end()` and `cancel()` idempotent?**
A: Both methods start by checking `self._shutdown_event.is_set()` and returning immediately if it is. This makes it safe to call them from multiple places (e.g. a bus message *and* a signal arrive nearly simultaneously) without sending duplicate shutdown messages to workers or hanging.

**Predict-the-behavior questions — answer before you read the implementation:**

1. A `BusEndMessage` arrives on the bus while the runner is running. What method on the runner handles it, and what happens next?
2. You call `runner.end()` twice in quick succession. What does the second call do?
3. `auto_end=True` and you have two root workers. The first finishes. Does the runner shut down? What if the second finishes a second later?

## Why this matters

`WorkerRunner.run()` is the `python my_bot.py` moment — without it, nothing in your pipeline ever starts. Understanding the startup sequence (bus → signal handlers → workers → `on_ready`) lets you inject setup logic in exactly the right place and diagnose "the bot won't start" failures at the right layer. Understanding the shutdown paths (auto-end, explicit `end()`/`cancel()`, signals, bus messages) lets you debug "the bot hangs on Ctrl-C" or "the process exits too early" without guessing.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/workers/runner.py`. Find `WorkerRunner.run`. Replace its body with `raise NotImplementedError("Stage 22")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me:
>
> 1. The full signature of `run()` — parameters and their defaults.
> 2. The type and role of `self._bus`.
> 3. How `self._entries` is structured (what is `_WorkerEntry`?).
> 4. What `self._auto_end` controls.
> 5. What `self._shutdown_event` is and how it is used.

## Your tasks (in order)

- [ ] **Read and trace.** Read `WorkerRunner.run`, `_setup_session`, `_start_worker`, `_run_worker`, `end`, `cancel`, and `on_bus_message` in full. Trace the path from `await runner.run()` to the moment the first frame enters the pipeline. Write the trace in your own words before touching any code.

- [ ] **State the contract.** Before you write a line of implementation, write down (in comments or a scratch note) exactly what `run()` must do:
  - start the bus and register signal handlers
  - fire `on_ready` after setup
  - start every entry in `_entries` as a background task
  - block until `_shutdown_event` fires
  - cancel remaining tasks, clean up, stop the bus

- [ ] **Gut check.** Answer the three predict-the-behavior questions above. Check your answers against the source.

- [ ] **Implement: start bus and workers, fire `on_ready`.** Restore `run()` so that it calls `_setup_session`, fires `on_ready`, and then waits on `_shutdown_event`. Run:
  ```
  uv run pytest tests/test_runner.py::TestWorkerRunner::test_run_starts_bus_and_tasks
  ```

- [ ] **Auto-end: bus message triggers end.** Verify that `on_bus_message` routes `BusEndMessage` to `end()`, and that `end()` sets `_shutdown_event`. Run:
  ```
  uv run pytest tests/test_runner.py::TestWorkerRunner::test_bus_end_message_triggers_end
  ```

- [ ] **Edge case: idempotent `end`.** Confirm `end()` guards on `_shutdown_event.is_set()` so a second call is a no-op. Run:
  ```
  uv run pytest tests/test_runner.py::TestWorkerRunner::test_end_is_idempotent
  ```

- [ ] **Edge case: idempotent `cancel`.** Same guard applies to `cancel()`. Run:
  ```
  uv run pytest tests/test_runner.py::TestWorkerRunner::test_cancel_is_idempotent
  ```

- [ ] **Reflect last.** Without looking at the file, describe the full lifecycle of a single-pipeline bot from `python my_bot.py` to process exit. Include: which object starts the bus, when `on_ready` fires, what makes `_shutdown_event` fire when the pipeline finishes, and which cleanup steps happen before the process returns.

## Stuck? (paste into Claude)

> I'm reimplementing `WorkerRunner.run`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`WorkerRunner` is the integration point for everything you built across 21 stages. By tracing its startup path you can now answer any "why won't my bot start?" question precisely: is the bus not started yet? did `on_ready` fire before the worker was attached? did the pipeline worker's `run()` never get called because `_start_worker` was skipped?

By tracing its shutdown paths you can answer any "why won't my bot stop?" question: is `_shutdown_event` never set because `auto_end=False` and no one called `end()`? is the `BusEndMessage` arriving but `on_bus_message` ignoring it because `message.source == self.name`? is `cancel()` being called but the worker task not responding because it isn't listening for `BusCancelWorkerMessage`?

The runner is the seam where asyncio, signal handling, the bus, and your pipeline all meet. Knowing it well means you can debug at that seam instead of guessing.

## You did it

You have now personally reimplemented, from scratch, the `Frame` (stage 1), the priority and frame queues (stages 2–3), processor linking and frame routing (stages 4–5), the interruption mechanism (stage 6), the `Pipeline` (stage 7), the `PipelineWorker` and heartbeats (stages 8–10), the full VAD subsystem (stages 11–13), the conversation context and aggregators (stages 14–15), the STT, TTS, and LLM service contracts (stages 16–19), the input transport (stage 20), and — just now — the `WorkerRunner` that ties all of it together (stage 22).

Run the full suite to confirm every layer you built is green together:

```
uv run pytest
```

Then open any example bot under `examples/` and trace a single audio frame from the moment it arrives on the transport, through VAD, STT, the LLM, TTS, and back out — using only the mental model you assembled across these 22 stages. No docs, no comments as a crutch: you built this.

Return to `00-START-HERE.md` for the architecture map and a list of all 22 stages — you now understand every box on it.

## Next → You've finished the course. Back to 00-START-HERE.md
