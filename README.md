# oattosong — Oatto Music song planning & handoff

Claude plans, writes and packages the song. You generate and listen. Hermes publishes.

## Daily use

```
/oattosong            # new song: pick 1 of 5 concepts, everything else runs
/oattosong plan       # 4-week release plan
/oattosong audio ID   # you've made the song — measure it, upload, hand off
/oattosong batch 3    # three songs from one concept round
```

## What happens

**Stage A (you pick a concept — the only question)**
duplicate check → structure → original lyrics (singability-linted) → Suno prompts (primary + 2 alternatives + compact versions + exclude list) → cover & visualizer prompts → title/description/tags/AI-disclosure → 3–5 Shorts plan → concept scorecard → `WAITING_FOR_AUDIO`

**You**: generate takes in Suno (the skill gives a playbook), listen, choose one.

**Stage B (`/oattosong audio ID`)**
measure the file (duration, codec, sample rate, loudness, peak, clipping, silence) → finalise metadata → Drive upload (or URL fallback) → handoff → drafted message for Hermes.

## Honest limits

- Claude can't hear. Hook memorability, vocal quality, melody originality and "does it sound like something else" are rated by you; the scorecard shows them blank until you do.
- Large audio files can't be pushed through the Drive connector — copy within Drive, or use the URL fallback (`READY_FOR_UPLOAD_VIA_URL`).
- Streaming-link, view, and retention numbers are never made up; give Claude real data (or Hermes' reports) and it will use them.

See `SKILL.md` for the complete workflow and `CLAUDE.md` for configuration.
