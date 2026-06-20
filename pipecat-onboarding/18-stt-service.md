# Stage 18 — STTService base — audio to text

## Where you are

```
Transport-in
     │
     │  InputAudioRawFrame (raw PCM bytes)
     ▼
┌─────────────────────────────────────────┐
│           STTService                    │  ← YOU ARE HERE
│                                         │
│  process_frame                          │
│    └── process_audio_frame              │
│              │                          │
│              ▼                          │
│        ┌──────────────┐                 │
│        │  run_stt()   │ ← @abstractmethod │
│        │  (provider   │   subclass plugs │
│        │   slot)      │   in here        │
│        └──────┬───────┘                 │
│               │ AsyncGenerator[Frame]   │
│               ▼                         │
│        process_generator()              │
│               │ TranscriptionFrame      │
└───────────────┼─────────────────────────┘
                │
                ▼
        LLMUserAggregator
        (adds text to context)
```

`STTMuteFrame(mute=True)` short-circuits `process_audio_frame` — audio is dropped without calling `run_stt`.

---

## How it works (orientation — answer these before you code)

**Q: What is the service pattern?**

`STTService` extends `AIService` (which extends `FrameProcessor`). It is a base class: it provides all the routing, mute logic, TTFB tracking, and audio passthrough — but delegates the actual speech recognition to a single `@abstractmethod` called `run_stt`. Concrete providers (Deepgram, Azure, AssemblyAI, etc.) subclass `STTService` and implement `run_stt`; they inherit everything else for free.

**Q: What does `process_frame` do at runtime?**

`process_frame` is the dispatcher. For the main audio path:

1. An `InputAudioRawFrame` arrives (raw PCM from the transport).
2. `process_frame` calls `process_audio_frame(frame, direction)`.
3. `process_audio_frame` checks the mute flag, records the `user_id`, then calls `await self.process_generator(self.run_stt(frame.audio))`.
4. `process_generator` (on `AIService`) iterates the async generator returned by `run_stt`, and calls `push_frame(f)` for every non-`None` frame yielded — which pushes each `TranscriptionFrame` downstream.
5. If `audio_passthrough=True` (the default), the original `InputAudioRawFrame` is also forwarded downstream after transcription.

**Q: What is the `run_stt` contract?**

```python
@abstractmethod
async def run_stt(self, audio: bytes) -> AsyncGenerator[Frame | None, None]:
    ...
```

- Input: `audio` — raw PCM bytes from one `InputAudioRawFrame` chunk.
- Output: an `AsyncGenerator` that yields zero or more `Frame` objects (typically `TranscriptionFrame`, sometimes `InterimTranscriptionFrame` or `ErrorFrame`). Yielding `None` is allowed and skipped.
- There is no return value — the generator is consumed entirely by `process_generator`.

**Q: How does the STT mute protocol work?**

`STTMuteFrame` is a `SystemFrame` with a single field: `mute: bool`. When `process_frame` sees a `STTMuteFrame`, it sets `self._muted = frame.mute`. Then in `process_audio_frame`, the first check is `if self._muted: return` — audio is silently dropped, `run_stt` is never called, no `TranscriptionFrame` is emitted. Sending `STTMuteFrame(mute=False)` re-enables the service.

**Q: Where would you add a new STT provider?**

Create a subclass, implement `run_stt`. Everything else — frame routing, muting, TTFB tracking, keepalive, settings updates — is inherited.

**Predict-the-behavior questions (answer before coding):**

1. If `run_stt` yields three frames — an `InterimTranscriptionFrame`, a `TranscriptionFrame`, and then `None` — how many frames reach the downstream processor? Which ones?
2. What happens to audio arriving while `self._muted is True`? Does it reach `run_stt`? Does the audio passthrough fire?
3. An `STTMuteFrame(mute=True)` arrives. Then a `VADUserStartedSpeakingFrame` arrives. Then `STTMuteFrame(mute=False)` arrives. At what point does `run_stt` start receiving audio again?

**Note on tests:** There is NO framework-level unit test for `STTService.process_frame` in isolation. The test suite tests concrete provider integrations — see `tests/test_assemblyai_stt.py`, `tests/test_azure_stt.py`, `tests/test_deepgram_stt.py` — each with network mocks specific to that provider's protocol. The closest generic exercise of the STT→aggregator path is `tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_default_user_turn_strategies`, but it injects a `TranscriptionFrame` directly rather than going through `STTService`. The checkpoint below shows you how to verify `STTService.process_frame` using a thin mock.

---

## Why this matters

