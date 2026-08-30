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

## Context benchmark evidence (2026-08-30)

These measurements are the evidence behind the separate oMLX hard ceilings
and OpenCode comfort limits below. They were run from oMLX's benchmark UI on a
64 GB Apple Silicon Mac with oMLX 0.6.4, the Auto engine, Code (Python)
prompts, native MTP enabled at draft depth 4, and a fixed 128-token generation.
Prompt sizes are the benchmark's `pp` token counts. TTFT therefore reflects a
cold full-prompt prefill; cached incremental OpenCode turns can be much faster.

The kernel Metal wired limit had been raised for this session with
`sudo sysctl iogpu.wired_limit_mb=59392`, giving oMLX a 58.0 GiB Metal ceiling.
That change is temporary and resets at reboot. Memory guard was Balanced, so
its dynamic ceiling also depended on foreground-app memory pressure. A dash
means the value was not retained, not that it was zero.

| Model | Prompt | Result | TTFT | TPOT | Prefill | Decode | E2E | Peak memory |
|---|---:|---|---:|---:|---:|---:|---:|---:|
| Qwen3.6 35B-A3B | 32,768 | Pass | 18.56 s | — | 1,765.4 tok/s | — | — | — |
| Qwen3.6 35B-A3B | 65,536 | Pass | 52.87 s | — | 1,239.6 tok/s | — | — | — |
| Qwen3.6 35B-A3B | 131,072 | Pass | 177.87 s | — | 736.9 tok/s | — | — | — |
| Qwen3.6 35B-A3B | 200,000 | Pass | 420.57 s | — | 475.5 tok/s | — | 425.14 s | — |
| Qwen3.8 27B | 1,024 | Pass | 2.395 s | 38.54 ms/tok | 427.6 tok/s | 26.1 tok/s | 7.320 s | 26.55 GB |
| Qwen3.8 27B | 4,096 | Pass | 9.068 s | 37.59 ms/tok | 451.7 tok/s | 26.8 tok/s | 13.875 s | 28.04 GB |
| Qwen3.8 27B | 32,768 | Pass | 89.632 s | 37.18 ms/tok | 365.6 tok/s | 27.1 tok/s | 94.390 s | 32.56 GB |
| Qwen3.8 27B | 65,536 | Pass | 234.804 s | 54.20 ms/tok | 279.1 tok/s | 18.6 tok/s | 241.768 s | 37.93 GB |
| Qwen3.8 27B | 131,072 | Guard failure at 104,448 | — | — | — | — | — | — |
| Gemma 4 31B | 32,768 | Pass | 107.765 s | — | 304.1 tok/s | — | 116.153 s | — |
| Gemma 4 31B | 65,536 | Initial preflight rejection | — | — | — | — | — | — |
| Gemma 4 31B | 65,536 | Pass after closing apps | 264.871 s | — | 247.4 tok/s | — | 277.582 s | — |
| Gemma 4 E4B | 32,768 | Pass | 9.75 s | — | 3,360.1 tok/s | — | — | — |
| Gemma 4 E4B | 65,536 | Pass | 25.25 s | — | 2,596.0 tok/s | — | — | — |
| Gemma 4 E4B | 131,072 | Pass | 66.66 s | — | 1,966.3 tok/s | — | — | — |

The retained Qwen3.8 continuous-batching run used `pp1024/tg128`:

| Batch | Aggregate decode | Speedup | Aggregate prefill | Prefill/request | Average TTFT | E2E |
|---:|---:|---:|---:|---:|---:|---:|
| 1× | 26.1 tok/s | 1.00× | 427.6 tok/s | 427.6 tok/s | 2.395 s | 7.320 s |
| 2× | 24.0 tok/s | 0.92× | 315.7 tok/s | 157.8 tok/s | 4.847 s | 17.145 s |
| 4× | 44.0 tok/s | 1.69× | 277.5 tok/s | 69.4 tok/s | 9.051 s | 26.406 s |

This shows why the dense model feels responsive at ordinary prompt sizes but
degrades sharply on a cold long-context prefill. The 4× run improves aggregate
decode throughput, not individual-request latency. Qwen3.6's public benchmark
upload returned HTTP 500 after its run, but the complete local results above
were retained and the model unloaded cleanly.

