# WebLLM advanced: workers, structured output, vision, embeddings, resumable generation

## Web-worker engine (keep the UI responsive)

```ts
// worker.ts
import { WebWorkerMLCEngineHandler } from "@mlc-ai/web-llm";
const handler = new WebWorkerMLCEngineHandler();
self.onmessage = (msg: MessageEvent) => { handler.onmessage(msg); };
```

```ts
// main.ts
import { CreateWebWorkerMLCEngine } from "@mlc-ai/web-llm";
const engine = await CreateWebWorkerMLCEngine(
  new Worker(new URL("./worker.ts", import.meta.url), { type: "module" }),
  selectedModel,
  { initProgressCallback },
);
// Same MLCEngineInterface from here on — swap-in replacement.
```

⚠️ `logitProcessorRegistry` does not work through a worker (main-thread engine only).

## Service-worker engine (survive page reloads)

The engine runs **inside the service worker thread** — it is *not* an HTTP endpoint, and it needs **no COOP/COEP/SharedArrayBuffer headers**.

```ts
// sw.ts — MUST instantiate at top level of the script, not inside activate/message
// listeners: the browser can restart an active SW without re-firing activate.
import { ServiceWorkerMLCEngineHandler } from "@mlc-ai/web-llm";
new ServiceWorkerMLCEngineHandler();
```

```ts
// main.ts
import { CreateServiceWorkerMLCEngine } from "@mlc-ai/web-llm";
navigator.serviceWorker.register(new URL("sw.ts", import.meta.url), { type: "module" });
const engine = await CreateServiceWorkerMLCEngine(selectedModel, { initProgressCallback });
```

Gotchas: the SW can be killed at any time without notifying the page; WebLLM sends heartbeats to keep it alive (`keepAliveMs` / missed-heartbeat handling in `src/service_worker.ts`) — your app must still handle engine death/restart. Full example: `examples/service-worker` in the repo (also `examples/chrome-extension-webgpu-service-worker` for extension background pages).

## JSON mode (structured output)

Use `response_format` with `type: "json_object"`; the optional `schema` is enforced by XGrammar in WASM:

```ts
const reply = await engine.chat.completions.create({
  messages,
  response_format: {
    type: "json_object" as const,
    schema: {
      type: "object",
      properties: { name: { type: "string" }, age: { type: "number" } },
      required: ["name", "age"],
    },
  },
});
const parsed = JSON.parse(reply.choices[0].message.content);
```

- There is **no** OpenAI-style `{ type: "json_schema" }` — the schema lives inside `json_object` (source throws otherwise).
- Complex schemas can consume many tokens: watch `finish_reason: "length"` / `max_tokens` truncation.
- Playground to iterate on schemas: https://huggingface.co/spaces/mlc-ai/WebLLM-JSON-Playground

## Function calling (preliminary)

Only works with function-calling-capable models (see `functionCallingModelIds` in config — the Hermes-2/3 family: Hermes-2-Pro-Llama-3-8B, Hermes-2-Pro-Mistral-7B, Hermes-3-Llama-3.1-8B). `tool_choice` accepts `"auto"` or `"none"` only.

```ts
const reply = await engine.chat.completions.create({
  messages,
  tools: [{
    type: "function",
    function: {
      name: "get_current_weather",
      description: "Get weather for a city",
      parameters: { type: "object", properties: { city: { type: "string" } }, required: ["city"] },
    },
  }],
  tool_choice: "auto",
});
// tool call arrives in reply.choices[0].message.tool_calls — execute it, append the
// result as a { role: "tool" } message, and call create() again.
```

Advanced alternative that works with any model: structural tags (`examples/structural-tag-tool-use`).

## Vision (multimodal)

Prebuilt VLM: `Phi-3.5-vision-instruct-q4f16_1-MLC` (or q4f32_1). `model_type: "VLM"` records get image inputs:

```ts
const reply = await engine.chat.completions.create({
  messages: [{
    role: "user",
    content: [
      { type: "text", text: "List the items in each image concisely." },
      { type: "image_url", image_url: { url: base64DataUrl } },   // base64 data URLs
      { type: "image_url", image_url: { url: corsProxyUrl } },    // remote URLs need CORS
    ],
  }],
});
```

Constraints: exactly one text part; `image_url.detail` **throws**; remote images hit CORS (use base64 or a proxy). Gemma-3 vision families exist in the runtime but have no prebuilt entry — compile it yourself.

## Embeddings

Prebuilt embedding models (Snowflake Arctic Embed, `model_type: "embedding"`):
`snowflake-arctic-embed-m-q0f32-MLC-b32` / `-b4`, `snowflake-arctic-embed-s-q0f32-MLC-b32` / `-b4` (b32/b4 = batch-size variants).

```ts
const reply = await engine.embeddings.create({ input: ["hello world", "webgpu"], model: modelId });
const vectors = reply.data.map((d) => d.embedding);   // + reply.usage
```

Nuance: unlike chat, `embeddings.create` **does** read `model` — it auto-`reload()`s if the requested model differs from the loaded one. The repo's `examples/embeddings` ships a LangChain.js-compatible `WebLLMEmbeddings` adapter for RAG with one engine doing both LLM + embeddings.

## Resumable generation (opt-in, new)

Persist a streaming generation so it survives reloads/crashes — journal + optional KV checkpoints go to same-origin **OPFS**:

```ts
const chunks = await engine.chat.completions.create({
  messages, stream: true,          // streaming + text-only only
  // @ts-expect-error extra_body extension
  extra_body: { resumable: { enabled: true, sessionId: crypto.randomUUID(), strictPersistence: true } },
});
// later / after crash:
const sessions = await engine.listResumableSessions();
const recovery = await engine.resumeChatCompletion(sessionId);           // read-only
const continued = await engine.resumeChatCompletion(sessionId, { continueGeneration: true }); // resume streaming
await engine.deleteResumableSession(sessionId);
```

Restrictions: no images, no grammar/JSON constraints, no custom logit processors, `completions` endpoint unsupported. KV-checkpoint recovery needs checkpoint-capable model libs; otherwise it token-replays. Durability via `durabilityMode: "exact" | "relaxed"`. Docs: https://webllm.mlc.ai/docs/user/resumable_generation.html

## GPU capability probing

```ts
const maxSize = await engine.getMaxStorageBufferBindingSize();  // compare vs ModelRecord.buffer_size_required_bytes
const vendor  = await engine.getGPUVendor();                   // "apple" | "nvidia" | "qualcomm" | "arm" | ""
```

- Models listing `required_features: ["shader-f16"]` need the f16 shader feature (common for q4f16_1 builds) — check `navigator.gpu.requestAdapter()` features or WebGPU report.
- `examples/subgroups-usage` demonstrates capability-based routing between WASM builds.
