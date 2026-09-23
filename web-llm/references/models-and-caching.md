# WebLLM models, VRAM, and caching

For `@mlc-ai/web-llm@^0.2.85`. The **source of truth for prebuilt models is `webllm.prebuiltAppConfig.model_list`** (mirrors `src/config.ts` on GitHub) — the README's model table is stale. Model libs and weights live on GitHub/Hugging Face and update independently of the npm package.

## Choosing a model

A `ModelRecord` has: `model` (HF weights URL), `model_id`, `model_lib` (wasm URL), `overrides` (e.g. `context_window_size`), `vram_required_MB`, `low_resource_required`, `buffer_size_required_bytes`, `required_features` (e.g. `["shader-f16"]`), `model_type` (`LLM` | `embedding` | `VLM`), `integrity` (optional SRI hashes).

List/filter models at runtime instead of hard-coding IDs:

```ts
import { prebuiltAppConfig } from "@mlc-ai/web-llm";

const small = prebuiltAppConfig.model_list
  .filter((m) => m.low_resource_required)
  .map((m) => ({ id: m.model_id, vramMB: m.vram_required_MB }));
```

### Quantization suffixes

| Suffix | Meaning | Notes |
|---|---|---|
| `q4f16_1` | 4-bit weights, f16 activations | smallest VRAM (≈0.75–0.85× q4f32_1); some models need the `shader-f16` GPU feature |
| `q4f32_1` | 4-bit weights, f32 activations | widest GPU compatibility |
| `q0f16` / `q0f32` | unquantized weights | fidelity at large VRAM cost |
| `q3f16_1` | 3-bit | big models (e.g. Llama-3.1-70B) |

`-1k` variants trim the context window to 1024 tokens to cut KV-cache memory.

### VRAM samples (from `prebuiltAppConfig`)

| Model (q4f16_1 unless noted) | VRAM |
|---|---|
| SmolLM2-135M (q0f16) | 360 MB |
| SmolLM2-360M | 376 MB |
| gemma3-1b-it | 711 MB |
| Llama-3.2-1B-Instruct | 879 MB |
| Qwen2.5-0.5B-Instruct | 945 MB |
| Qwen3-0.6B | 1.4 GB |
| SmolLM2-1.7B | 1.77 GB |
| gemma-2-2b-it | 1.9 GB |
| Qwen3-1.7B | 2.0 GB |
| Llama-3.2-3B-Instruct | 2.3 GB |
| Qwen2.5-3B-Instruct | 2.5 GB |
| Qwen3-4B / Phi-4-mini | ~3.4 GB |
| Phi-3.5-vision-instruct | 3.95 GB |
| Llama-3.1-8B-Instruct (4k ctx) | 5.0 GB |
| Qwen3-8B | 5.7 GB |

Families in the prebuilt list include Llama 3.2/3.1/3/2, Qwen3.5/Qwen3/Qwen2.5 (incl. Coder/Math), Phi-4-mini/Phi-3.5 (incl. vision)/Phi-3, Gemma-3/Gemma-2, SmolLM2, Mistral-7B, DeepSeek-R1-Distill (7B/8B), Hermes-2/3, OLMo-2, Ministral-3, TinyLlama, StableLM-2, plus Snowflake Arctic embedding models. **Low-resource picks** (flagged `low_resource_required`): SmolLM2-135M/360M, gemma3-1b, Llama-3.2-1B, Qwen2.5-0.5B, Qwen3-0.6B/1.7B, and `-1k` variants.

### Gotchas by family

- Qwen3.5 models are hybrid/recurrent — history is bounded by `max_history_size`, not context window semantics you may expect.
- Some q4f16_1 models (Mistral, Gemma, SmolLM2) require the `shader-f16` GPU feature.
- Models above ~4–6 GB are impractical for most users' GPUs; the 70B q3f16_1 build exists but needs a workstation GPU.

## Custom models / AppConfig

Provide your own model list via `appConfig` — weights from any `https://huggingface.co/{user}/{model}` MLC-compiled repo:

```ts
const myAppConfig = {
  model_list: [
    {
      model: "https://huggingface.co/mlc-ai/Llama-3.2-1B-Instruct-q4f16_1-MLC",
      model_id: "my-llama",
      model_lib: tempModelLibUrl,     // wasm lib URL for the model's version/arch
      vram_required_MB: 879.04,
      overrides: { context_window_size: 4096 },
    },
  ],
};
const engine = await CreateMLCEngine("my-llama", { appConfig: myAppConfig });
```

To compile new models to MLC format, follow the MLC LLM WebLLM deployment docs (https://llm.mlc.ai/docs/deploy/webllm.html) and `webllm.mlc.ai/docs/developer/add_models.html`. Optional `integrity` field (SRI `sha256-<base64>` hashes for config/wasm/tokenizer, `onFailure: "error" | "warn"`) throws `IntegrityError` on mismatch.

## Caching

- Default backend: **Cache API** (best tested). Alternatives via `AppConfig.cacheBackend`: `"indexeddb"`, `"opfs"` (+ `opfsAccessMode: "async" | "sync" | "auto"`), experimental `"cross-origin"` (needs a companion Chrome extension; falls back to default).
- Cache is origin-scoped and subject to browser storage quotas/eviction. First run downloads multi-GB — always show `initProgressCallback` in the UI.
- Cache utilities (exported from the package root):

```ts
import {
  hasModelInCache, deleteModelAllInfoInCache, deleteModelWasmInCache,
  deleteChatConfigInCache, deleteModelInCache,
} from "@mlc-ai/web-llm";

if (await hasModelInCache("Llama-3.2-1B-Instruct-q4f16_1-MLC")) { /* warm start */ }
```

- `examples/simple-chat-upload` in the repo shows loading model files from local disk instead of downloading.
