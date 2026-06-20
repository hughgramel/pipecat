# Stage 20 — LLMService function-call dispatch

## Where you are

```
LLM emits a function-call request
         │
         ▼
  run_function_calls()
  builds FunctionCallRunnerItem(s)
         │
         ▼
  ┌─────────────────────────────────────────────────────┐
  │          _run_function_call(runner_item)            │  ← YOU ARE HERE
  │                                                     │
  │  1. Re-resolve name in self._functions              │
  │     ├─ found by name  → use that handler            │
  │     ├─ catch-all (None key) → use catch-all         │
  │     ├─ was already the missing handler → reuse it   │
  │     └─ just unregistered → build missing handler    │
  │                                                     │
  │  2. broadcast_frame(FunctionCallInProgressFrame)    │
  │                                                     │
  │  3. await handler(FunctionCallParams)               │
  │       handler calls params.result_callback(result)  │
  │                                                     │
  │  4. result_callback →                               │
  │     broadcast_frame(FunctionCallResultFrame)        │
  └─────────────────────────────────────────────────────┘
         │
         ▼
  FunctionCallResultFrame flows back into context
  aggregator; LLM continues with the result
```

## How it works (orientation — answer these before you code)

**Q: What is function calling, and what is `_run_function_call`'s role?**

Function calling (also called "tool use") lets an LLM pause generation, request that a named Python function be executed with specific arguments, and then resume generation using the returned result. `_run_function_call` is the dispatcher: given a queued `FunctionCallRunnerItem` it finds the right handler, calls it, and routes the result back into the pipeline as a `FunctionCallResultFrame`.

**Q: What is the function registry and how does lookup work?**

`self._functions` is a `dict[str | None, FunctionCallRegistryItem]`. Keys are function names; the special key `None` is a catch-all handler that handles any name. At dispatch time `_run_function_call` checks:

1. Is `runner_item.function_name` a key in `self._functions`? Use it.
2. Is `None` a key (catch-all)? Use that.
3. Does `runner_item.registry_item.handler` already point at `_missing_function_call_handler` (the LLM hallucinated a tool that was never registered)? Reuse the runner item's own registry entry — no double-logging.
4. Otherwise the function was registered at queue time but unregistered before execution — log a warning and synthesize a `FunctionCallRegistryItem` that routes to `_missing_function_call_handler`.

**Q: What is `FunctionCallParams` and how is it used?**

`FunctionCallParams` is a dataclass containing everything a handler needs: `function_name`, `tool_call_id`, `arguments`, `llm` (the service), `pipeline_worker`, `context` (the `LLMContext`), and a `result_callback`. The handler is expected to call `await params.result_callback(result)` to deliver its answer — it does not return a value.

**Q: How does the result get back to the model?**

The closure `function_call_result_callback` inside `_run_function_call` calls `broadcast_frame(FunctionCallResultFrame, ...)`. That frame carries the function name, tool_call_id, arguments, and the result value. The LLM context aggregator sees it and appends a tool-result message so the next LLM call can continue.

**Q: Where does a tool get registered?**

Call `service.register_function("my_tool", my_async_handler)` before the pipeline runs. You can also use `FunctionSchema(handler=...)` inside an `LLMContext` or `LLMSetToolsFrame`; those direct functions are auto-registered when the context is set.

**Predict the behavior:**

1. The model calls `"book_flight"` but you forgot to call `register_function`. What happens?
   (Hint: trace through the four lookup steps above.)

2. You call `register_function("check_inventory", handler)`, the call is queued, then you call `unregister_function("check_inventory")` before execution runs. Does the original handler run or not?

3. A handler calls `params.result_callback` twice. What does the caller observe and why?
   (Hint: look at the timeout cancellation logic inside the closure.)

## Why this matters

Tools are how a voice agent does anything real — booking a reservation, querying a database, sending a message. Without a working dispatch loop the model's function requests silently die. Knowing this code means you can debug "the model called my function but nothing happened" (missing registration, wrong name), "I'm getting an unknown-tool warning" (hallucination vs. developer error, see `_log_missing_function_call`), or a crash in the exception handler at the bottom of `_run_function_call` that eats the error and emits a non-fatal `ErrorFrame`.

## Step 1 — Set up this stage (paste into Claude)

> Open `src/pipecat/services/llm_service.py`. Find `LLMService._run_function_call`. Replace its body with `raise NotImplementedError("Stage 20")` but KEEP the signature, docstring, and decorators exactly. Don't touch any other code. Then show me the signature, the function registry (`self._functions`), `FunctionCallParams`, and `FunctionCallResultFrame` so I know the contract.

## Your tasks (in order)

- [ ] **Read and trace.** Read `_run_function_call` in full (lines ~1344–1468). Then read `_missing_function_call_handler` and `_build_missing_function_call_registry_item`. Trace a call for a registered function and a missing function on paper before touching anything.

- [ ] **State the contract.** Write in your own words: what goes in (`FunctionCallRunnerItem` fields), what is looked up (`self._functions`), what is awaited (the handler via `FunctionCallParams`), and what frames come out (`FunctionCallInProgressFrame` then `FunctionCallResultFrame`). Include the error path.

- [ ] **Gut check.** With your implementation stubbed out, run the full test file to confirm the three target tests fail (not error):
  ```
  uv run pytest tests/test_llm_service.py::TestLLMService -x 2>&1 | head -30
  ```

- [ ] **Implement happy path.** Add lookup (name → `self._functions`), broadcast `FunctionCallInProgressFrame`, build `FunctionCallParams`, await the handler, and wire `result_callback` to broadcast `FunctionCallResultFrame`. Verify with:
  ```
  uv run pytest tests/test_llm_service.py::TestLLMService::test_catch_all_handler_suppresses_missing_warnings -v
  ```

- [ ] **Missing function emits a terminal result.** Add the branch that falls through to `_missing_function_call_handler` when the name is not registered. Run:
  ```
  uv run pytest tests/test_llm_service.py::TestLLMService::test_missing_function_call_emits_terminal_result -v
  ```

- [ ] **Function unregistered between queue and execute.** Add the branch that detects a name that was present at queue time but removed before execution (the `else` branch with the `"just unregistered"` warning). Run:
  ```
  uv run pytest tests/test_llm_service.py::TestLLMService::test_function_unregistered_between_queue_and_execute -v
  ```

- [ ] **Mute cleanup survives a missing call.** Confirm that `FunctionCallUserMuteStrategy` is left unmuted after a missing-function result (the result frame must be broadcast even for unknown tools). Run:
  ```
  uv run pytest tests/test_llm_service.py::TestLLMService::test_missing_function_call_allows_user_mute_cleanup -v
  ```

- [ ] **Reflect.** What would break in a real voice agent if the missing-function path raised an exception instead of returning a terminal result frame? Which frame would never arrive and why would the conversation stall?

## Stuck? (paste into Claude)

> I'm reimplementing `LLMService._run_function_call`. Here's my attempt: [code]. Don't give me the answer — ask me one question that points at what I'm missing.

## What you learned

`_run_function_call` is the bridge between the LLM's intent (call this function) and your Python code. In production debugging, most tool failures fall into three buckets: the function was never registered (missing-function path, check your `register_function` calls), the function was registered under a different name than the model was told (check the `FunctionSchema.name` you advertised), or the handler raised an exception that got swallowed by the `except` block and emitted a non-fatal `ErrorFrame` instead of a result (add logging inside the handler). Knowing where each of these surfaces in `_run_function_call` cuts diagnosis time dramatically.

## Next → 21-input-transport.md
