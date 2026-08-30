# omlx-setup

A working, tuned setup for **local agentic coding on Apple Silicon**:
[OpenCode](https://opencode.ai) as the agent, [oMLX](https://omlx.ai) as the
inference server.

- [`omlx-opencode-setup.md`](omlx-opencode-setup.md) — the guide: component
  overview, model choices for 64 GB, reproducible context benchmark evidence,
  decision points, tuned parameters with rationale, and gotchas (auth toggle,
  memory guard, kernel Metal cap).
- [`opencode.jsonc`](opencode.jsonc) — drop-in OpenCode config
  (`~/.config/opencode/opencode.jsonc`) with the oMLX provider and tuned
  per-model settings.

Clone it, swap the model list for whatever you run, and make it your own.
