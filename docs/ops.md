# Ops

Day-to-day procedures. Parameter rationale is in [tuning.md](tuning.md); hosts/ports in
[infrastructure.md](infrastructure.md).

## Run the server (Mac)

```sh
mkdir -p ~/Library/Caches/ds4-server/kv     # first time only
~/git/ds4-ops/scripts/ds4-start.sh
```

Runs in the foreground. For background:
```sh
nohup ~/git/ds4-ops/scripts/ds4-start.sh > ~/ds4-server.log 2>&1 &
```

Stop:
```sh
kill "$(pgrep -f ds4-server)"
```

`caffeinate` is baked into the script and exits with the server; no separate step.

## Client (Windows)

Set per-session env, then launch Claude Code:
```powershell
$env:ANTHROPIC_BASE_URL = "http://<mac-ip>:8000"
$env:ANTHROPIC_AUTH_TOKEN = "dummy"
$env:CLAUDE_CODE_AUTO_COMPACT_WINDOW = "393216"
$env:CLAUDE_AUTOCOMPACT_PCT_OVERRIDE = "75"
claude
```

Optional per-tier thinking split: also set the `ANTHROPIC_DEFAULT_*_MODEL` vars from
[tuning.md](tuning.md#per-tier-thinking-split-without-a-router-optional).

## Monitoring (Mac)

| Check | Command |
|---|---|
| Sleep events (should be none while serving) | `pmset -g log \| grep -i "entering sleep"` |
| KV cache size | `du -sh ~/Library/Caches/ds4-server/kv` |
| Memory pressure / swap | `sysctl vm.swapusage` |
| Thinking on/off per request | grep the server log for `THINKING` in the `chat ...` lines |
| Process alive / one instance | `pgrep -fl ds4-server` |

## Recovery

| Symptom | Action |
|---|---|
| `400 context_length_exceeded` | Client conversation grew past ctx. `/compact`, or `/clear`, or raise `--ctx` and restart (the running conversation then fits). |
| `error during compaction` | The compaction request itself exceeds ctx. Ensure `CLAUDE_CODE_AUTO_COMPACT_WINDOW` + `PCT_OVERRIDE` are set so compaction fires *before* the ceiling; otherwise `/clear` or restart the server at higher ctx so the current conversation fits, then compact. |
| Server unresponsive / client API errors after idle | Check `pmset -g log` for a sleep window. caffeinate should prevent it; confirm the process is still wrapped. |
| `kv cache evicted reason=disk-cache-full` | Normal capacity management — not an error. Ignore. |

## Enable Think Max (accuracy option)

All three required (see [tuning.md](tuning.md#think-max-3-conditions-all-required)):
1. `--ctx >= 393216` ✅ (current config)
2. Thinking on (model resolves to `deepseek-reasoner` or default, not `deepseek-chat`)
3. **Client sends `effort=max`** — CC defaults to `xhigh`→HIGH, so this is the missing piece;
   verify CC can be set to max and forwards it before expecting Think Max.
