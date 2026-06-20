# Stage 11 — Heartbeat monitoring

## Where you are

```
PipelineWorker
│
├─ _heartbeat_push_handler()   ← YOU ARE HERE
│   │  loops: queue HeartbeatFrame(timestamp) → pipeline every heartbeats_period_secs
│   ▼
│  [Source] → [Proc A] → [Proc B] → ... → [Sink]
│                                             │
│                                    HeartbeatFrame arrives at Sink
│                                    → enqueued onto _heartbeat_queue
│                                             │
└─ _heartbeat_monitor_handler()  ◄────────────┘
    │  wait_for(queue.get(), timeout=heartbeats_monitor_secs)
    │  OK  → log round-trip time (trace level)
    └─ TIMEOUT → logger.warning("heartbeat frame not received for more than N seconds")
```

The push handler and the monitor handler are two independent asyncio tasks.
Both are started lazily in `_maybe_start_heartbeat_tasks()`, which is called from
`_sink_push_frame()` the moment the `StartFrame` exits the pipeline.
Both are cancelled in `_maybe_cancel_heartbeat_tasks()` during shutdown.

---

## How it works (orientation — answer these before you code)

**Why does a pipeline need heartbeats?**

> A voice-AI pipeline is a chain of async processors. If one of them blocks
> (waiting on a slow LLM, a hung network socket, or a deadlocked queue) nothing
> breaks loudly — the pipeline simply goes silent. Heartbeat frames are a
> lightweight canary: because they must traverse every processor in sequence, a
> missed heartbeat is proof that *something* upstream is stuck.

**Runtime details — answer each before opening the implementation:**

1. What is the default send interval?
   `HEARTBEAT_SECS = 1.0` (module constant). Configurable via
   `PipelineParams.heartbeats_period_secs`.

2. What is the monitor's timeout?
   `HEARTBEAT_MONITOR_SECS = 10.0`. Configurable via
   `PipelineParams.heartbeats_monitor_secs`.

3. What happens when a heartbeat is late?
   The monitor calls `asyncio.wait_for(queue.get(), timeout=wait_time)` and
   catches `TimeoutError` → logs a `WARNING`. The loop then continues waiting;
   it does **not** cancel the pipeline automatically.

4. What timestamp does the `HeartbeatFrame` carry, and why?
   `HeartbeatFrame(timestamp=self._clock.get_time())` — nanoseconds from the
   worker's `BaseClock`. The monitor subtracts it from the arrival time to log
   how long the frame spent traversing the pipeline.

5. Why does the push handler call `self._pipeline.queue_frame()` directly instead
   of `self.queue_frame()`?
   Because `queue_frame()` funnels through `_push_queue`, which stops accepting
   frames once an `EndFrame` has been queued. The heartbeat handler bypasses that
   gate so it can keep sending right up until the pipeline is torn down.

6. When are heartbeat tasks started and stopped?
   Started: inside `_sink_push_frame()` when a `StartFrame` arrives at the sink
   (i.e., after every processor has seen the start signal).
   Stopped: inside `_cancel_tasks()` → `_maybe_cancel_heartbeat_tasks()`.

**Predict the behavior — think through these before coding:**

- Q: If `heartbeats_period_secs = 0.2` and `heartbeats_monitor_secs = 10.0`,
  roughly how many heartbeats should arrive at the monitor before a timeout
  warning fires?
  *About 50 — the monitor window is 50× the send interval.*

- Q: A `HeartbeatBlocker` processor swallows all `HeartbeatFrame`s. After how
  long will the monitor warn?
  *After `heartbeats_monitor_secs` seconds from the last heartbeat it received
  (or from startup if none ever arrived).*

- Q: If the pipeline is cancelled mid-run, what prevents the two heartbeat tasks
  from leaking?
  *`_cancel_tasks()` is called in the `finally` block of `run()`. It calls
  `_maybe_cancel_heartbeat_tasks()`, which awaits `cancel_task()` on each.*

---

## Why this matters

