# omlx-setup

A working, tuned setup for **local agentic coding on Apple Silicon**:
[OpenCode](https://opencode.ai) as the agent, [oMLX](https://omlx.ai) as the
inference server.

## Current configuration

| Model | Role | oMLX hard ceiling | OpenCode comfort limit |
|---|---|---:|---:|
| `Qwen3.6-35B-A3B-oQ4e-MTP` | default coding model | 262,144 | 65,536 |
| `Qwen3.8-27B-oQ6e-MTP` | dense coding second opinion | 98,304 | 32,768 |
| `gemma-4-31B-oQ4e-MTP` | dense general + vision | 65,536 | 32,768 |
| `gemma-4-E4B-it-oQ4e-mtp` | small model + vision | 131,072 | 65,536 |

## Documentation

- [Setup and model configuration](docs/setup.md) — components, model roles,
  OpenCode wiring and current tuned parameters.
- [Context benchmark — 2026-08-30](docs/benchmarks/2026-08-30-context.md) —
  immutable test environment, raw measurements, guard failures and context
  decision record.
- [Operations and troubleshooting](docs/operations.md) — memory guard, kernel
  Metal limit, authentication, ports and acceleration settings.
- [`opencode.jsonc`](opencode.jsonc) — drop-in OpenCode configuration; copy it
  to `~/.config/opencode/opencode.jsonc`.

Clone it, swap the model list for whatever you run, and make it your own.
