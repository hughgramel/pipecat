# Stage 09 — Pipeline frame routing

## Where you are

```
                 ┌──────────────────────────────────────────────────────┐
                 │                  Pipeline (FrameProcessor)           │
                 │                                                      │
  frame, DOWN ──►│──► _source.queue_frame(frame, DOWNSTREAM) ──►──►──► │──► (out)
                 │         [PipelineSource]   [proc1] [proc2] [sink]   │
  frame, UP   ◄──│◄── _sink.queue_frame(frame, UPSTREAM)   ◄──◄──◄─── │◄── (in)
                 │         [PipelineSink]                               │
                 │                                                      │
                 │          ▲ process_frame lives here                  │
                 └──────────────────────────────────────────────────────┘

   Outer Pipeline sees this whole box as ONE FrameProcessor — nestable.
```

## How it works (orientation — answer these before you code)

**Q: Why does `Pipeline` subclass `FrameProcessor`?**
A: Because a pipeline is itself a processing node. Anything that receives a frame and emits frames is a `FrameProcessor`. Subclassing lets a `Pipeline` be wired into a larger pipeline just like any leaf processor — the outside world never needs to know (or care) that it contains more processors inside.

**Q: What does `process_frame` actually do at runtime?**
A: It looks at the `direction` argument and makes one of two calls:
- `FrameDirection.DOWNSTREAM` → `await self._source.queue_frame(frame, FrameDirection.DOWNSTREAM)` — hands the frame to `PipelineSource`, which is the head of the internal chain, so it flows through every internal processor toward the sink.
- `FrameDirection.UPSTREAM` → `await self._sink.queue_frame(frame, FrameDirection.UPSTREAM)` — hands the frame to `PipelineSink`, which is the tail of the internal chain, so it travels backward through every internal processor toward the source.

It also calls `await super().process_frame(frame, direction)` first — the base class handles system frames (like `StartFrame`, `EndFrame`, `CancelFrame`) that need special treatment regardless of direction.

**Q: How does this enable nesting?**
A: `PipelineSource.process_frame` routes upstream frames to `self._upstream_push_frame` — which was set to the outer `Pipeline`'s own `push_frame` during construction (stage 08). So when an inner pipeline's source bubbles a frame upstream, it lands back in the outer pipeline's chain. The boundary is invisible — any `queue_frame` call on the nested `Pipeline` object triggers the same routing logic recursively.

**Q: Where in application code would you nest a sub-pipeline?**
A: Anywhere a flat processor list gets hard to reason about. Common cases: grouping STT + VAD into a shared input pipeline, or packaging a TTS + audio-output pair into a reusable output sub-pipeline. Pass the nested `Pipeline` instance in the processors list exactly like any other `FrameProcessor`.

**Predict the behavior:**

1. You have `Pipeline([ProcessorA, Pipeline([ProcessorB, ProcessorC]), ProcessorD])`. A `TextFrame` enters downstream. Which processors see it, and in what order?
2. A frame travels UPSTREAM out of `ProcessorB` in the inner pipeline. What happens when it reaches the inner pipeline's `_source`?
3. A `StopFrame` arrives at `Pipeline.process_frame` with direction `DOWNSTREAM`. The `super().process_frame` call happens first — what might the base class do with it before your routing code runs?

## Why this matters

A `Pipeline` that is itself a `FrameProcessor` lets you compose whole processing graphs the same way you compose individual processors — you can swap a leaf node for an entire sub-graph without changing the code around it. When debugging "frames enter my nested pipeline but don't come out," the first thing to check is whether `process_frame` on the outer pipeline is routing into `_source` (DOWNSTREAM) vs. `_sink` (UPSTREAM) correctly — a swapped direction silently discards frames at the wrong end of the inner chain.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/pipeline/pipeline.py`. Find `Pipeline.process_frame`. Replace its body with `raise NotImplementedError("Stage 09")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the direction enum values, and the `_source`/`_sink` fields so I know the contract.

## Your tasks (in order)

- [ ] **Read and trace** — Before writing any code, read `Pipeline.__init__` to see where `_source` and `_sink` come from (stage 08 wired them). Then read `PipelineSource.process_frame` and `PipelineSink.process_frame` to understand what `queue_frame` triggers on each end. Explain in one sentence what each of the two routing branches does.

- [ ] **Write the contract** — In a comment or on paper, write the two branches: `if direction == DOWNSTREAM → _source.queue_frame(..., DOWNSTREAM)`; `elif direction == UPSTREAM → _sink.queue_frame(..., UPSTREAM)`. Do NOT open your editor yet. Confirm: which object owns the internal chain's entry point, and which owns the exit point?

- [ ] **Gut check** — `queue_frame` is async. Does your implementation need to `await` it? Check `FrameProcessor.queue_frame`'s signature to confirm, then answer why this matters for backpressure.

- [ ] **Implement downstream routing** — Write the `DOWNSTREAM` branch only (plus the `super()` call). Run:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_single
  ```
  This test sends a `TextFrame` through `Pipeline([IdentityFilter()])` and expects it to arrive downstream. It should pass with only the downstream branch.

- [ ] **Add the multi-processor test** — Confirm the chain still works with three processors chained:
  ```
  uv run pytest tests/test_pipeline.py::TestPipeline::test_pipeline_multiple
  ```
  This test uses `Pipeline([identity1, identity2, identity3])` with the same `TextFrame`. If it passes, your downstream routing is correct for any chain length.

- [ ] **Edge case: upstream routing into `_sink`** — Add the `UPSTREAM` branch. Think through: what frame type would travel upstream in a real voice pipeline? (Hint: consider `ErrorFrame` or an acknowledgment frame.) To verify your upstream branch, look at `PipelineSink.process_frame` — trace what it does with an UPSTREAM frame. There is no dedicated pytest checkpoint for this branch in the two named tests; reason through it by reading `PipelineSink` rather than guessing.

- [ ] **Reflect** — In your own words: why does `process_frame` call `super().process_frame` first, before the direction branches? What category of frames does the base class handle that your routing code should not try to re-route? If you removed the `super()` call, which system frames might break?

## Stuck? (paste into Claude)

> I'm reimplementing `Pipeline.process_frame`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`Pipeline.process_frame` is the seam that makes nested pipelines possible. By routing DOWNSTREAM frames into `_source` and UPSTREAM frames into `_sink`, a `Pipeline` exposes exactly the same interface to its parent as any leaf `FrameProcessor` does. You can now compose entire processing graphs — group STT + language detection as one "input pipeline," wrap TTS + output transport as another — and wire them together or swap them out without touching the code that sits above or below them in the chain.

## Next → 10-worker-startup.md
