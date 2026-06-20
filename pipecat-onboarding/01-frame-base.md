# Stage 01 — Frame base class

## Where you are

```
  transport-in → [VAD] → STT → user-agg → LLM → assistant-agg → TTS → transport-out
  └─────────────────────── PipelineWorker ─────────────────────────────────────────┘
                           run by WorkerRunner

  Every arrow above carries Frames ← you build the Frame (the atom flowing through every box).
```

The `Frame` class is the single unit of currency in Pipecat. Nothing moves through a pipeline that is not a `Frame` (or a subclass). Before you can wire up any processor, aggregator, or transport, you need to understand what every frame carries and how it gets those values.

---

## How it works (orientation — answer these before you code)

**What does `__post_init__` do, and why does every frame need it?**

`Frame` is a `@dataclass`. All six instance attributes — `id`, `name`, `pts`, `broadcast_sibling_id`, `metadata`, and `transport_source`/`transport_destination` — are declared with `field(init=False)`, which means they are *not* constructor arguments. Python calls `__post_init__` automatically right after `__init__` finishes, so that is the one guaranteed place to set them. Without `__post_init__`, every frame instance would have unset attributes and could not be logged, traced, or routed correctly.

**It's a `@dataclass` — what does `__post_init__` run after?**

Python's generated `__init__` runs first and assigns any `init=True` fields (none for `Frame` itself, but subclasses may add them). Then Python calls `__post_init__` immediately. Subclasses that add their own `init=True` fields must call `super().__post_init__()` if they need the base behaviour — though in practice Pipecat subclasses rely on inheritance and the MRO handles it automatically.

**How are ids generated — a counter, a UUID?**

There are two separate integer sequences, both defined in `src/pipecat/utils/utils.py`:

- `obj_id()` — a **global** monotonically-increasing counter (`itertools.count`) protected by a `threading.Lock`. It returns the next integer across *all* objects ever constructed. This is `frame.id`.
- `obj_count(obj)` — a **per-class** counter, also backed by `itertools.count` but keyed by `obj.__class__.__name__`. It returns how many instances of that exact class have been created so far. This feeds into `frame.name`.

The resulting name looks like `TextFrame#3` — class name, `#`, instance index for that class. No UUIDs, no random bytes: deterministic integer sequences that are cheap and log-friendly.

**Where would you add a new frame type or change the taxonomy?**

All frame definitions live in `src/pipecat/frames/frames.py`. To add a new concrete frame you subclass one of the three tier classes (see below) and add `@dataclass` plus any fields you need. To change which tier a frame belongs to, change its parent class in that file.

**Three-tier taxonomy:**

| Class | Behaviour |
|---|---|
| `SystemFrame(Frame)` | Highest priority; not cancelled by interruptions; processed immediately |
| `DataFrame(Frame)` | Ordered delivery; carries audio, text, video, LLM context; *cancelled* by user interruptions |
| `ControlFrame(Frame)` | Ordered like data frames; carries control signals (end, settings updates); *cancelled* by interruptions |

`UninterruptibleFrame` is a **mixin** (also a `@dataclass`, but it does not extend `Frame`). Mix it into a `DataFrame` or `ControlFrame` to mark instances that must survive an interruption and be delivered to completion regardless. Example usage: `class EndFrame(UninterruptibleFrame, ControlFrame): pass`.

**Predict-the-behaviour questions:**

1. You construct `TextFrame("hello")` then `TextFrame("world")`. Will `frame_a.id == frame_b.id`? Will `frame_a.name == frame_b.name`? *(Both ids differ; both names are `TextFrame#N` and `TextFrame#N+1` for successive N.)*
2. You subclass `DataFrame` to create `MyCustomFrame`. What does `MyCustomFrame#0.name` look like after the very first instance? *(It uses `MyCustomFrame` as the class name, and `#0` because the per-class counter for `MyCustomFrame` starts at 0.)*
3. You add `extra: str` to the metadata dict of a frame in-flight. Does the next frame you construct share that dict? *(No — `__post_init__` creates a fresh `{}` for every frame instance.)*

---

## Why this matters

When something goes wrong in a live voice bot — a dropped sentence, an echo, an out-of-order response — the first thing you do is read the log and find the frame by its `name` field (`TextFrame#42`) or filter on `id`. Every log line, every observer callback, and every metrics event is keyed to these values. Understanding exactly how they are assigned tells you whether two log lines refer to the same frame object and which processor created it.

---

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/frames/frames.py`. Find `Frame.__post_init__`. Replace its body with `raise NotImplementedError("Stage 01")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature and any class fields so I know the contract I'm implementing.

---

## Your tasks (in order)