The two failures need context:

- **Qwen3.8 at 131K:** chunked prefill reached `kv_len=104448`. oMLX
  estimated 50.82 GB current allocation plus 1.69 GB transient allocation,
  exceeding its 52.20 GB prefill safety cap (90% of the 58.0 GB Metal
  ceiling). The preceding 98,304-token chunk completed, so 98K is retained as
  the server ceiling with roughly one 6K-token chunk of margin; the largest
  complete benchmark request was 65K.
- **Gemma 31B at 65K:** the first attempt predicted 45.33 GB against a
  45.06 GB Balanced dynamic ceiling and was rejected by only 0.27 GB. After
  closing desktop apps, the same 65K test passed. This makes 65K a
  memory-pressure-sensitive hard ceiling, not a guarantee under heavy desktop
  load. A 131K run was not attempted; based on the 65K allocation and ten wide
  full-attention layers, it would exceed the 58 GB safety envelope.

### Decisions derived from the benchmark

| Model | Native metadata | Largest verified evidence | oMLX hard ceiling | OpenCode comfort limit | Decision |
|---|---:|---|---:|---:|---|
| Qwen3.6 35B-A3B | 262,144 | 200K full pass | 262,144 | 65,536 | Keep native server capability because hybrid attention has cheap KV and 200K passed; compact normal coding near 65K because 200K TTFT was about seven minutes. The untested 262K endpoint is capability, not a performance claim. |
| Qwen3.8 27B | 262,144 | 65K full pass; prefill reached 98K | 98,304 | 32,768 | Stop the server one chunk below the 104,448 guard failure. Compact daily sessions at 32K because even 65K took nearly four minutes. |
| Gemma 4 31B | 262,144 | 65K pass after freeing desktop memory | 65,536 | 32,768 | Keep the measured 65K server option, but compact at 32K because 65K took 4.4 minutes and was sensitive to other apps. Treat 131K failure as an evidence-based inference, not a measured run. |
| Gemma 4 E4B | 131,072 | 131K full pass | 131,072 | 65,536 | Retain the full native server window; use 65K in OpenCode because it took 25 seconds versus 67 seconds at 131K. |

These are intentionally two layers of policy: oMLX's `max_context_window` is
the safety/capability boundary, while OpenCode's `limit.context` is the normal
experience boundary that drives earlier compaction. Revisit the OpenCode
limits first when optimizing latency; raise an oMLX ceiling only after a new
memory-guard benchmark under representative desktop load.

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
| small model | `gemma-4-E4B-it-oQ4e-mtp` | routes lightweight OpenCode work such as title generation to the ~5.2 GB, ~89 tok/s model |
| `temperature` | **0.2** | low entropy → fewer runaway/degenerate generations. **Set in the model `options`, NOT the agent** — OpenCode drops agent-level temperature for openai-compatible providers (verified on the wire). |
| `top_p` | 0.9 | same |
| `max_tokens` (+ `limit.output`) | **2048** | bounds a runaway generation |
| `limit.context` | **65536 / 32768 / 32768 / 65536** | OpenCode comfort budgets for Qwen3.6, Qwen3.8, Gemma 31B, and Gemma E4B; these trigger compaction before multi-minute fresh prefills while oMLX retains the higher measured hard ceilings |
| `compaction` | **auto + prune + 8192 reserved** | compacts with 8k tokens of headroom and removes stale bulky tool output from the active prompt while durable session history remains available |
| `timeout` / `headerTimeout` / `chunkTimeout` | **300000 / 120000 / 90000** | a stalled stream becomes a resendable error instead of an infinite TUI freeze; still allows cold loads and dense-model prefills |

## Gotchas
- **The UI's "disable API key" toggle may not persist.** If the server keeps
  returning 401 after you disable auth, set
  `auth.skip_api_key_verification: true` in `~/.omlx/settings.json` directly
  and restart the server process.
- **Memory guard math**: ceiling ≈ free + inactive + 50% of active RAM,
  hard-capped by the kernel Metal limit (`iogpu.wired_limit_mb`; the observed
  macOS default was ~51.8 GB on this 64 GB Mac, while the benchmark session
  temporarily raised it to 58.0 GB). On "would exceed the dynamic memory
  ceiling", unload the
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
