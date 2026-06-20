# Pipecat Internals — A Reconstruction Practicum

## Laboratory Reference: The Test Harness

Read this document before Assignment 01. This practicum never requires you to author a
test. Every assignment is verified by Pipecat's own `pytest` suite — the same tests run
by the maintainers and by continuous integration. This reference explains how to use
that suite.

### 1. Running the test suite

All commands are issued from the repository root
(`/Users/hughgramelspacher/repos/pipecat`). On its first invocation, `uv` installs the
project dependencies; this first run is slow (approximately 20 seconds), after which an
individual test runs in one to two seconds.

```bash
uv run pytest                                   # the entire suite
uv run pytest tests/test_frame_processor.py     # a single file
uv run pytest tests/test_frame_processor.py::TestFrameProcessor::test_broadcast_interruption  # a single test
```

The final form — `file::Class::test_name` — is used most frequently. Each assignment's
task list supplies the exact path.

### 2. Command-line options

| Option | Effect | Typical use |
|--------|--------|-------------|
| `-x` | stop at the first failure | isolating the first error after a change |
| `-k "broadcast"` | run tests whose name matches an expression | running a related group without full paths |
| `--lf` | re-run only the tests that failed last time | the implement-and-verify loop |
| `-q` | quiet output (one character per test) | reduced noise |
| `-s` | do not capture standard output (`print` is shown) | debugging with print statements |
| `-v` | verbose test names (enabled by default in this repository) | confirming which test ran |

A representative tight loop during implementation:

```bash
uv run pytest tests/test_frame_processor.py -k interruption -x -q
```

### 3. Provenance: why `pytest`, and is it genuinely Pipecat's?

It is. `pytest`, `pytest-asyncio`, and `pytest-aiohttp` are declared in the development
dependency group of `pyproject.toml`; the suite is configured under
`[tool.pytest.ini_options]`; and `uv run pytest` is named as the test command in the
repository's own `AGENTS.md`. The configuration sets `pythonpath = ["src"]`, which is
why the command locates the source without an installation step. When an assignment
directs you to run a test, you are running the maintainers' own specification.

### 4. Reading a failing test as a specification

This is the central skill of the practicum. When you remove a function, its covering
test fails, and **that failing test is the specification**: it states precisely what the
function must do. The procedure:

1. **Open the test file** named in the assignment and read the failing test in full.
2. **Identify the inputs and expected outputs.** Most tests construct a processor and
   provide `frames_to_send` together with `expected_down_frames` and/or
   `expected_up_frames`.
3. **Read the assertions** — `self.assertEqual(...)`, `assert x == y`, or the expected
   frame lists. These are the invariants your implementation must satisfy.
4. **Make the smallest change** that satisfies one assertion, re-run, and repeat.

You are never required to guess the definition of correct behavior; the test encodes it.

### 5. The `run_test` harness

Most Pipecat tests do not invoke a processor directly. They use `run_test()` from
`src/pipecat/tests/utils.py`, which assembles the processor into a working `Pipeline`,
`PipelineWorker`, and `WorkerRunner`, sends frames through it, and asserts on the frames
that emerge in each direction. Reading this function once makes the entire suite legible.

The signature, reduced to its salient parameters:

```python
async def run_test(
    processor,                       # the FrameProcessor under test
    *,
    frames_to_send,                  # the list of frames to send in
    expected_down_frames=None,       # frame TYPES expected to flow downstream
    expected_up_frames=None,         # frame TYPES expected to flow upstream
    frames_to_send_direction=FrameDirection.DOWNSTREAM,
    ignore_start=True,               # disregard the StartFrame when asserting
    send_end_frame=True,             # send an EndFrame to close the pipeline
) -> tuple[Sequence[Frame], Sequence[Frame]]:   # (downstream, upstream) received
```

A representative example, lightly abridged from `tests/test_user_turn_processor.py`:

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

Two points warrant attention:

- **`expected_down_frames` are types, not instances.** The harness verifies the *kind*
  of each emitted frame, in order.
- **`SleepFrame(sleep=N)`** may be included in `frames_to_send` to introduce a genuine
  timing gap, which is used to test timeouts (for example, "the user stopped speaking
  but no transcription arrived").

### 6. Asynchronous tests

Because Pipecat is built on `asyncio`, its tests are `async def` and run under
`pytest-asyncio`. Two styles occur in the repository:

- **Class-based:** a subclass of `unittest.IsolatedAsyncioTestCase` with `async def
  test_*` methods using `self.assertEqual(...)`. This is the majority of the suite.
- **Function-based:** a bare `async def test_*()` decorated with `@pytest.mark.asyncio`,
  using plain `assert`.

You will seldom author such tests, since you are reconstructing code that existing tests
already cover. The exception is the assignments described in §7.

### 7. Assignments without a dedicated test

Assignments 01, 02, 03, 18, and 19 concern infrastructure or abstract methods that have
no dedicated unit test; they are exercised only indirectly. These assignments state this
explicitly and provide a **lightweight verification harness** — a few lines you run
yourself to demonstrate that your reconstruction is correct — in place of a test that
does not exist.

For a service base class (Assignments 18 and 19), the harness subclasses the base with a
minimal `run_stt` or `run_tts` that yields a single frame, drives it with `run_test()`,
and asserts the type of the emitted frame. For a pure data structure (Assignments 01 and
03) the harness is smaller still: construct the object, call the method, and assert the
invariant directly. The assignment supplies the exact code.

### 8. When you are unable to proceed

1. Re-read the failing assertion; it names the exact expected value.
2. Run `git restore <file>` to view the original implementation, study how it operates,
   then isolate the function again and reattempt from your own understanding.
3. Each assignment provides a "Stuck?" prompt that directs the assistant to pose a
   guiding question rather than supply the answer.

→ Return to [`00-START-HERE.md`](00-START-HERE.md), or open the Completion Record
([`PROGRESS.md`](PROGRESS.md)) to record finished assignments.
