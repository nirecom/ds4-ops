# Tuning — parameters and their rationale

Every ds4-server flag and Claude Code env var, with the value and **why**. Design context is
in [architecture.md](architecture.md); procedures in [ops.md](ops.md); the chronological
incidents that produced these values in [history.md](history.md).

## Server flags (ds4-server)

| Flag | Value | Why |
|---|---|---|
| `--metal` | — | Apple GPU backend. |
| `--quality` | on | Prefer exact kernels over faster approximate paths. Direct accuracy lever under the q2-q4 quant; costs some tok/s (accepted). |
| `--ctx` | `393216` | 384K context, and the gate for Think Max (`DS4_THINK_MAX_MIN_CONTEXT = 393216`). See "Think Max" below. |
| `--kv-disk-dir` | `~/Library/Caches/ds4-server/kv` | Persistent + Time Machine-excluded by macOS default. `/tmp` was rejected (non-persistent, and TM-*included* → would back up 100s of GB). Same SSD volume, so no speed loss. |
| `--kv-disk-space-mb` | `32768` | 32 GB cap: skips pathologically large single checkpoints and bounds eviction. Not a total-write cap (eviction deletes, so churn can still exceed it). |
| `--kv-cache-cold-max-tokens` | `90000` | Largest single cold-save snapshot; big enough to cache a typical CC initial context in one write, below `--ctx`. |
| `--kv-cache-continued-interval-tokens` | `25000` | **The write-amplification lever.** Each continued checkpoint rewrites the whole live prefix (not a delta), doubled f16→f32. The default 10000 caused a 137 GB write storm; 25000 more than halved the churn. |
| `--warm-weights` | on | Page in the whole model at startup. RSS ~90.9 GB is expected, not a leak. |
| `--host 0.0.0.0` | — | LAN-reachable, **no auth**. Trusted network only. |

## Memory budget (128 GB)

Weights ~90.9 GB resident; KV grows lazily as a session fills (startup RSS ≈ weights only).

| ctx | full-context KV | peak RSS | macOS headroom |
|---|---|---|---|
| 327680 | ~8.5 GB | ~99.4 GB | ~28.6 GB |
| **393216** | ~10.2 GB | **~101.1 GB** | **~26.9 GB** |

393216 fits comfortably as a dedicated server. `1M` ctx (~26 GB KV → ~117 GB peak) is too
much — do not.

## Thinking control

- ds4 defaults **every** chat request to HIGH thinking.
- `effort` is nearly inert: `low`/`medium`/`high`/`xhigh` all collapse to HIGH; only `max`
  maps to Think Max. Claude Code's default `xhigh` therefore yields HIGH, unchanged from the
  server default.
- The **model name** is the real switch: `deepseek-chat` = no thinking, `deepseek-reasoner`
  = thinking on, anything else = default HIGH. In thinking mode, client sampling knobs
  (temperature/top_p/top_k) are ignored, like the official API — so accuracy cannot be tuned
  via sampling.

### Think Max (3 conditions, all required)

1. Client sends `reasoning_effort=max` or `output_config.effort=max`
2. Server `--ctx >= 393216` ✅ (this config)
3. Thinking mode on (not `deepseek-chat` / `think:false`)

Claude Code defaults to `xhigh` → HIGH, so **ctx alone does not give max** — the client must
emit `effort=max`, and whether CC forwards that to a custom endpoint is unverified. Think Max
= more reasoning = slower; a modest accuracy lever for hard planning only. For most work,
HIGH + `--quality` is the better trade.

## Client env vars (Claude Code)

| Var | Value | Why |
|---|---|---|
| `ANTHROPIC_BASE_URL` | `http://<mac-ip>:8000` | Route CC to ds4. |
| `ANTHROPIC_AUTH_TOKEN` | any non-empty | ds4 does not authenticate; a dummy satisfies the client. |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | `393216` | Tell CC ds4's real ceiling (CC otherwise assumes the model's nominal 200K/1M window and never compacts in time). Keep in sync with `--ctx`. |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | `75` | Compact at ~75% (~245K by CC's count). Absorbs the tokenizer mismatch — ds4/DeepSeek counts more tokens than CC/Claude for the same text, so CC must compact well before its own ceiling to keep ds4's count under 393216. |

### Per-tier thinking split without a router (optional)

To give heavy-reasoning agents thinking and mechanical workers non-thinking on ds4, use
Claude Code's alias-resolution vars (they also propagate to sub-agent `model:` frontmatter):

| Var | Value |
|---|---|
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `deepseek-reasoner` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `deepseek-chat` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `deepseek-chat` |

Caveat: whether CC accepts a non-Anthropic alias target on a custom endpoint is the one thing
to verify empirically (grep the ds4 log for `THINKING` per request). If CC rejects it, fall
back to a router.
