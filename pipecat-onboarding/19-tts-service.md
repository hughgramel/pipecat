# Stage 19 — TTSService base — text to audio

## Where you are

```
                  ┌─────────────────────────────────────────────────────┐
                  │                   TTSService                        │
                  │                                                     │
TTSTextFrame ───► │  process_frame                                      │
TTSSpeakFrame ──► │    │                                                │
                  │    ▼                                                │
                  │  _push_tts_frames                                   │
                  │    │                                                │
                  │    ├─ start_ttfb_metrics() ◄── TTFB clock starts   │
                  │    │                                                │
                  │    ▼                                                │
                  │  run_tts(text, context_id)  ◄── PLUGGABLE SLOT     │
                  │    │  (abstract async generator)                    │
                  │    │  yields TTSAudioRawFrame chunks                │
                  │    │                                                │
                  │    ├─ stop_ttfb_metrics()   ◄── first audio chunk  │
                  │    │                                                │
                  │    ▼                                                │
                  │  audio context queue (serialized, in-order)        │
                  └──────────────────────┬──────────────────────────────┘
                                         │
                         TTSStartedFrame │
                         TTSAudioRawFrame│  (ordered chunks)
                         TTSAudioRawFrame│
                         TTSStoppedFrame │
                                         ▼
                                  transport-out
```

`TTSService` is the mirror of `STTService`: text in, ordered audio chunks out. `run_tts` is the single abstract method you override to plug in a provider. The base class wraps every call with TTFB and processing-time metrics, serializes audio chunks through a per-context queue, and guarantees downstream ordering regardless of whether the provider is HTTP-synchronous or WebSocket-async.

---

## How it works (orientation — answer these before you code)

**Q: What is the high-level contract of TTSService?**

A: `TTSService` receives text (`TTSSpeakFrame` or aggregated `TextFrame` tokens) and emits ordered audio chunks (`TTSAudioRawFrame`) wrapped in `TTSStartedFrame` / `TTSStoppedFrame` bookends. It handles text aggregation (sentence or token mode), provider call lifecycle, metrics, interruption cleanup, and downstream ordering — all so individual provider subclasses only need to implement `run_tts`.

**Q: How does process_frame route text to run_tts?**

A: `process_frame` dispatches on frame type. A `TTSSpeakFrame` triggers `_push_tts_frames` directly with its text. Plain `TextFrame` tokens flow through `_process_text_frame` → `SimpleTextAggregator` (which buffers until a sentence boundary) → `_push_tts_frames`. `AggregatedTextFrame` (already aggregated upstream) enters `_push_tts_frames` directly. `_push_tts_frames` calls `tts_process_generator(context_id, self.run_tts(prepared_text, context_id))`, which drains the async generator and appends each yielded `Frame` to the audio context queue for that context.

**Q: What is the run_tts async generator contract?**

A: `run_tts(text: str, context_id: str) -> AsyncGenerator[Frame | None, None]`

- Yields `TTSAudioRawFrame` chunks (and optionally `TTSStartedFrame` / `TTSStoppedFrame` if `push_start_frame=False`).
- For HTTP-style providers: yield frames synchronously; the base class sees `_is_yielding_frames_synchronously=True` and closes the context after the generator returns.
- For WebSocket-style providers: yield nothing (the generator is an empty async generator); a background task appends frames to the audio context via `append_to_audio_context` and calls `remove_audio_context` when done.
- Never call `push_frame` directly — everything must go through the audio context so ordering is preserved.

**Q: Where does TTFB fit?**

A: `start_ttfb_metrics()` fires in `_push_tts_frames` just before `run_tts` is called. `stop_ttfb_metrics()` fires in `_handle_audio_context` on the first `TTSAudioRawFrame` chunk that arrives. The interval between those two calls is the time-to-first-byte: how long the provider took to produce its first audio.

**Q: Where would you add a new TTS provider?**

A: Subclass `TTSService`, implement `run_tts` as an async generator that calls your provider's API and yields `TTSAudioRawFrame` chunks. Set `push_start_frame=True` if you want the base class to emit `TTSStartedFrame`, or emit it yourself inside `run_tts` if the provider drives timing (WebSocket pattern). Optionally override `flush_audio`, `on_audio_context_completed`, and `on_audio_context_interrupted` for cleanup.

**Predict-the-behavior questions:**

1. A `TTSSpeakFrame` arrives while the pipeline is mid-LLM-response (`_processing_text=True`). What happens to `_turn_context_id`? (Hint: look at how `TTSSpeakFrame` saves and restores it.)
2. Two `TTSSpeakFrame` calls are sent back-to-back with `pause_frame_processing=False`. What guarantees the second group's `TTSAudioRawFrame` frames don't interleave with the first group's? (Hint: `_serialization_queue`.)
3. If `run_tts` raises an exception, how does it propagate? (Hint: `push_error`.)

**Test coverage note:** There is no isolated unit test for the base `TTSService.run_tts` contract in isolation. Provider-level behavior is tested in `tests/test_cartesia_tts.py` (word-timestamp normalization), `tests/test_elevenlabs_tts.py`, and `tests/test_azure_tts.py`. Frame ordering across all three delivery patterns (HTTP sync, WebSocket no-pause, WebSocket with pause) is tested in `tests/test_tts_frame_ordering.py`, which is where you will verify your implementation.

---

## Why this matters