- [ ] **1 — Read and trace before you touch anything.**
  Open `src/pipecat/frames/frames.py` and read the `Frame` class (lines 59–97). Then open `src/pipecat/utils/utils.py` and read `obj_id` and `obj_count`. In your own words: what is the difference between the two counters? What would happen to the count if you constructed ten `TextFrame` instances and then one `AudioRawFrame`?

- [ ] **2 — State the contract.**
  Before opening an editor, write down (in a comment or on paper) the exact postconditions `__post_init__` must establish. There are six attributes. For each one write: its type, its initial value, and whether it can be `None`. Check against the `Frame` class docstring.

- [ ] **3 — Gut the implementation (paste the Step 1 prompt into Claude).**
  After Claude replaces the body with `raise NotImplementedError`, run:
  ```bash
  python -c "from pipecat.frames.frames import TextFrame; TextFrame()"
  ```
  Confirm you see `NotImplementedError: Stage 01`. This is your red baseline.

- [ ] **4 — Implement `id` and `name` first, then run the thin-mock checkpoint.**
  Restore (or rewrite) just the `id` and `name` lines:
  ```python
  self.id: int = obj_id()
  self.name: str = f"{self.__class__.__name__}#{obj_count(self)}"
  ```
  Note: there is **no isolated unit test** for `Frame.__post_init__` in the test suite. The closest coverage is `tests/test_frame_processor.py`, which constructs frames implicitly. Instead, run this thin-mock checkpoint directly:
  ```bash
  python -c "
  from pipecat.frames.frames import TextFrame
  a = TextFrame()
  b = TextFrame()
  assert a.id != b.id, 'ids must differ'
  assert len(a.name) > 0, 'name must be non-empty'
  assert 'TextFrame' in a.name, 'name must contain class name'
  print('checkpoint 4 passed:', a.name, b.name)
  "
  ```

- [ ] **5 — Add the remaining four attributes, then verify metadata isolation.**
  Add the four remaining assignments (`pts`, `broadcast_sibling_id`, `metadata`, `transport_source`, `transport_destination`). Then run:
  ```bash
  python -c "
  from pipecat.frames.frames import TextFrame
  a = TextFrame()
  b = TextFrame()
  assert a.pts is None
  assert a.broadcast_sibling_id is None
  assert a.metadata == {}
  assert a.transport_source is None
  assert a.transport_destination is None
  a.metadata['key'] = 'value'
  assert 'key' not in b.metadata, 'metadata dicts must be independent per frame'
  print('checkpoint 5 passed')
  "
  ```
  This edge case — two frames sharing the same dict — would happen if you wrote `metadata: dict = {}` as a class-level default instead of creating a fresh dict in `__post_init__`.

- [ ] **6 — Verify the taxonomy and the `UninterruptibleFrame` mixin.**
  Confirm your understanding with:
  ```bash
  python -c "
  from pipecat.frames.frames import Frame, SystemFrame, DataFrame, ControlFrame, UninterruptibleFrame, EndFrame
  assert issubclass(SystemFrame, Frame)
  assert issubclass(DataFrame, Frame)
  assert issubclass(ControlFrame, Frame)
  # UninterruptibleFrame is a mixin — it does NOT extend Frame
  assert not issubclass(UninterruptibleFrame, Frame)
  # EndFrame combines the mixin with ControlFrame
  assert issubclass(EndFrame, UninterruptibleFrame)
  assert issubclass(EndFrame, ControlFrame)
  print('checkpoint 6 passed')
  "
  ```

- [ ] **7 — Reflect.**
  Answer these before moving on: (a) Why are `id` and `name` declared `field(init=False)` rather than being normal dataclass fields with defaults? (b) If you swapped `obj_id()` for `uuid.uuid4()`, what would break in the existing codebase? (c) A `SystemFrame` and a `DataFrame` are both constructed in the same test. Can their `id` values collide? Why or why not?

---

## Stuck? (paste into Claude)

> I'm reimplementing `Frame.__post_init__`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

---

## What you learned

Every frame in Pipecat — whether it carries a 20 ms audio chunk, a completed transcript, or an end-of-pipeline signal — is automatically tagged at birth with a unique integer `id` and a human-readable `name` combining class and instance count. When you read frame dumps in logs or step through an observer callback, those two fields are how you answer "which frame is this and how many of its kind have been created?" The three-tier taxonomy (`SystemFrame` / `DataFrame` / `ControlFrame`) plus the `UninterruptibleFrame` mixin give you a vocabulary for describing not just what a frame carries but how the pipeline should treat it under pressure — a vocabulary you will use in every subsequent stage.

---

## Next → 02-priority-queue.md
