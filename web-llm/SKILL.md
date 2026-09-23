---
name: web-llm
description: Use WebLLM (@mlc-ai/web-llm) to run LLMs entirely in the browser with WebGPU acceleration — no server needed. OpenAI-compatible chat completions, streaming, JSON mode, function calling, vision, embeddings, web-worker and service-worker engines, cached model weights for offline reuse. Use for on-device / in-browser LLM features in JavaScript/TypeScript. WHEN: web-llm, WebLLM, MLC, run LLM in browser, in-browser inference, WebGPU LLM, client-side chatbot, on-device AI JavaScript, offline LLM.
---

# WebLLM

[WebLLM](https://webllm.mlc.ai/docs/) (`@mlc-ai/web-llm`) is a high-performance in-browser LLM inference engine: everything runs client-side, accelerated by **WebGPU**, with a fully **OpenAI-compatible** API surface (`engine.chat.completions.create`, `engine.embeddings.create`). Apache-2.0, ESM-native, one tiny runtime dependency. It is the browser companion of MLC LLM — weights come from `mlc-ai/*-MLC` repos on Hugging Face, compiled model executables from MLC's binary-libs repo.

**Pin or newer:** `@mlc-ai/web-llm@^0.2.85`. Note: model libraries version independently of the npm package (via `modelVersion` in the config) — a newer package may fetch new model libs.

## Setup

```bash
npm install @mlc-ai/web-llm
```

```ts
import * as webllm from "@mlc-ai/web-llm";
// or
import { CreateMLCEngine } from "@mlc-ai/web-llm";
```

CDN (officially supported, no bundler needed — works on JSFiddle/CodePen):

```js
import * as webllm from "https://esm.run/@mlc-ai/web-llm";
// or dynamically:
const webllm = await import("https://esm.run/@mlc-ai/web-llm");
```

## Quick start

```ts
import { CreateMLCEngine } from "@mlc-ai/web-llm";

const engine = await CreateMLCEngine(
  "Llama-3.1-8B-Instruct-q4f32_1-MLC",
  {
    initProgressCallback: (p) => console.log(p.text),  // { progress, timeElapsed, text }
  },
);

const messages = [
  { role: "system", content: "You are a helpful AI assistant." },
  { role: "user", content: "Hello!" },
];
const reply = await engine.chat.completions.create({ messages });
console.log(reply.choices[0].message);
console.log(reply.usage);
```

Streaming:

```ts
const chunks = await engine.chat.completions.create({
  messages,
  temperature: 1,
  stream: true,
  stream_options: { include_usage: true },  // usage arrives only on the LAST chunk
});
let text = "";
for await (const chunk of chunks) {
  text += chunk.choices[0]?.delta?.content || "";
  if (chunk.usage) console.log(chunk.usage);
}
```

## Mental model

1. `CreateMLCEngine(modelId, config)` downloads the model (multi-GB on first run — wire up `initProgressCallback` for UX), then keeps it in the browser **cache** for near-instant reloads later.
2. All interaction is OpenAI-style: `engine.chat.completions.create({ messages, ... })`. **You maintain the `messages` array** across turns — multi-round chat is your job (WebLLM reuses the KV cache internally when it detects multi-round chat, but the history is yours to track).
3. Swap models with `await engine.reload(otherModelId)`; load several with an array. Requests to the same model serialize.
4. Everything is client-side: WebGPU is a hard requirement, and output quality/feasibility is bounded by the user's VRAM (see `references/models-and-caching.md`).

## Browser requirements

- **WebGPU**: Chrome/Edge 113+ (desktop), Chrome 121+ (Android 12+, Qualcomm/ARM GPUs); Firefox 141+ (Windows), 145+ (macOS Apple Silicon), 147+ (all macOS); Safari 26 (macOS/iOS). Linux depends on GPU/driver (recent Chrome, Intel Gen12+/NVIDIA 535+).
- **HTTPS / secure context** is required.
- Feature-detect with `if (!("gpu" in navigator)) …`; point users at https://webgpureport.org/ to verify their setup.

## Gotchas

- **The `model` field in `create()` is ignored** for chat — the model is fixed by `CreateMLCEngine(modelId)` / `engine.reload(modelId)`.
- The README's "built-in models" list is **stale**. The source of truth is `webllm.prebuiltAppConfig.model_list` (or `src/config.ts` on GitHub). List/filter it at runtime rather than hard-coding IDs.
- `device lost` errors are usually **VRAM OOM** → `reload()` a smaller model or a smaller `context_window_size` (the `-1k` model variants trim context to 1024 tokens).
- JSON mode uses `response_format: { type: "json_object", schema }` — the schema goes **inside** `json_object`; there is no OpenAI-style `"json_schema"` type.
- Function calling (`tools`/`tool_choice`) is preliminary and only works with specific Hermes models; `tool_choice` accepts only `"auto"` or `"none"`.
- Vision messages: `image_url.detail` is **not supported** (it throws). Remote image URLs hit CORS — prefer base64 data URLs.
- `logitProcessorRegistry` works on the main-thread `MLCEngine` only, not worker engines.
- No COOP/COEP/SharedArrayBuffer headers are needed anywhere — WebLLM is WebGPU + WASM + Cache API. (The service-worker mode is *not* an OpenAI-compatible HTTP endpoint; it keeps the engine alive across page reloads.)
- There is no CHANGELOG.md — version history lives at GitHub releases.

## References

Load on demand — do not read these up front:

| File | Read when… |
|---|---|
| `references/api-reference.md` | You need the full engine API, chat/completion request parameters and their validation bounds, or the exact OpenAI-compatibility surface |
| `references/models-and-caching.md` | Choosing a model (VRAM/quantization/low-resource picks), listing models at runtime, custom models, or cache management |
| `references/advanced.md` | Web-worker or service-worker engines, JSON mode, function calling, vision, embeddings, resumable generation, GPU capability probing |

Official docs: [get started](https://webllm.mlc.ai/docs/user/get_started.html) · [basic usage](https://webllm.mlc.ai/docs/user/basic_usage.html) · [advanced](https://webllm.mlc.ai/docs/user/advanced_usage.html) · [API reference](https://webllm.mlc.ai/docs/user/api_reference.html) · [examples](https://github.com/mlc-ai/web-llm/tree/main/examples) · [releases](https://github.com/mlc-ai/web-llm/releases)
