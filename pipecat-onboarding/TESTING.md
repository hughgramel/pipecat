# How testing works here (read before stage 01)

This course never asks you to invent a test. Every stage is verified by Pipecat's
**own** pytest suite — the same tests the maintainers and CI run. This page is your
manual for using it.

---

## Running tests

All commands run from the repo root (`/Users/hughgramelspacher/repos/pipecat`). `uv`
installs deps on first run (the first invocation is slow, ~20s; after that single
tests run in ~1–2s).

```bash
uv run pytest                                   # the whole suite
uv run pytest tests/test_frame_processor.py     # one file
uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption  # ONE test
```

That last form — `file::Class::test_name` — is the one you'll use constantly. Each
stage's task list gives you the exact path to paste.

### Flags worth knowing

| Flag | What it does | When |
|------|--------------|------|
| `-x` | stop at the first failure | you broke something and want the first error only |
| `-k "broadcast"` | run tests whose name matches | run a related cluster without typing full paths |
| `--lf` | "last failed" — rerun only what failed last time | tight red→green loop while rebuilding |
| `-q` | quiet (one char per test) | less noise |
| `-s` | don't capture stdout (`print` shows) | debugging with print statements |
| `-v` | verbose names (this repo sets it by default) | see exactly which test ran |

Example tight loop while rebuilding a stage:
```bash
uv run pytest tests/test_frame_processor.py -k interruption -x -q
```

---

## Why pytest, and is it really Pipecat's?

Yes — it's declared in `pyproject.toml` (`pytest`, `pytest-asyncio`, `pytest-aiohttp`
in the dev group), configured under `[tool.pytest.ini_options]`, and named as *the*
test command in the repo's own `AGENTS.md`. The config sets `pythonpath = ["src"]`,
which is why `uv run pytest` finds the source without installing the package. When a
stage tells you to run a test, you are running the maintainers' real spec.

---

## How to read a failing test as a spec

This is the core skill of the course. When you gut a function, its covering test goes
red. **That red test is the specification** — it tells you exactly what the function
must do. Workflow:

1. **Open the test file** named in the stage and read the failing test top to bottom.
2. **Find what it sends and what it expects.** Most tests build a processor, list
   `frames_to_send`, and list `expected_down_frames` / `expected_up_frames`.
3. **Read the assertions** — `self.assertEqual(...)`, `assert x == y`, or the frame
   lists. Those are the invariants your code has to make true.
4. **Make the smallest change** that turns one assertion green, rerun, repeat.

You are never guessing what "correct" means — the test already encodes it.

---

## The one helper you'll see everywhere: `run_test()`

Most Pipecat tests don't poke a processor directly. They use `run_test()` from
`src/pipecat/tests/utils.py`, which wires your processor into a real `Pipeline` +
`PipelineWorker` + `WorkerRunner`, sends frames through it, and asserts on the frames
that come out each direction. Read it once and every test makes sense.

Signature (trimmed to what matters):

```python
async def run_test(
    processor,                       # the FrameProcessor under test
    *,
    frames_to_send,                  # list of Frames you push in
    expected_down_frames=None,       # frame TYPES you expect flowing downstream
    expected_up_frames=None,         # frame TYPES you expect flowing upstream
    frames_to_send_direction=FrameDirection.DOWNSTREAM,
    ignore_start=True,               # ignore the StartFrame in assertions
    send_end_frame=True,             # auto-send EndFrame to close the pipeline
) -> tuple[Sequence[Frame], Sequence[Frame]]:   # (downstream, upstream) received
```

A real example, lightly trimmed from `tests/test_user_turn_processor.py`:

```python
frames_to_send = [UserStartedSpeakingFrame(), TranscriptionFrame(...), UserStoppedSpeakingFrame()]
expected_down_frames = [UserStartedSpeakingFrame, UserStoppedSpeakingFrame]

await run_test(
    pipeline,
    frames_to_send=frames_to_send,
    expected_down_frames=expected_down_frames,
)
self.assertTrue(should_start)   # event handlers fired
self.assertTrue(should_stop)
```

Two things to notice:
- **`expected_down_frames` are TYPES, not instances** — the helper checks the *kind*
  of frame that came out, in order.
- **`SleepFrame(sleep=N)`** can be put in `frames_to_send` to insert a real timing gap
  — used to test timeouts (e.g. "user stopped but no transcription arrived").

---

## Async tests (you'll write a couple)

Pipecat is `asyncio`, so tests are `async def` and run under `pytest-asyncio`. Two
styles appear in this repo:

- **Class style:** a `unittest.IsolatedAsyncioTestCase` subclass with `async def
  test_*` methods and `self.assertEqual(...)`. (Most of the suite.)
- **Function style:** a bare `async def test_*()` decorated with
  `@pytest.mark.asyncio`, using plain `assert`.

You rarely write these from scratch — you're rebuilding code that *existing* tests
cover. The exception is the thin-mock stages below.

---

## Stages with no isolated test (the thin-mock pattern)

Stages **01, 02, 03, 18, 19** target infrastructure or abstract methods that have no
dedicated unit test — they're exercised only indirectly. Those pages tell you so and
give you a **thin mock**: a few lines you run yourself to prove your rebuild works,
instead of pretending a test exists.

The shape (e.g. for an STT/TTS service in stage 18/19): subclass the base, give it a
fake `run_stt`/`run_tts` that yields one frame, then drive it with `run_test()` and
assert the output frame type. The page gives you the exact snippet.

For a pure data structure (stage 01/03) the thin mock is even smaller — construct it,
call the method, `assert` the invariant. No framework needed.

---

## When you're stuck

1. Re-read the failing assertion — it names the exact expected value.
2. `git restore <the-file>` un-guts the function back to the real implementation so
   you can read how it actually does it, then gut it again and retry.
3. Each stage has a "Stuck?" prompt that makes Claude ask you a pointed question
   instead of handing over the answer.

→ Back to `00-START-HERE.md`, or open `PROGRESS.md` to track stages as you finish.
