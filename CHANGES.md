# Changes in `duffercn/grok2api` — branch `feat/extend-prompt`

Fork of [chenyme/grok2api](https://github.com/chenyme/grok2api), based on upstream `main` (commit `717d2fc`).

## Added: `extend_prompt` — per-segment prompt for multi-segment video

**Problem**: When generating videos longer than 10s, grok2api splits generation into multiple segments (e.g. 16s = 10s + 6s). The same `prompt` was used for every segment, making it impossible to direct the narrative arc of the video — the second segment would just continue the first with no new instructions.

**Solution**: Added an `extend_prompt` field to `POST /v1/videos`. When provided, continuation segments use `extend_prompt` instead of the original `prompt`.

```
POST /v1/videos
  prompt         = "opening scene prompt"
  extend_prompt  = "closing scene prompt"   ← NEW
  seconds        = 16
```

Files changed: `app/products/openai/video.py`, `app/products/openai/router.py`

---

## Added: `segment_seconds` — explicit per-segment duration control

**Problem**: Multi-segment splits were hardcoded server-side (`16s` always splits as `10+6`, never `6+10`). Clients had no way to control the duration of each segment independently.

**Solution**: Added a `segment_seconds` field (comma-separated string, e.g. `"6,10"`) that overrides the built-in split table. Each value must be `6` or `10`. At most 2 segments supported.

```
POST /v1/videos
  prompt           = "first segment prompt"
  extend_prompt    = "second segment prompt"
  seconds          = 16
  segment_seconds  = "6,10"   ← NEW: first 6s, then 10s
```

Supported combinations:

| segment_seconds | total | description |
|-----------------|-------|-------------|
| `"6"`           | 6s    | single segment |
| `"10"`          | 10s   | single segment |
| `"6,6"`         | 12s   | two 6s segments |
| `"10,6"`        | 16s   | 10s then 6s (default for 16s) |
| `"6,10"`        | 16s   | 6s then 10s ← **newly unlocked** |
| `"10,10"`       | 20s   | two 10s segments |

Files changed: `app/products/openai/video.py`, `app/products/openai/router.py`

---

## Usage with the `ai-gen` client tool

These changes are designed to work with the `--seg` flag in [`~/.claude/skills/ai-gen`](https://github.com/duffercn/grok2api):

```bash
# 6s intro + 10s extended reaction (previously impossible)
generate_video.py \
  --seg "6:Host asks a provocative question, handheld camera" \
  --seg "10:Guest reacts, laughs, delivers witty comeback" \
  --provider grok2api --ar 16:9 -o output.mp4
```

The client automatically sends both `extend_prompt` and `segment_seconds` to the server.