A blocked processor kills a live voice call silently: the bot stops speaking, the
user hears nothing, and nothing in the logs points at a root cause. Heartbeats
make the silence observable — when the warning fires you know exactly how long
the pipeline has been stuck and can correlate it with other log events (a slow
LLM call, a network timeout, a queue deadlock). Without them, debugging "the bot
just went dead mid-call" means guessing which of a dozen processors froze; with
them, the timestamp gap tells you exactly when forward progress stopped.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/pipeline/worker.py`. Find the `_heartbeat_push_handler`
> coroutine on `PipelineWorker` — the one that sends `HeartbeatFrame`s on an
> interval. Replace its body with `raise NotImplementedError("Stage 11")` but
> KEEP the signature, docstring, and any decorators exactly as they are. Don't
> touch any other code. Then show me:
> 1. The full method signature and docstring.
> 2. The two module-level constants that govern the default interval and the
>    monitor timeout.
> 3. The `HeartbeatFrame` import line so I can see what fields it exposes.

---

## Your tasks (in order)

- [ ] **Read and trace.** Before writing a line of code, read
  `_heartbeat_push_handler`, `_heartbeat_monitor_handler`, and
  `_maybe_start_heartbeat_tasks` in full. Draw (on paper or in a comment) the
  two-task loop: sender enqueues, sink re-enqueues onto `_heartbeat_queue`,
  monitor dequeues. Confirm where each task is created and cancelled.

- [ ] **Write the contract.** In your own words, state the invariant the push
  handler must uphold: *"While the pipeline is running, a `HeartbeatFrame`
  carrying the current clock time must arrive at `_heartbeat_queue` no less
  frequently than every `heartbeats_period_secs` seconds — as long as the
  pipeline is not blocked."* Identify the three things your implementation needs:
  the frame class, the clock call, and the sleep duration.

- [ ] **Gut check.** Before implementing, answer: why is `asyncio.sleep` placed
  *after* `queue_frame` rather than before? What would change if you swapped
  the order? (Hint: think about the very first heartbeat and the start-up
  sequence.)

- [ ] **Implement the periodic send loop** and verify the happy path:
  ```
  uv run pytest tests/test_pipeline.py::TestPipelineTask::test_task_heartbeats
  ```
  The test creates a pipeline with `heartbeats_period_secs=0.2`, waits for 5
  heartbeats via an observer callback, then asserts both `count >= 5` and
  `elapsed >= (5-1) * 0.2`.

- [ ] **Respect the custom monitor timeout.** Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipelineTask::test_heartbeat_monitor_respects_custom_timeout
  ```
  The test inserts a `HeartbeatBlocker` that swallows all `HeartbeatFrame`s,
  sets `heartbeats_monitor_secs=0.3`, runs for 0.6 s, then asserts the log
  contains `"more than 0.3 seconds"`. This test exercises `_heartbeat_monitor_handler`
  — make sure `wait_time` comes from `self._params.heartbeats_monitor_secs`, not
  a hard-coded constant.

- [ ] **Name the edge case: clean cancellation on shutdown.** Trace what happens
  to the two heartbeat tasks when `CancelFrame` arrives. Confirm that
  `_maybe_cancel_heartbeat_tasks()` is called and that `asyncio.CancelledError`
  propagates cleanly out of `asyncio.sleep` (no try/except suppressing it in
  your loop). No additional pytest for this step — you're tracing, not testing.

- [ ] **Reflect.** Answer: if you were debugging a production outage where the
  bot went silent 45 seconds into a call, what would you grep for in the logs
  to pinpoint which processor was stuck? What would you change in
  `PipelineParams` to detect the hang faster?

---

## Stuck? (paste into Claude)

> I'm reimplementing `_heartbeat_push_handler` on `PipelineWorker`. Here's my
> attempt: [paste your code]. Don't give me the answer — ask me one question
> that points at what I'm missing.

---

## What you learned

The heartbeat mechanism is Pipecat's built-in liveness probe: a periodic
`HeartbeatFrame` that must traverse every processor in the chain and surface at
the sink within a configurable window. When the monitor's `asyncio.wait_for`
times out and logs a warning, you have a timestamp and a pipeline to bisect.
Without heartbeats a hung processor is invisible until a user complains; with
them you can detect a stall in `heartbeats_monitor_secs` seconds and correlate
the warning with the exact moment a slow service call or a deadlocked queue
stopped forward progress.

---

## Next → 12-vad-state-machine.md
