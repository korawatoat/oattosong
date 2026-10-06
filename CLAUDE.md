# Oatto Song — Skill Registry and Configuration

This repository contains the `oattosong` skill: planning and packaging of original songs for the Oatto Music YouTube channel, ending in a measured, structured handoff to Hermes AI for publishing.

## Skill Registration

Defined in `SKILL.md` at the repository root:

- **Name**: `oattosong`
- **Trigger Commands**: `/oattosong`, `/song`, `/music`
- **Subcommands**: `plan` (4-week release plan), `audio <song_id>` (Stage B), `batch N`
- **Mode**: Interactive
- **Purpose**: concept → lyrics → Suno prompts → cover/visualizer prompts → metadata → Shorts plan → scorecard (Stage A); audio QC → Drive → Hermes handoff (Stage B)

Source brief: `OATTO_MUSIC_AUTOMATION_SKILL_BRIEF.md` (kept alongside as reference).

## Role boundary

- **Claude produces** — release plan, concept, lyrics, prompts, packaging, measured audio QC of a real file, Drive upload, handoff, notification draft
- **พี่โอ๊ต decides and makes the song** — picks the concept, generates and *listens to* the audio, approves the take
- **Hermes publishes** — verifies files/hashes, ffprobe, YouTube duplicate check, upload, schedule, playlists, Facebook mirror, records IDs
- Claude never uploads/schedules/publishes, never generates the song itself, never invents analytics

## Three invariants

1. **Claude cannot hear.** Sound-related judgments are `owner_rated` or `not_verified`; only measurable properties (duration, codec, sample rate, channels, loudness, peak, clipping, silence) are recorded as measured. The Stage A scorecard is a text-based estimate and says so.
2. **A link is not an upload.** `READY_FOR_UPLOAD` = audio bytes physically in Drive. Otherwise Controlled URL Fallback → `READY_FOR_UPLOAD_VIA_URL`.
3. **Every QC value is measured, never assumed.**

## Relationship to the other skills

- `oattomusic` (Shorts footage from a finished song) is downstream; it can consume `song_package.json`.
- `youtubeoat`, `meaw`, `longlifeheath` share the Fast Mode shape (Phase 1 is the only interactive stop), Batch Mode, the Hermes producer/publisher split, and the URL fallback contract. **Status vocabulary note**: this skill uses the brief's vocabulary (`READY_FOR_UPLOAD`, `WAITING_FOR_AUDIO`, …) rather than `READY_FOR_HERMES`, because Hermes already knows the Oatto brief's names. If the shared rules change in a sibling (URL fallback contract, Fast Mode shape), change them here too — they drift silently otherwise.

## Setup

1. GitHub connection — https://claude.ai/connect-github, make sure the Claude GitHub App can see this repository
2. Register the skill — add the repository to your Claude workspace; `SKILL.md` at the root is discovered
3. Connectors: Google Drive (required for handoff), Higgsfield (optional, cover art). No Telegram bot — the notification is drafted for you to send.

## Files

- `SKILL.md` — full workflow
- `README.md` — getting started
- `CLAUDE.md` — this file
