# Progress Tracker

Check a stage off when its listed `pytest` checkpoints are all green. `main` stays
pristine as your reference; `git restore <file>` un-guts any stage you want to redo.

> Tip: from the repo root, `git diff main --stat` shows which files you've touched.

## Foundations — the atom and the queues
- [ ] 01 — Frame base class (`Frame.__post_init__`)
- [ ] 02 — Priority-queue routing (`FrameProcessorQueue.put`)
- [ ] 03 — Frame queue + uninterruptible tracking (`FrameQueue.reset`)

## The node — FrameProcessor
- [ ] 04 — Processor linking (`FrameProcessor.link`)
- [ ] 05 — Push-frame routing (`__internal_push_frame`)
- [ ] 06 — Interruption broadcast (`broadcast_interruption`)
- [ ] 07 — Interruption handling (`_start_interruption`)

## The chain — Pipeline & Worker
- [ ] 08 — Pipeline wiring (`Pipeline._link_processors`)
- [ ] 09 — Pipeline routing (`Pipeline.process_frame`)
- [ ] 10 — PipelineWorker startup
- [ ] 11 — Heartbeat monitoring

## VAD — when is the user speaking
- [ ] 12 — VAD confidence state machine (`VADAnalyzer._run_analyzer`)
- [ ] 13 — VADController event dispatch
- [ ] 14 — VADProcessor frame injection

## Conversation — context & aggregation
- [ ] 15 — LLMContext (the conversation state)
- [ ] 16 — LLMUserAggregator (collecting a user turn)
- [ ] 17 — LLMAssistantAggregator (collecting the LLM reply)

## The AI & I/O edges
- [ ] 18 — STTService base (audio → text)
- [ ] 19 — TTSService base (text → audio)
- [ ] 20 — LLMService function-call dispatch
- [ ] 21 — BaseInputTransport (audio ingress)
- [ ] 22 — WorkerRunner (the outermost loop)

---

**Capstone:** after stage 22, run the whole suite — `uv run pytest` — to confirm
everything you rebuilt is green together. Then check this:

- [ ] 🎓 Full suite green; traced one audio frame from transport-in → transport-out
      from memory.