Audio chunks that arrive out of order produce garbled, clipped, or stuttering speech — the listener hears syllables from sentence two before sentence one is done. Debugging "the bot's voice sounds scrambled" or "words are in the wrong order" almost always traces back to frames bypassing the serialization queue. The TTFB metric is equally important: if `stop_ttfb_metrics` is never called (because the first audio chunk is never pushed through the context), latency dashboards will report infinite time-to-first-byte even when the provider responded quickly.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/services/tts_service.py`. Find `TTSService.process_frame`. Replace its body with `raise NotImplementedError("Stage 19")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the abstract `run_tts` contract, and the frame types (`TTSTextFrame`, `TTSAudioRawFrame`) so I know the contract.

---

## Your tasks (in order)

- [ ] **Read and trace.** Read `TTSService.process_frame` (lines 670–808 in `tts_service.py`). Trace one `TTSSpeakFrame` through the dispatch chain: `process_frame` → `_push_tts_frames` → `tts_process_generator` → `run_tts` → `append_to_audio_context` → `_audio_context_task_handler` → `_handle_audio_context` → `push_frame`. Write a one-sentence summary of each hop in your own words before touching code.

- [ ] **State the contract before coding.** Answer in a comment block at the top of your scratch file: (a) what frame types trigger a TTS synthesis call; (b) what `run_tts` must yield and in what order; (c) what `tts_process_generator` does with those yielded frames; (d) when TTFB starts and stops.

- [ ] **Gut check — predict frame ordering.** Given two consecutive `TTSSpeakFrame` calls and `push_start_frame=True`, list the exact downstream frame sequence you expect for both (e.g. `AggregatedTextFrame`, `TTSStartedFrame`, `TTSAudioRawFrame`, `TTSStoppedFrame`, then repeat). Verify your prediction by reading `_handle_audio_context` before running any tests.

- [ ] **Implement the happy path.** Restore `process_frame`'s body. Focus first on the `TTSSpeakFrame` branch (lines 752–778): save/restore `_turn_context_id`, call `_push_tts_frames`, call `on_turn_context_completed`. Then verify with the thin-mock checkpoint below.

  **Thin-mock checkpoint** (no isolated base test exists — this is your substitute):

  ```python
  # In a scratch test file or pytest session:
  import asyncio
  from collections.abc import AsyncGenerator
  from pipecat.frames.frames import Frame, TTSAudioRawFrame, TTSSpeakFrame
  from pipecat.services.tts_service import TTSService
  from pipecat.tests.utils import run_test

  _FAKE_AUDIO = b"\x00\x01" * 320

  class FakeTTSService(TTSService):
      def __init__(self):
          super().__init__(push_start_frame=True, push_stop_frames=True,
                           push_text_frames=False, sample_rate=16000)
      def can_generate_metrics(self): return False
      async def run_tts(self, text: str, context_id: str) -> AsyncGenerator[Frame, None]:
          yield TTSAudioRawFrame(audio=_FAKE_AUDIO, sample_rate=16000,
                                 num_channels=1, context_id=context_id)

  async def checkpoint():
      tts = FakeTTSService()
      frames, _ = await run_test(
          tts,
          frames_to_send=[TTSSpeakFrame(text="hello", append_to_context=False)],
      )
      audio = [f for f in frames if isinstance(f, TTSAudioRawFrame)]
      assert len(audio) == 1, f"Expected 1 TTSAudioRawFrame, got {len(audio)}"
      print("checkpoint passed:", [type(f).__name__ for f in frames])

  asyncio.run(checkpoint())
  ```

  This confirms: text in → `run_tts` called → audio frame out in order.

- [ ] **Verify ordering is preserved.** Run the ordering test suite:

  ```bash
  uv run pytest tests/test_tts_frame_ordering.py -v
  ```

  All three patterns must pass: `test_http_tts_frame_ordering`, `test_websocket_tts_no_pause_frame_ordering`, `test_websocket_tts_with_pause_frame_ordering`. These tests use mock subclasses very similar to the thin-mock above.

- [ ] **Edge case — TTFB metrics.** Find where `start_ttfb_metrics()` is called in `_push_tts_frames` and where `stop_ttfb_metrics()` fires in `_handle_audio_context`. Add a comment in your notes explaining: what would happen to the TTFB metric if `stop_ttfb_metrics` was never called (e.g. if `run_tts` yielded nothing and no audio arrived)? Then look at the `TimeoutError` branch in `_handle_audio_context` to see how the base class protects against that case.

- [ ] **Edge case — processing metrics.** Sentence mode (`TextAggregationMode.SENTENCE`) calls `start_processing_metrics()` before `run_tts` and `stop_processing_metrics()` after. Token mode skips this. Find where these calls live in `_push_tts_frames` and explain in one sentence why per-token processing time is not meaningful to measure.

- [ ] **Reflect.** In your own words: if a user reports "the bot's voice is choppy and words sound scrambled on the second sentence", what is the first code path you would check? If they report "the first word always takes 2 seconds to play even though the API responds in 200ms", what metric would you look at and what object records it?

---

## Stuck? (paste into Claude)

> I'm reimplementing `TTSService.process_frame`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`TTSService` teaches the mirror pattern to `STTService`: instead of audio-in/text-out, it is text-in/audio-out. The key insight is that audio ordering is not free — without the serialization queue and per-context audio queues, chunks from a second sentence would race chunks from the first whenever the provider is async. TTFB wraps the provider call at the finest granularity: the interval between "text handed to provider" and "first audio byte returned". When a user hears choppy or scrambled speech, the serialization queue is where you look. When a user complains about slow first-word latency, TTFB is the metric that locates whether the delay is in the provider or in the pipeline.

---

## Next → 20-llm-function-calls.md
