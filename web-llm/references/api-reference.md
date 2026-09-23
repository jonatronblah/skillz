# WebLLM API reference

For `@mlc-ai/web-llm@^0.2.85`. Compiled from the official docs plus `src/types.ts` and `src/config.ts`.

## Instantiation

```ts
import { CreateMLCEngine, MLCEngine } from "@mlc-ai/web-llm";

// One-shot (async): downloads + loads the model
const engine = await CreateMLCEngine(modelId, engineConfig?);

// Split form: sync constructor, async load — handy for UI wiring
const engine = new MLCEngine({ initProgressCallback });
await engine.reload(modelId);   // long-running

// Swap or load models later; per-model chat options optional
await engine.reload("Llama-3.1-8B-Instruct-q4f32_1-MLC", { temperature: 0.7 });
// Multi-model: arrays for ids and (optional) chat options
await engine.reload(["Llama-3.1-8B-Instruct-q4f32_1-MLC", "gemma-2-2b-it-q4f16_1-MLC"],
                    [{ temperature: 0.7 }, { top_p: 0.9 }]);
```

**MLCEngineConfig** (all optional): `appConfig` (custom model config), `initProgressCallback` (progress: `{ progress, timeElapsed, text }`), `logitProcessorRegistry` (main-thread engine only), `logLevel` (`"TRACE"|"DEBUG"|"INFO"|"WARN"|"ERROR"|"SILENT"`, default `"WARN"`).

## Engine interface (`MLCEngineInterface`)

| Method | Notes |
|---|---|
| `reload(modelId \| modelIds, chatOpts?)` | Load one/many models sequentially. Throws on device lost (usually OOM) — retry with a smaller model or `context_window_size`. |
| `unload()` | Unload models, wait for the GPU device to finish, destroy it. |
| `interruptGenerate()` | Abort ongoing generation. |
| `resetChat(keepStats?, modelId?)` | Clear chat state. |
| `getMessage(modelId?)` | Latest generated message as a string. |
| `runtimeStatsText(modelId?)` | Prefill/decode throughput stats. |
| `chat.completions.create(req)` | OpenAI-style: non-streaming → `ChatCompletion`; `stream: true` → `AsyncIterable<ChatCompletionChunk>`. |
| `completions.create(req)` | Plain **text completion** (no chat template). |
| `embeddings.create(req)` | OpenAI-style embeddings (see `advanced.md`). |
| `setAppConfig` / `setInitProgressCallback` / `getInitProgressCallback` | Runtime config setters. |
| `setLogLevel(level)` | Runtime log level. |
| `getMaxStorageBufferBindingSize()` | Probe device limits (compare vs model's `buffer_size_required_bytes`). |
| `getGPUVendor()` | e.g. `"apple"`, `"nvidia"`, `"qualcomm"`; `""` if unknown. |
| `forwardTokensAndSample(inputIds, isPrefill, modelId?)` | Low-level token loop (custom logit processors). |
| `listResumableSessions()` / `resumeChatCompletion(...)` / `deleteResumableSession(id)` | Resumable generation (see `advanced.md`). |

With more than one model loaded, pass the `modelId` argument to utility methods. Requests to the same model serialize.

## Chat completion request

```ts
await engine.chat.completions.create({
  messages,                       // you own the history
  temperature, top_p, max_tokens, stop,
  frequency_penalty, presence_penalty, repetition_penalty,
  seed, n, logit_bias, logprobs, top_logprobs,
  response_format,                // JSON mode (see advanced.md)
  stream, stream_options,         // stream_options: { include_usage: true }
  // extra_body-style MLC extensions:
  // enable_thinking (thinking models), enable_latency_breakdown
});
```

**Parameter bounds (validated in source):**

| Param | Constraint |
|---|---|
| `temperature` | ≥ 0 |
| `top_p` | 0 < t ≤ 1 |
| `max_tokens` | > 0 |
| `frequency_penalty` / `presence_penalty` | [-2, 2] (OpenAI bounds; setting one defaults the other to 0 with a warning) |
| `repetition_penalty` | > 0 (MLC-only extension) |
| `logit_bias` | keys = numeric token-id strings; values ∈ [-100, 100]; -100 ≈ ban |
| `logprobs` / `top_logprobs` | `top_logprobs` ∈ [0, 5], requires `logprobs: true` |
| `ignore_eos`, `n`, `seed`, `stop` | supported |

Omitted values fall back to the model's `ChatConfig` (`mlc-chat-config.json`), which you can override at load time:

```ts
await engine.reload(modelId, {
  temperature: 0.7,
  repetition_penalty: 1.1,
  context_window_size: 4096,
});
```

## OpenAI-compatibility surface

**Supported:** chat completions (streaming + `stream_options.include_usage`), text completions, embeddings, JSON mode with schema (XGrammar), `seed`, `logprobs`/`top_logprobs`, `logit_bias`, `frequency_penalty`/`presence_penalty`, vision `image_url` content parts.

**Not supported / diverges from OpenAI:**

| Feature | Reality |
|---|---|
| `model` param in `create()` | Ignored — model fixed at engine creation/reload. (Exception: `embeddings.create` reads `model` and auto-reloads if it doesn't match the loaded model.) |
| `response_format: { type: "json_schema" }` | Doesn't exist. Use `{ type: "json_object", schema }` — schema only valid with `json_object`. |
| `tools` / `tool_choice` | Preliminary; Hermes-function-calling models only. `tool_choice` accepts `"auto"` or `"none"` (string) — anything else throws. |
| `image_url.detail` | Throws if set. |
| Streaming usage | Only on the last chunk, and only with `stream_options: { include_usage: true }`. |

## Text completion (non-chat)

```ts
const reply = await engine.completions.create({ prompt: "Once upon a time", max_tokens: 64 });
```

Plain text in/out — no chat template applied. (`stop`, sampling params as above.)