Every STT provider in Pipecat follows this exact contract — implement `run_stt`, get frame routing, muting, TTFB tracking, settings management, and keepalive for free. When transcripts aren't flowing, the two most common causes are both visible here: either `run_stt` is not yielding frames (provider issue), or `self._muted` is `True` because a `STTMuteFrame` was sent and never cleared (the bot's own audio was being transcribed, so the service was muted, and then a bug left it muted). Knowing this base class tells you exactly where to look.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/services/stt_service.py`. Find `STTService.process_frame`. Replace its body with `raise NotImplementedError("Stage 18")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the abstract `run_stt` contract, and the frame types (`InputAudioRawFrame`, `TranscriptionFrame`, `STTMuteFrame`) so I know the contract before I implement anything.

---

## Your tasks (in order)

- [ ] **Read and trace.** Read `src/pipecat/services/stt_service.py` in full. Follow the call chain: `process_frame` → `process_audio_frame` → `process_generator(self.run_stt(...))`. Note that `process_generator` lives on `AIService` (in `src/pipecat/services/ai_service.py`), not on `STTService` itself.

- [ ] **State the contract before writing any code.** In your own words: what does `process_frame` receive, what does it call, and what reaches the next processor in the pipeline? Include the mute path. Write this down (comment, notebook, whatever) before you touch the implementation.

- [ ] **Happy path — route audio to `run_stt` and push transcriptions.** Implement the `AudioRawFrame` branch in `process_frame`: call `process_audio_frame`, and optionally push the audio frame downstream if `self._audio_passthrough` is set. The core of `process_audio_frame` is already stubbed for you to understand — the key line is `await self.process_generator(self.run_stt(frame.audio))`.

- [ ] **Checkpoint — thin mock.** No isolated framework unit test exists, so verify with a thin mock:

  ```python
  import asyncio
  from collections.abc import AsyncGenerator
  from pipecat.frames.frames import (
      AudioRawFrame, Frame, InputAudioRawFrame, TranscriptionFrame
  )
  from pipecat.services.stt_service import STTService
  from pipecat.tests.utils import run_test

  class FakeSTT(STTService):
      async def run_stt(self, audio: bytes) -> AsyncGenerator[Frame | None, None]:
          yield TranscriptionFrame(text="hello", user_id="", timestamp="now")

  async def test_stt_routes_audio_to_transcription():
      stt = FakeSTT()
      audio_frame = InputAudioRawFrame(
          audio=b"\x00" * 3200, sample_rate=16000, num_channels=1
      )
      down_frames, _ = await run_test(
          stt,
          frames_to_send=[audio_frame],
          expected_down_frames=[InputAudioRawFrame, TranscriptionFrame],
      )
      assert isinstance(down_frames[1], TranscriptionFrame)
      assert down_frames[1].text == "hello"

  asyncio.run(test_stt_routes_audio_to_transcription())
  ```

  This is the primary verification checkpoint since no isolated test exists in the repo.

- [ ] **Edge case — muting via `STTMuteFrame`.** Implement the `STTMuteFrame` branch: set `self._muted = frame.mute`. Then verify: send `STTMuteFrame(mute=True)` followed by an `InputAudioRawFrame` — confirm no `TranscriptionFrame` appears downstream. Then send `STTMuteFrame(mute=False)` and audio again — confirm transcription resumes.

- [ ] **Integration green-light.** Run the closest existing test that exercises the STT→aggregator frame flow:

  ```bash
  uv run pytest tests/test_context_aggregators_universal.py::TestLLMUserAggregator::test_default_user_turn_strategies -v
  ```

  This test injects a `TranscriptionFrame` directly into the aggregator (no `STTService` in the pipeline), but it confirms that `TranscriptionFrame` downstream of where your `STTService` sits will correctly trigger a user turn. It should pass without touching your implementation.

- [ ] **Reflect.** In a comment or note: if a production STT provider's transcripts stopped arriving, what two places would you look first — one inside `STTService`, one inside the concrete `run_stt` subclass?

---

## Stuck? (paste into Claude)

> I'm reimplementing `STTService.process_frame`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

`STTService` is the clearest example of Pipecat's "base processor + abstract provider slot" pattern. The base class owns the frame routing contract — audio in, transcription out — and every protocol detail of how that audio flows through the pipeline. The concrete subclass owns exactly one thing: calling the provider's API. When you integrate a new STT provider, you implement `run_stt`, and when you debug a broken one, you check `self._muted` first and `run_stt`'s yield path second.

---

## Next → 19-tts-service.md
