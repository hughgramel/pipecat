# Stage 08 — Pipeline wiring

## Where you are

```
Pipeline.__init__
        │
        ▼
 _link_processors()   ◄── you are here
        │
        ▼
[PipelineSource] ──► p1 ──► p2 ──► ... ──► [PipelineSink]
      │                                           │
      │  upstream frames exit here                │  downstream frames exit here
      ▼                                           ▼
 self._upstream_push_frame()            self._downstream_push_frame()
(routes back to PipelineWorker)          (routes back to PipelineWorker)
```

`_link_processors` is called once at construction time. It wires every adjacent pair in `self._processors` so frames have a path to travel. Without it the list is just a list — nothing flows.

---

## How it works (orientation — answer these before you code)

**Q: What does `_link_processors` do, at the highest level?**
A: It makes the flat list `self._processors` into a linked chain by calling `link()` on every adjacent pair. After it returns, a frame queued at `self._processors[0]` will propagate all the way to `self._processors[-1]` without any further coordination.

**Q: Where do `PipelineSource` and `PipelineSink` come from, and when are they added?**
A: `Pipeline.__init__` builds `self._processors` as `[self._source, *processors, self._sink]` before calling `_link_processors`. So by the time `_link_processors` runs, the bracket nodes are already in the list — the method only needs to loop over `self._processors` as-is, without prepending or appending anything itself.

**Q: How does the `link()` loop work? (Stage 04 recap)**
A: Start with `prev = self._processors[0]`. For each `curr` in `self._processors[1:]`, call `prev.link(curr)`, then advance `prev = curr`. One pass; O(n) calls; no index arithmetic needed.

**Q: When a frame travels upstream, how does it leave the pipeline?**
A: It reaches `PipelineSource`, whose `process_frame` checks `FrameDirection.UPSTREAM` and calls `self._upstream_push_frame(frame, direction)`. That handler is `Pipeline.push_frame`, supplied at construction — so the frame exits the pipeline and arrives at whoever is outside it (e.g. `PipelineWorker`).

**Q: In a real bot, where do you add a new processor?**
A: In the list you pass to `Pipeline(...)`. For example:

```python
Pipeline([transport.input(), stt, llm, tts, transport.output()])
```

`_link_processors` will wire them all — you never call `link()` by hand.

**Predict-the-behavior questions:**
1. If you call `Pipeline([])` (empty list), `self._processors` is `[source, sink]`. What does `_link_processors` produce, and does a frame still flow?
2. If you forget to prepend the source in `self._processors`, what happens when a downstream frame arrives at `Pipeline.process_frame`?
3. Suppose you add a second call to `_link_processors` after the pipeline is already running. What could go wrong with the existing links?

---

## Why this matters

`_link_processors` is the assembly step that turns a bag of processors into a working bot — every STT/LLM/TTS/transport you add to the list becomes part of the chain here. When a processor "isn't receiving frames," the first thing to check is whether it was in the list passed to `Pipeline(...)` and whether `_link_processors` ran (it always runs in `__init__`, so a missing processor usually means it was accidentally omitted from the constructor argument). Understanding this method also makes it clear why you cannot hot-swap a processor after construction without re-linking.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/pipeline/pipeline.py`. Find `Pipeline._link_processors`. Replace its body with `raise NotImplementedError("Stage 08")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the `PipelineSource`/`PipelineSink` classes, and the `self._processors` field assignment in `__init__` so I know the contract.

---

## Your tasks (in order)

- [ ] **Read and trace.** Read `Pipeline.__init__` (lines ~99–121) and `PipelineSource.process_frame` / `PipelineSink.process_frame`. Trace what happens to a `TextFrame` sent downstream: source → p1 → … → sink → exits. Then trace an upstream frame in reverse. Write one sentence for each direction before writing any code.

- [ ] **State the contract.** Before implementing, answer in your own words: (a) what is `self._processors` when `_link_processors` is called? (b) what must be true after it returns? (c) what method from Stage 04 does all the actual wiring?

- [ ] **Gut check.** How many iterations does the loop need for a pipeline with three user processors? List the pairs that get linked. Verify your list includes `(source, p1)`, `(p1, p2)`, `(p2, p3)`, and `(p3, sink)`.

- [ ] **Implement the happy path — single processor.**
  Write the body of `_link_processors`. Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_single
  ```
  It should pass with one `IdentityFilter` in the chain.

- [ ] **Verify multiple processors.**
  Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_multiple
  ```
  This chains three `IdentityFilter` instances. If it fails, re-examine the loop boundary — are you linking every adjacent pair or stopping one short?

- [ ] **Verify the upstream exit path.**
  Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_task_queue_frame_upstream
  uv run pytest tests/test_pipeline.py::TestPipeline::test_task_queue_frames_upstream
  ```
  These tests queue frames in `FrameDirection.UPSTREAM` and assert the `PipelineWorker` receives them via `on_frame_reached_upstream`. If they fail, check that `PipelineSource._upstream_push_frame` is reachable — meaning source is correctly at position 0 and linked to the rest of the chain.

- [ ] **Edge case — empty processor list.**
  Call `Pipeline([])` in a scratch test or a REPL. Confirm it doesn't raise. After `_link_processors`, `self._processors` is `[source, sink]`. One `link()` call should connect them. Trace what happens to a downstream frame: source → sink → exits immediately. Is that the right behavior?

- [ ] **Reflect.** In one paragraph: why does bracketing with source/sink make the pipeline composable (i.e., a `Pipeline` is itself a `FrameProcessor` that can be nested inside another pipeline)? What role does `_upstream_push_frame` play in making that work across nesting levels?

---

## Stuck? (paste into Claude)

> I'm reimplementing `Pipeline._link_processors`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`_link_processors` is a three-line method with outsized consequences: it is the moment a configuration list becomes a running bot. Every processor you choose for a real voice agent — transport input, VAD, STT, LLM, TTS, transport output — becomes part of the chain because of this loop. The source/sink bracket keeps the pipeline self-contained and composable: upstream frames exit through the source's handler (wired to whoever owns the pipeline), downstream frames exit through the sink's handler, and the pipeline itself looks like a single `FrameProcessor` to anything outside it. That composability is what lets you nest `Pipeline` inside `ParallelPipeline` (next stages) without changing how linking works.

---

## Next → 09-pipeline-routing.md
