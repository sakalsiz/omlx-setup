# Operations and troubleshooting

This guide covers runtime behavior and recovery for the configuration in
[Setup and model configuration](setup.md). Historical measurements and the
reasoning behind context limits live in the
[2026-08-30 context benchmark](benchmarks/2026-08-30-context.md).

## Memory guard and macOS

With the Balanced policy, the dynamic ceiling is approximately free memory +
inactive memory + 50% of active memory, hard-capped by the kernel Metal limit.
The observed macOS default `iogpu.wired_limit_mb` was about 51.8 GB on this
64 GB Mac. The benchmark session temporarily raised it to 59,392 MB (58 GiB):

```sh
sudo sysctl iogpu.wired_limit_mb=59392
```

The `sysctl` change resets at reboot. Wired GPU memory cannot be reclaimed by
macOS like ordinary application memory, so leave several GB of headroom and
do not treat the 58 GiB hard cap as a safe everyday allocation target.

The Balanced dynamic ceiling changes with other application usage. A model
request that passes after closing apps can still be rejected under desktop
load; Gemma 31B at 65K demonstrated this directly. Prefer closing memory-heavy
apps or reducing/compacting context over disabling the guard.

If oMLX reports that a request would exceed the dynamic memory ceiling, unload
the least-recently-used model from the dashboard or with:

```sh
curl -X POST http://127.0.0.1:8000/admin/api/models/<id>/unload
```

oMLX may automatically evict a model under pressure. This is expected and is
safer than forcing macOS into severe compression or swap pressure.

## Authentication

The UI's “disable API key” toggle may not persist. If localhost requests still
return HTTP 401, set `auth.skip_api_key_verification: true` in
`~/.omlx/settings.json` and restart the oMLX server process.

## Server port and duplicate processes

The port in `~/.omlx/settings.json` is authoritative; this setup uses port
8000. A second listener on another port usually indicates a leftover server
instance. Restart the app rather than configuring OpenCode around the stale
listener.

## MTP and acceleration settings

Native MTP is persisted per model. Enable it once with:

```text
PUT /admin/api/models/<id>/settings
{"mtp_enabled":true,"mtp_num_draft_tokens":4}
```

Verify that the server log reports `Lightning MTP`; an `MTP` suffix in a model
folder name does not activate it by itself. The current lineup uses MTP with
four draft tokens on every model. TurboQuant KV is disabled because oMLX
treats it as conflicting with MTP; DFlash and speculative prefill are also
disabled in the current measured configuration.

Some dashboard settings say “restart needed” because they are read only while
the server initializes. Per-model context-window updates used in this setup
were accepted without a server reload; follow the dashboard's explicit status
for other settings.
