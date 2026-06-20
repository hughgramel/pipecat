# Pipecat Internals — A Reconstruction Practicum

## Completion Record

Record each assignment as complete once all of its `pytest` checkpoints pass. The `main`
branch holds the unmodified original for reference; `git restore <file>` returns any
assignment to its original state should you wish to reattempt it.

> To review the work completed so far, run `git diff main --stat` from the repository
> root; it lists every file you have modified.

### Unit I — Foundations: the unit of data and the queues
- [ ] 01 — Frame base class (`Frame.__post_init__`)
- [ ] 02 — Priority-queue routing (`FrameProcessorQueue.put`)
- [ ] 03 — Frame queue and uninterruptible tracking (`FrameQueue.reset`)

### Unit II — The processing node: `FrameProcessor`
- [ ] 04 — Processor linking (`FrameProcessor.link`)
- [ ] 05 — Push-frame routing (`__internal_push_frame`)
- [ ] 06 — Interruption broadcast (`broadcast_interruption`)
- [ ] 07 — Interruption handling (`_start_interruption`)

### Unit III — The chain: pipeline and worker
- [ ] 08 — Pipeline wiring (`Pipeline._link_processors`)
- [ ] 09 — Pipeline routing (`Pipeline.process_frame`)
- [ ] 10 — Pipeline-worker startup
- [ ] 11 — Heartbeat monitoring

### Unit IV — Voice-activity detection
- [ ] 12 — VAD confidence state machine (`VADAnalyzer._run_analyzer`)
- [ ] 13 — VAD controller event dispatch
- [ ] 14 — VAD processor frame injection

### Unit V — Conversation: context and aggregation
- [ ] 15 — `LLMContext` (the conversation-state object)
- [ ] 16 — User aggregator (assembling a user turn)
- [ ] 17 — Assistant aggregator (assembling an assistant turn)

### Unit VI — The AI and I/O edges
- [ ] 18 — STT service base (audio → text)
- [ ] 19 — TTS service base (text → audio)
- [ ] 20 — LLM service function-call dispatch
- [ ] 21 — Base input transport (audio ingress)
- [ ] 22 — Worker runner (the outermost loop)

---

### Final assessment

After Assignment 22, run the complete suite — `uv run pytest` — to confirm that every
reconstruction passes together. Then complete the capstone exercise:

- [ ] Full suite passing; able to trace a single audio frame from `transport-in` to
      `transport-out` from memory, and to answer the conceptual examination in the
      Learning Outcomes handout ([`MASTERY.md`](MASTERY.md)).
