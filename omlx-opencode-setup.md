# Local AI Coding Stack — OpenCode + oMLX (Apple Silicon)

A working, tuned setup for agentic coding with local models. Tuned on a
**64 GB Apple Silicon Mac** — scale the models down if you have less RAM.

## Components
- **oMLX** (menu-bar Mac app + `omlx-server` python child) — MLX inference
  server with continuous batching and a paged SSD KV cache. Serves an
  OpenAI-compatible API at `http://127.0.0.1:8000/v1` (and an
  Anthropic-compatible `/v1/messages`, so it can back Claude Code too).
  Settings: `~/.omlx/settings.json` · logs: `~/.omlx/logs/server.log` ·
  admin API: `/admin/api/`.
- **OpenCode** CLI — the agent (config: `opencode.jsonc` in this repo →
  `~/.config/opencode/opencode.jsonc`).

## Why oMLX for agentic coding
- **Paged SSD KV cache** — coding agents resend a huge, mostly-identical
  prompt every turn (OpenCode's system prompt alone is ~9k tokens with a full
  tool set). oMLX restores matching prefixes from disk instead of recomputing,
  so warm-turn TTFT drops from tens of seconds to seconds; the cache even
  survives server restarts. This also makes thinking-heavy models usable
  (see Qwen3.8 below).
- **Continuous batching** — agents fire parallel requests (title-gen + main
  turn); oMLX batches them into single GPU passes instead of queueing.
- **Memory guard** — refuses to load a model that would exceed a dynamic RAM
  ceiling instead of swap-thrashing the machine into a reboot.

## Models
oMLX auto-discovers MLX-format models from the folders in
`model.model_dirs` (e.g. `~/MLXModels`, two-level `org/model-name/` layout).
Model IDs are the **case-sensitive folder names**.

On 64 GB, this combination works well:
- **Daily driver: `Qwen3.6-35B-A3B-OptiQ-4bit`** (~18 GB) — MoE (only ~3B
  active per token → fast), strong tool-calling. Measured here: cold load
  3.4 s, ~38 tok/s decode.
- **Second opinion: `Qwen3.8-27B-MXFP8`** (~27 GB, dense 8-bit) — higher
  quality, slower per token. Its chat template defaults thinking ON at high
  effort *and* preserves `<think>` blocks in conversation history, so prompts
  snowball across turns — fine under oMLX because the SSD prefix cache
  absorbs the re-prefill, painful on servers without prefix caching.
- General/vision models (LFM2, Gemma) load fine but are weak at proactive
  tool use — poor as autonomous coding agents.

Rule of thumb: only one big model fits under the memory guard at a time next
to normal desktop apps. Unload before switching (see Gotchas).

## OpenCode setup
1. Put `opencode.jsonc` from this repo at `~/.config/opencode/opencode.jsonc`.
2. Auth: this setup runs localhost-only with
   `auth.skip_api_key_verification: true`, so no key is needed. If you keep
   verification on, add the key via `opencode auth login` → omlx provider.
3. Verify connectivity: `curl http://127.0.0.1:8000/v1/models` should list
   your model folders' names.

## The tuned parameters (and why)
| Setting | Value | Why |
|---|---|---|
| default model | `Qwen3.6-35B-A3B-OptiQ-4bit` | fits resident + fast + real tool use |
| `temperature` | **0.2** | low entropy → fewer runaway/degenerate generations. **Set in the model `options`, NOT the agent** — OpenCode drops agent-level temperature for openai-compatible providers (verified on the wire). |
| `top_p` | 0.9 | same |
| `max_tokens` (+ `limit.output`) | **2048** | bounds a runaway generation |
| `limit.context` | **32768** | matches oMLX `sampling.max_context_window` |
| `timeout` / `headerTimeout` / `chunkTimeout` | **120000 / 60000 / 25000** | a stalled stream becomes a resendable error instead of an infinite TUI freeze; headerTimeout stays generous for cold multi-k-token prefills |

## Gotchas
- **The UI's "disable API key" toggle may not persist.** If the server keeps
  returning 401 after you disable auth, set
  `auth.skip_api_key_verification: true` in `~/.omlx/settings.json` directly
  and restart the server process.
- **Memory guard math**: ceiling ≈ free + inactive + 50% of active RAM,
  hard-capped by the kernel Metal limit (`iogpu.wired_limit_mb`; ~51.8 GB on
  a 64 GB Mac). On "would exceed the dynamic memory ceiling", unload the
  other model:
  `curl -X POST http://127.0.0.1:8000/admin/api/models/<id>/unload`
  (or use the dashboard). You *can* raise the kernel cap
  (`sudo sysctl iogpu.wired_limit_mb=...`) to co-load two big models, but
  wired memory can't be reclaimed by the OS — leave several GB of headroom
  (≤ ~55 GB on 64 GB), and note it resets on reboot.
- The port in `settings.json` is authoritative (default 8000). A stray second
  listener on another port means a leftover server instance — restart the app.
- Quantized repacks from third-party HF orgs may bundle engine-specific
  sidecar files (MTP/speculative-decoding tuning, serving presets). oMLX
  ignores them — behavior comes from the standard `config.json`,
  `generation_config.json`, and chat template. Don't expect speculative-
  decoding speedups advertised for other engines to apply.
