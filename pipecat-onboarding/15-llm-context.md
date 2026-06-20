# Stage 15 — LLMContext — the conversation state

## Where you are

```
                   ┌─────────────────────────────────┐
  UserAggregator ──►                                 │
  (stage 16)       │   *** LLMContext ***            │◄── LLM Service reads
                   │   (shared notebook)             │    messages + tools
  AssistantAgg  ──►│                                 │
  (stage 17)       └─────────────────────────────────┘

  Both aggregators WRITE to the same LLMContext object.
  The LLM service READS it (via get_messages / tools).
  Neither party should mutate each other's inputs.
```

## How it works (orientation — answer these before you code)

**Q: What does LLMContext hold?**
Three things: a list of conversation messages (`_messages`), an optional set of tool definitions (`_tools`), and an optional tool-choice strategy (`_tool_choice`). Everything else — images, audio, system prompts — is just a particular kind of message in that list.

**Q: What message format does it use?**
OpenAI's `ChatCompletionMessageParam` — a dict with at minimum a `"role"` key (`"system"`, `"user"`, `"assistant"`, `"tool"`) and a `"content"` value that is either a plain string or a list of typed content blocks (`{"type": "text", "text": ...}`, `{"type": "image_url", ...}`, `{"type": "input_audio", ...}`). These types are re-exported from the `openai` package as `LLMStandardMessage`, but callers should treat them as LLMContext's own types.

**Q: How are tools stored?**
The constructor (and `set_tools`) always calls `_normalize_and_validate_tools`. If you pass a plain Python list, it is wrapped into a `ToolsSchema(standard_tools=list)`. An empty `ToolsSchema` (no standard tools, no custom tools) is converted to `NOT_GIVEN`. The stored value is always `ToolsSchema | NotGiven`.

**Q: Does `get_messages` copy the list?**
`get_messages()` with no filter returns `self._messages` directly — the same list object. `get_messages(truncate_large_values=True)` returns a new list of `copy.deepcopy`-ed messages with binary blobs replaced by short placeholders. The design means: the copy happens only when truncation is requested; the no-truncation path is cheap but callers must not mutate what they receive.

**Q: What is `llm_specific_filter` for?**
Some providers need to slip in a `LLMSpecificMessage` (a wrapper carrying an opaque blob tagged with an LLM name). `get_messages(llm_specific_filter="openai")` keeps only standard messages plus `LLMSpecificMessage` objects whose `.llm` field matches `"openai"`, discarding the rest. Mismatches are logged as errors.

**Q: Where would you seed a system prompt?**
Pass `messages=[{"role": "system", "content": "You are ..."}]` to the constructor. To replace all messages later, call `context.set_messages(new_list)`. To append, call `context.add_message(msg)`.

**Predict-the-behavior questions:**

1. You call `LLMContext(messages=my_list)`. Then you call `context.add_message({"role": "user", "content": "hello"})`. Does `my_list` now have 2 elements? *Think about what the constructor assigns to `self._messages`.*

2. You call `context.get_messages(truncate_large_values=True)` on a context with a base64 image message, then immediately call `context.get_messages()`. Is the original image URL still intact?

3. You call `LLMContext(tools=[])`. Is `context.tools` a `ToolsSchema` or `NOT_GIVEN`?

## Why this matters

`LLMContext` is the bot's memory: every user utterance, every assistant reply, and every tool call lives in its message list. If the LLM "forgets" what the user said, the aggregator that writes to context is the first place to look. If the system prompt silently changes or leaks between calls, look at who called `set_messages` or mutated the list that was passed to the constructor. Knowing this object cold means you can inspect live conversation state, seed history for testing, or surgically inject a corrected turn without restarting the pipeline.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/processors/aggregators/llm_context.py`. Find the `LLMContext` constructor (`__init__`) and its `get_messages` method. Replace their bodies with `raise NotImplementedError("Stage 15")` but KEEP the signatures, all parameter defaults, docstrings, and decorators exactly as they are. Don't touch any other code. Then show me:
> - The full `__init__` signature and what fields it initializes
> - The `get_messages` signature including all parameters and defaults
> - The `ToolsSchema` import and what arguments it accepts

## Your tasks (in order)

- [ ] **Read and trace.** Read `src/pipecat/processors/aggregators/llm_context.py` top-to-bottom. Trace the flow: `__init__` → `_normalize_and_validate_tools` → `_tools`. Then trace `get_messages` → `_truncate_large_values_from_messages` → `_truncate_long_strings`. Understand every branch before touching anything.

- [ ] **State the contract.** Before writing any code, write (as a comment or on paper) answers to: What does `__init__` store in `_messages` when `messages=None`? What is the type of `_tools` after a plain list is passed? What does `get_messages()` return when `llm_specific_filter` is `None` and `truncate_large_values` is `False`?

- [ ] **Stub it out.** Replace the bodies of `__init__` and `get_messages` with `raise NotImplementedError("Stage 15")`. Confirm the test file imports without error: `uv run python -c "from pipecat.processors.aggregators.llm_context import LLMContext"`.

- [ ] **Implement `__init__` and run the happy-path test.**
  - Store messages: `self._messages = messages if messages else []`
  - Normalize tools: `self._tools = LLMContext._normalize_and_validate_tools(tools)`
  - Store tool choice: `self._tool_choice = tool_choice`
  - Run: `uv run pytest tests/test_llm_context.py::TestGetMessagesTruncateLargeValues::test_default_preserves_all_data`

- [ ] **Implement `get_messages` for text-only messages.**
  - Handle the `llm_specific_filter` branch (filter out non-matching `LLMSpecificMessage` objects and log on mismatch).
  - Handle `truncate_large_values=False`: just return the messages list as-is.
  - Run: `uv run pytest tests/test_llm_context.py::TestGetMessagesTruncateLargeValues::test_text_only_messages_unchanged`

- [ ] **Edge case: no-mutation.** The truncating path must call `_truncate_large_values_from_messages` (which deep-copies). Verify that after calling `get_messages(truncate_large_values=True)`, the original messages inside the context are unchanged.
  - Run: `uv run pytest tests/test_llm_context.py::TestGetMessagesTruncateLargeValues::test_does_not_mutate_original`

- [ ] **Run the full suite.** Once all three named tests pass, run the whole file: `uv run pytest tests/test_llm_context.py -v`. All 18 tests should be green.

- [ ] **Reflect.** Answer: (a) Why does the no-filter path return the internal list directly instead of always copying? (b) When would `NOT_GIVEN` as the stored `_tools` value matter to an LLM service adapter?

## Stuck? (paste into Claude)

> I'm reimplementing `LLMContext`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`LLMContext` is the single source of truth for a bot's conversation memory. Every debugging session that starts with "the LLM said the wrong thing" eventually leads here: you check what messages were in the context, in what order, and whether the system prompt was present. The copy-on-truncate design means inspection is cheap and safe for logging — call `get_messages(truncate_large_values=True)` to see the full conversation without blowing up your log file with megabytes of base64 audio. Understanding that the default path returns the live list (not a copy) explains why the aggregators that write to context (stages 16-17) use `add_message` and `set_messages` rather than mutating the list directly — the LLM service might be reading it concurrently.

## Next → 16-user-aggregator.md
