# Setup and model configuration

A working, tuned setup for agentic coding with local models on a **64 GB Apple
Silicon Mac**. Scale the model lineup down if you have less RAM. The raw
measurements behind the limits are preserved in the
[2026-08-30 context benchmark](benchmarks/2026-08-30-context.md).

## Components

- **oMLX** (menu-bar Mac app + `omlx-server` Python child) — MLX inference
  server with continuous batching and a paged SSD KV cache. Serves an
  OpenAI-compatible API at `http://127.0.0.1:8000/v1` and an
  Anthropic-compatible `/v1/messages` endpoint.
  Settings: `~/.omlx/settings.json` · logs: `~/.omlx/logs/server.log` ·
  admin API: `/admin/api/`.
- **OpenCode** CLI — the agent. Copy `opencode.jsonc` from the repository to
  `~/.config/opencode/opencode.jsonc`.

## Why oMLX for agentic coding

- **Paged SSD KV cache** — coding agents resend a large, mostly identical
  prompt every turn (OpenCode's system prompt alone is about 9K tokens with a
  full tool set). oMLX restores matching prefixes from disk instead of
  recomputing them, so warm-turn TTFT drops from tens of seconds to seconds;
  the cache also survives server restarts.
- **Continuous batching** — agents can issue parallel requests such as title
  generation and the main turn; oMLX batches them into GPU passes instead of
  simply queueing them.
- **Memory guard** — refuses an allocation that would exceed a dynamic RAM
  ceiling instead of swap-thrashing the machine into a reboot.

## Model lineup

oMLX auto-discovers MLX-format models from folders in `model.model_dirs`
(for example `~/MLXModels`, using a two-level `org/model-name/` layout). Model
IDs are the **case-sensitive folder names**.

The current 64 GB lineup uses oMLX-native Jundot oQ quants throughout:

- **Daily driver: `Qwen3.6-35B-A3B-oQ4e-MTP`** (~20 GB) — fast MoE with
  strong tool-calling. Lightning MTP measured 85.8 tok/s, versus 70.8 without
  MTP, at 85% draft acceptance.
- **Coding second opinion: `Qwen3.8-27B-oQ6e-MTP`** (~22 GB) — dense 6-bit
  mixed quant at near-8-bit quality. Lightning MTP measures about 27.6 tok/s.
  Its chat template defaults thinking ON and preserves think blocks in
  history; the SSD prefix cache keeps repeated turns practical.
- **General/vision: `gemma-4-31B-oQ4e-MTP`** (~19 GB) — dense, multimodal,
  and roughly 21–27 tok/s with MTP. It replaced the 26B-A4B Gemma and the
  24B-A2B LFM: materially better instruction following at a smaller footprint.
- **Small/fast/vision: `gemma-4-E4B-it-oQ4e-mtp`** (~5.2 GB) — compact dense
  multimodal model with four MTP heads, measured around 89 tok/s. It replaced
  the 8B-A1B LFM despite that model's ~120 tok/s decode because the LFM leaked
  meta-reasoning and repeatedly ignored exact-output instructions.

The two Qwen models total about 42 GB and can co-reside when desktop memory
pressure is low. oMLX may still evict one under its dynamic soft ceiling; that
is expected and safer than forcing the machine into swap pressure.

## OpenCode setup

1. Copy `opencode.jsonc` from this repository to
   `~/.config/opencode/opencode.jsonc`.
2. This setup runs localhost-only with
   `auth.skip_api_key_verification: true`, so no key is needed. If you retain
   verification, add the key with `opencode auth login` and select the oMLX
   provider.
3. Verify connectivity with
   `curl http://127.0.0.1:8000/v1/models`; it should list the model folder
   names.

## Tuned parameters

| Setting | Value | Why |
|---|---|---|
| default model | `Qwen3.6-35B-A3B-oQ4e-MTP` | fast MoE + native MTP + reliable tool use |
| small model | `gemma-4-E4B-it-oQ4e-mtp` | routes lightweight OpenCode work such as title generation to the ~5.2 GB, ~89 tok/s model |
| `temperature` | **0.2** | low entropy reduces runaway or degenerate generations; set in model `options` because OpenCode drops agent-level temperature for openai-compatible providers (verified on the wire) |
| `top_p` | **0.9** | keeps sampling controlled for agentic use |
| `max_tokens` + `limit.output` | **2048** | bounds a runaway generation |
| `limit.context` | **65536 / 32768 / 32768 / 65536** | OpenCode comfort budgets for Qwen3.6, Qwen3.8, Gemma 31B and Gemma E4B; they trigger compaction before multi-minute fresh prefills while oMLX retains higher hard ceilings |
| `compaction` | **auto + prune + 8192 reserved** | compacts with 8K tokens of headroom and removes stale bulky tool output from the active prompt while durable session history remains available |
| `timeout` / `headerTimeout` / `chunkTimeout` | **300000 / 120000 / 90000** | turns a stalled stream into a bounded, resendable error while allowing cold loads and dense-model prefills |

The model-specific context decisions are documented in the
[benchmark decision table](benchmarks/2026-08-30-context.md#decisions-derived-from-the-benchmark).
For memory management and failure recovery, see
[Operations and troubleshooting](operations.md).
