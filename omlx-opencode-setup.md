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
1. Put `opencode.jsonc` from this repo at `~/.config/opencode/opencode.jsonc`.
2. Auth: this setup runs localhost-only with
   `auth.skip_api_key_verification: true`, so no key is needed. If you keep
   verification on, add the key via `opencode auth login` → omlx provider.
3. Verify connectivity: `curl http://127.0.0.1:8000/v1/models` should list
   your model folders' names.

## The tuned parameters (and why)
| Setting | Value | Why |
|---|---|---|
| default model | `Qwen3.6-35B-A3B-oQ4e-MTP` | fast MoE + native MTP + reliable tool use |
| `temperature` | **0.2** | low entropy → fewer runaway/degenerate generations. **Set in the model `options`, NOT the agent** — OpenCode drops agent-level temperature for openai-compatible providers (verified on the wire). |
| `top_p` | 0.9 | same |
| `max_tokens` (+ `limit.output`) | **2048** | bounds a runaway generation |
| `limit.context` | **131072 / 65536 / 32768 / 131072** | per-model client budgets for Qwen3.6, Qwen3.8, Gemma 31B, and Gemma E4B respectively |
| `timeout` / `headerTimeout` / `chunkTimeout` | **300000 / 120000 / 90000** | a stalled stream becomes a resendable error instead of an infinite TUI freeze; still allows cold loads and dense-model prefills |

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
- Native MTP is persisted per model. Enable it once with
  `PUT /admin/api/models/<id>/settings` and
  `{"mtp_enabled":true,"mtp_num_draft_tokens":4}`; verify the log says
  `Lightning MTP` rather than assuming a checkpoint name activates it.
