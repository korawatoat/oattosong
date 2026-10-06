---
name: oattosong
description: "Plan and package original songs for the Oatto Music YouTube channel. The user picks a concept — that is the only decision point — then the skill runs unattended through duplicate check → lyrics → Suno prompts → cover/visualizer prompts → metadata → Shorts plan → concept scorecard → status WAITING_FOR_AUDIO. When the user later brings the finished audio, a second stage measures it, uploads to Google Drive and hands off to Hermes AI (พี่วอก) for YouTube publishing. Also builds a 4-week release plan. Use '/oattosong', '/song', or '/music'. Supports Batch Mode. For turning a finished song into Shorts footage, use the separate oattomusic skill."
trigger_commands:
  - /oattosong
  - /song
  - /music
mode: interactive
---

# Oatto Music — Song Planning & Packaging Skill

**Channel**: Oatto Music (English-language emotional cinematic songs)
**Positioning**: *Emotional cinematic songs for quiet hearts.* — "music for people who feel too much to say it out loud."
**Source brief**: `OATTO_MUSIC_AUTOMATION_SKILL_BRIEF.md` (the channel rules in §Channel Rules below are distilled from it; if the two ever disagree, the brief wins and this file gets fixed).

**Purpose**: Claude is the *thinking and packaging system*. It plans the release calendar, designs each song (scene + emotion + hook), writes original lyrics, writes the music-generation prompts, scores the concept, and produces a structured package. Then it hands the finished, measured package to Hermes.

## Role boundary

| Who | Does |
|---|---|
| **Claude** | concept, lyrics, Suno prompts, cover/visualizer prompts, title/description/tags, Shorts hooks, duplicate check against the catalog, concept scorecard, release plan, measured audio QC *of a file it actually has*, Drive upload, structured handoff, Hermes notification draft |
| **พี่โอ๊ต (the user)** | picks the concept, **generates the song in Suno (or another tool) and listens to it**, decides whether it is good enough, supplies the audio file |
| **Hermes / พี่วอก** | verifies files and hashes, ffprobe, duplicate-check on YouTube, upload, schedule, playlists, Facebook mirror, records video IDs, reports back in Thai |

Claude must **never**: upload to YouTube/Facebook, schedule or publish, claim a song passed QC without a real file, invent analytics, name a living artist to imitate, or use unverified Spotify/YouTube links.

---

## Three invariants

These are the rules that fail first under time pressure.

1. **Claude cannot hear.** Everything about how the song *sounds* — hook memorability, vocal character, melody originality, whether it clips musically, whether the vocal is intelligible — is unknown to Claude unless the user reports it. Claude measures what tooling can measure (duration, codec, sample rate, channels, loudness, peak) and records everything else under `not_verified` or `owner_rated`. A concept scorecard is an estimate from the *text*, and is labelled as one. Never write "PASS" for a property nobody measured or listened to.
2. **A link is not an upload.** `READY_FOR_UPLOAD` means the audio (and video, if any) bytes are physically in the Drive folder. Audio files of several MB cannot be written through the Drive connector (it only accepts inline content). If the user's file already lives in Drive, copy it in with the Drive connector's copy tool. Otherwise use the Controlled URL Fallback and the distinct status `READY_FOR_UPLOAD_VIA_URL`.
3. **Every QC value is measured, never assumed.** A number copied from the Suno page, the filename, or this skill's defaults is not a measurement.

---

## Entry points

| Command | What it does |
|---|---|
| `/oattosong` | Stage A for a new song: concept pick → full package → `WAITING_FOR_AUDIO` |
| `/oattosong plan` | Build or refresh the 4-week release plan (see Planning Mode) |
| `/oattosong audio <song_id>` | Stage B: the user brings the finished audio → QC → Drive → Hermes handoff |
| `/oattosong batch N` | Stage A for N songs from one concept round |

---

## ⚡ Fast Mode (default) — Stage A

**Phase 1 is the only stop-and-wait point.** Same automation shape as `youtubeoat`, `meaw`, `longlifeheath`. After the concept pick, treat the user as away and run to a terminal status.

- **Phase 1 — Concept**: propose 5 concept cards, wait for the pick. Last stop.
- **Phase 2 — Duplicate check**: automatic, against `catalog.json` (see Catalog). `possible_duplicate` → don't ask; auto-pivot (different place, different relationship situation, different hook image, different energy arc), record the pivot in `why_this_concept`, continue. `blocked` → the one case that returns to the user.
- **Phase 3 — Structure, Lyrics, Prompts, Package, Scorecard**: automatic.
- **Terminal status of Stage A**: `WAITING_FOR_AUDIO` — an external wait (the user must make the song), not a question. Print the package summary and the exact next step.

Stage B starts when the user returns with audio; it is again unattended after that single hand-over.

**Unattended-run rule**: never end a run on a question other than the Phase 1 pick. If the user wants to override an auto-decision (language, series, vocal/instrumental, BPM range), they say so when picking.

---

## 🗓 Planning Mode (`/oattosong plan`)

The user asked Claude to *plan the work*, so planning is a first-class mode, not an afterthought.

1. Read `catalog.json` (released + in-flight songs) and `release_plan.json` if present.
2. Produce a **4-week plan** at ~1 full song per week (brief §9), not several in one day:
   - one **pillar per week**, rotating so no pillar appears twice in a row and no pillar exceeds ~40% of the plan
   - mix of vocal / instrumental (instrumental ≈ 1 in 4–5)
   - for each slot: pillar, 2 concept seeds (scene + emotion + hook), target playlist, Shorts schedule (3–5 Shorts spread over the following days, not same-day)
   - **production lead time**: Stage A package due ≥ 5 days before release so the user has time to generate and choose among takes
3. Seasonal fit: Autumn Glow around autumn, Rainy City in rainy season — but never let the calendar override quality.
4. Save `release_plan.json` + a readable `release_plan.md` to `01_BRIEF/`. Planning mode asks no questions; it ends with one table.
5. After songs ship, if the user supplies real performance numbers (views, average view duration, repeat-listening / returning-viewer rate), fold them into the next plan. **Never fabricate analytics** — if none were given, the plan says "no performance data yet" and uses only the brief's priorities.

---

## 🔁 Batch Mode

Trigger: "3 songs", "ทำทีเดียว 4 เพลง". Default when unspecified: 3.

1. **One concept round**: propose ≥ 2×N cards across *different pillars*; user picks N, or says "auto pick N".
2. Duplicate check across the N picks and against the catalog (the picks must also differ from each other).
3. Run Stage A per song sequentially; one consolidated summary and **one** Hermes notification draft.
4. Batch does not mean batch release: the plan still spaces releases ~1 per week.

---

## 🎼 Stage A — Song Package

### Phase 1 — Concept Ideation

**Start from scene + emotion + hook — never from genre.** Propose 5 concept cards, each from a *different* pillar where possible (vary mood, energy, vocal/instrumental). Each card:

```
working_title
series / pillar        (A Midnight Transit | B Rainy City | C Autumn Glow | D Quiet Heart | E Emotional Instrumental)
scene                  where/when it happens, one concrete image
emotional_core         the one feeling (not "sad" — be specific: relief-that-hurts, politeness-as-grief…)
listener_situation     who is listening, when
one_sentence_story
central_image          the visual a listener will remember
hook_phrase            the line that gets sung/remembered
energy_arc             e.g. sparse → bloom at chorus 2 → hush
vocal_or_instrumental  with reason
```

Check each card against the **seven identity questions** (brief §4): where, who feels what, what memory/event, memorable image, hook, how it differs from the previous song, why replay. A card that cannot answer all seven is rewritten before being shown.

One pillar per song. No song carries every emotion.

Record `why_this_concept`, `performance_learning` (only from real data supplied; otherwise `"none supplied"`), `recommended_follow_up`.

### Phase 2 — Duplicate Check

Compare against `catalog.json` (title, pillar, scene, central image, hook phrase, key lyric images, BPM/key band, cover palette) and the Drive `04_READY_TO_UPLOAD/` + `05_ARCHIVE/` folders.

- Same pillar is fine. Same *scene + image + hook* with different wording is not.
- Also check **recency**: the new song must differ from the last 3 releases in at least two of: pillar, tempo band, vocal persona, central image, energy arc.
- Result: `clear` | `possible_duplicate` (auto-pivot) | `blocked`.

If the catalog is empty or unreadable, say so — `duplicate_check: clear — catalog empty/unavailable, first release or unverifiable` — rather than a bare `clear`.

### Phase 3 — Song Structure

Pick the structure the song needs; do not force one template.

- Pop: Intro → Verse 1 → Pre-Chorus → Chorus → Verse 2 → Chorus → Bridge → Final Chorus → Outro
- Two-part: Intro → A → B → A → B → Instrumental outro
- Instrumental: evolving form, with a stated **motif** (the 4–8 note idea listeners should be able to hum) and 3–4 named sections with dynamic levels

State the structure and why, plus target length (typically 2:45–3:45 for vocal songs; the generator may land elsewhere).

### Phase 4 — Lyrics (original, singable)

Rules (brief §5 Step 3):

- Fully original. No lines or signature phrases from copyrighted songs. No imitation of a named artist or band.
- Concrete, visible detail over abstractions. Avoid clichés ("broken heart", "forever", "never let you go") unless a fresh image carries them.
- Hook is short, rhythmic, and singable. Check syllable counts, stress placement, and rhyme per section; record syllables per line in `lyrics_notes`.
- English primary. Short Japanese words/phrases allowed sparingly; add a romanization/pronunciation note, and spell the line the way the generator will pronounce it (romaji, not kanji, unless tested).
- **No digits** in lyrics — write numbers as words ("twelve fifty-two", not "12:52").
- Use section tags: `[Intro] [Verse 1] [Pre-Chorus] [Chorus] [Verse 2] [Bridge] [Final Chorus] [Outro]` (instrumentals: tags describe sections and dynamics instead).
- Keep vocal-generator-friendly: no long parentheticals, no stacked unusual punctuation, avoid tongue-twisters and lines needing a breath mid-word.

**Lyric self-lint (run and record)**:
1. no digits / no all-caps shouting
2. banned-cliché scan (list in `lyrics_notes`, each kept cliché justified)
3. hook appears ≥ 3× (or Intro/Outro echo) and is ≤ ~8 syllables unless the melody plan justifies more
4. repeated-words check against the last 3 catalog songs' key images
5. no living-artist names, no brand/trademark lyrics unless central and safe
6. total lyric length fits the generator's lyric field (~3,000 characters is the long-standing Suno limit; **verify the current limit** — settings change)

### Phase 5 — Music Prompts

Produce, per brief §5 Step 4:

- `style_prompt` (primary): genre, mood, BPM, key if needed, instrument palette, vocal persona, production texture, dynamic arc, avoid-list
- `alternative_prompt_a`, `alternative_prompt_b` (meaningfully different — e.g. different tempo/instrumentation, not a synonym swap)
- `exclude_styles`
- `vocal_direction`, `arrangement_notes`
- a **compact variant** of each style prompt under ~200 characters for generators with short style fields; and keep the full primary under ~1,000 characters (**verify current field limits** — they have changed between Suno versions)

**Never name an artist or band** — describe the sound instead ("intimate breathy female vocal, close-mic'd, little vibrato").

Add a **generation playbook** for the user (they make the song): generate 4–6 takes of the primary, 2–3 of each alternative; listen for the hook landing in the first chorus; listen to Chorus 1 vs Final Chorus lift; reject takes where the vocal mispronounces the hook or Japanese words; use "Extend"/"Replace section" instead of regenerating the whole song when only one section is weak; record the Suno song ID of the chosen take in the handoff (`generator_take_id`).

### Phase 6 — Visual Direction

Oatto visuals are simple but memorable (brief §8). Produce:

- `cover_art_prompt`: one strong image; palette tied to the series (Midnight Transit: blue-black/sodium orange/wet glass; Rainy City: magenta-cyan reflection/wet asphalt; Autumn Glow: amber/cream/muted green; Quiet Heart: dim room/empty chair/phone glow). Little or no text on the image. No living-artist style names, no logos, no generic stock look.
- `visualizer_prompt`: slow zoom or slow parallax; rain/particles/light movement kept subtle; must never compete with the song.
- Text safety: any title text on the cover is a *separate overlay* decision (`thumbnail_text_optional`) — image generators misspell text, so default to no text in the generated image.

**Optional generation (only if the user asked, or `auto_cover: true` in the pick)**: Claude may generate the cover through Higgsfield `generate_image` (preflight cost where supported, state it, submit; decline preset recommendations and use the literal prompt; one job at a time). A generated cover is only `COVER_GENERATED`, never "cover QC passed", unless the frames were actually viewed.

### Phase 7 — Metadata, Shorts Plan

Produce `final_title` (+ 2 alternates), `full_description`, `tags`, `hashtags`, `playlist_recommendation` (one primary playlist + one mood playlist where it fits — see Playlist Architecture), `content_warnings`, `copyright_notes`.

**Description rules**: no overclaiming ("most beautiful…"), no unverified streaming links, lyrics optionally included, a short human line about the song's world before the metadata, and an **AI-use disclosure line**. Disclose honestly that the track was made with AI music tools; do *not* write it as a boast and do not hide it. YouTube's creator tools include an "altered or synthetic content" disclosure — flag in `copyright_notes` that Hermes/the user should answer it truthfully for the cover/visualizer and vocals.

**Shorts plan — 3–5 per song** (brief §9), each with a role, a hook line, a caption, and a link-back call to action to the full song:

1. Hook Short · 2. Story/POV Short · 3. Chorus Short · 4. Emotional-quote Short · 5. Instrumental-atmosphere Short

Timestamps (which seconds to cut) are `not_verified` until audio exists — Claude cannot hear where the chorus lands. Stage B fills them from the user's report or from measured structure when available.

### Phase 8 — Concept Scorecard (honest version)

The brief's scorecard has ten dimensions. Split them by what Claude can actually judge:

- **Claude-estimated from the text** (label `basis: concept+lyrics`): Emotional clarity, Fits Oatto identity, Visual potential, Short-form potential, Difference from recent releases, Hook *as written* (rhythm/syllables/imagery), Lyric originality
- **Owner-rated after listening** (`owner_rated`, blank until the user supplies): Hook memorability, Vocal character, Melody originality, Repeat-listening potential, Production fit
- Overall `concept_average` = mean of the Claude-estimated scores only. Never fill an owner-rated dimension by guessing.

Thresholds (brief §6): ≥ 8.0 recommend making; 7.0–7.9 revise before generating; < 7.0 rewrite or shelve. **A concept scoring < 7.0 gets one automatic rewrite pass**, then is reported honestly with its score — Claude does not inflate to clear the bar. After audio exists, the final release recommendation requires the owner-rated dimensions; without them the recommendation is `PENDING_OWNER_LISTEN`.

### Phase 9 — Write the package

Files written to `02_CONCEPTS/<song_id>/` in Drive (small text → connector is fine):

```
song_package.json     — the full package (schema below)
lyrics.txt            — tagged lyrics, plain text
suno_prompts.md       — primary, A, B, compact variants, exclude, playbook
cover_prompt.md       — cover + visualizer prompts
description.txt       — title options, description, tags, hashtags
shorts_plan.md        — 3–5 Shorts: role, hook, caption, CTA
scorecard.json        — estimated vs owner_rated
README.md             — status + exact next step for the user
```

`song_id` format: `OM-YYYY-MM-DD_slug` (e.g. `OM-2026-10-06_last-train-window`).

Update `catalog.json` (`status: WAITING_FOR_AUDIO`) so later duplicate checks see the in-flight song.

**`song_package.json` schema** (extends brief §7 — every field in the brief's list is present):

```json
{
  "song_id": "OM-YYYY-MM-DD_slug",
  "series": "Midnight Transit",
  "pillar": "A",
  "working_title": "",
  "final_title": "",
  "alt_titles": [],
  "artist": "Oatto Music",
  "language": "en",
  "concept": "",
  "emotional_core": "",
  "hook": "",
  "structure": "",
  "lyrics": "",
  "lyrics_notes": { "syllables_per_line": "", "lint": {}, "pronunciation_notes": "" },
  "style_prompt": "",
  "alternative_prompt_a": "",
  "alternative_prompt_b": "",
  "compact_prompts": { "primary": "", "a": "", "b": "" },
  "exclude_styles": "",
  "vocal_direction": "",
  "arrangement_notes": "",
  "target_bpm": 0,
  "cover_art_prompt": "",
  "visualizer_prompt": "",
  "full_description": "",
  "tags": [],
  "hashtags": [],
  "playlist_recommendation": { "primary": "", "mood": "" },
  "shorts_hooks": [],
  "shorts_captions": [],
  "thumbnail_text_optional": "",
  "content_warnings": "",
  "copyright_notes": "",
  "ai_disclosure": "",
  "duplicate_check": "clear",
  "why_this_concept": "",
  "performance_learning": "none supplied",
  "recommended_follow_up": "",
  "scorecard": { "basis": "concept+lyrics", "estimated": {}, "owner_rated": {}, "concept_average": 0 },
  "planned_release": "YYYY-MM-DD or null",
  "status": "WAITING_FOR_AUDIO",
  "created_by": "noteforoat@gmail.com",
  "created_date": ""
}
```

Print a one-screen summary to the user: title, pillar, hook, the scorecard line, the file locations, and the next step ("generate with the primary prompt in Suno; when you have the chosen take, run `/oattosong audio <song_id>`").

---

## 🎧 Stage B — Audio In, Handoff Out (`/oattosong audio <song_id>`)

Trigger: the user supplies the finished audio (local file, Drive file, or a URL they own). This is a single hand-over, then unattended.

### B1 — Receive (`AUDIO_RECEIVED`)

Locate the file. Record: source, filename, `generator_take_id` if given, any title/lyric changes the user made vs the package (**diff the delivered lyrics against `lyrics.txt` if the user provides them; if the sung lyrics differ, update the package and flag it — Hermes must not publish metadata that contradicts the audio**).

### B2 — Technical QC (`QC_PENDING`), measured

Do not assume `ffprobe` exists: check (`which ffprobe`), fall back to `ffmpeg`, then to parsing container atoms (`mvhd`, `stsd`) for MP4/M4A; for MP3/WAV read header fields. If nothing works, technical QC is `UNVERIFIED`, never `PASS`.

**Measure**:
- file opens; SHA-256; byte size
- container + codec; sample rate; channels (stereo expected); bit rate
- duration (compare to the package's expected length — flag > 20% difference, don't block)
- loudness: integrated LUFS and true peak via `ffmpeg -i f -af ebur128=peak=true -f null -` (or `loudnorm=print_format=json`). Streaming target is roughly −14 LUFS integrated with true peak ≤ −1 dBTP; **report the measured value and the delta, don't "fix" the master**.
- clipping: count of samples at/near full scale (`astats`), or peak at 0 dBFS
- leading/trailing silence length; any mid-track silence > 2 s (`silencedetect`)
- if a video file: resolution, aspect, fps, codecs; reject a render that is not 16:9 ≥ 1920×1080 for the full-song video, or 9:16 for Shorts — and **record measured pixels, not requested ones**

**Not verifiable by Claude — goes under `not_verified` or `owner_rated`**:
- audible distortion, vocal intelligibility, mispronounced words, awkward edits, mood match
- whether the melody resembles an existing song (Content ID risk) — the owner should listen with that question in mind; Claude can only say the lyric text was checked
- frames: frozen frames, watermarks, text/logo leaks, flicker — `not_verified — frames not inspected` unless someone actually viewed them (if Claude extracts frames and views them, say that, and what was seen)

Any hard failure (corrupt file, no audio stream, silent, duration implausible, heavy clipping) → `technical_qc.status: FAIL`, overall `BLOCKED` with reasons. Do not "fix" by substituting a different file.

### B3 — Metadata finalisation

Update `final_title` / description / Shorts timestamps with measured duration and any user-reported chorus timestamps. Compute `shorts_plan` cut points only from (a) user-reported times or (b) measured section boundaries; otherwise leave the timestamps `not_verified`.

### B4 — Drive upload (invariant 2)

Target:

```
Oatto Music/
├── 01_BRIEF/              (catalog.json, release_plan.*, series_bible.md)
├── 02_CONCEPTS/<song_id>/ (Stage A package)
├── 03_AUDIO_INBOX/        (optional drop zone for the user's raw files)
├── 04_READY_TO_UPLOAD/<song_id>/
│   ├── song_final.<wav|mp3|m4a>      (REQUIRED, or the fallback link file)
│   ├── cover.<png|jpg>               (if available)
│   ├── video_final.mp4               (if a visualizer was produced)
│   ├── lyrics.txt
│   ├── description.txt
│   ├── song_package.json
│   ├── shorts_plan.md
│   ├── metadata.json
│   ├── manifest.json
│   ├── qc_report.json
│   └── README.md
├── 05_ARCHIVE/
└── 06_BLOCKED/
```

1. If the audio already lives in Drive → copy into the folder with the connector's copy tool (the only reliable route for large files).
2. Otherwise attempt inline upload only if the file is tiny. A multi-MB file cannot go through the connector — don't burn turns on chunked base64, gcloud, or service-account workarounds.
3. After any upload, list the folder and **compare the returned size with the local byte count**. A returned id is not proof.

**Controlled URL Fallback** (`READY_FOR_UPLOAD_VIA_URL`): only when the file cannot go into Drive *and* a `source_url` exists (owner-provided link or generator CDN) whose downloaded bytes' SHA-256 equals the digest of the file that was QC'd. Write `AUDIO_DOWNLOAD_LINK.txt` into the folder (episode, title, url, sha256, bytes, measured duration/sample-rate/LUFS, "generator links may expire — archive a copy"). `technical_qc.status` is then `PENDING_HERMES_DOWNLOAD`, never `PASS`. If the digest doesn't match or the link is dead → `URL_DOWNLOAD_FAILED`.

### B5 — Handoff

Status rules (brief §14 vocabulary plus the URL fallback):

- `READY_FOR_UPLOAD` — audio bytes verified present in Drive, technical QC measured and passed, metadata consistent, owner has approved the take
- `READY_FOR_UPLOAD_VIA_URL` — fallback only
- `QC_PENDING` / `BLOCKED` / `URL_DOWNLOAD_FAILED` otherwise

**Owner approval gate**: a song only reaches `READY_FOR_UPLOAD` if the user has said, in this session or in the provided notes, that they listened and chose this take. If the user supplied audio but said nothing about listening, the status is `AUDIO_RECEIVED` / `QC_PENDING` with the line "waiting for owner's listening approval" — not `READY_FOR_UPLOAD`.

Handoff text (measured values only):

```
READY FOR HERMES REVIEW — SONG FILE PRESENT IN DRIVE
Folder: 04_READY_TO_UPLOAD/<song_id>/
Song: <final_title> — Oatto Music
Series: <series>   Playlist: <primary> + <mood>
Duration: <measured>   Sample rate: <measured>   Loudness: <LUFS> / <dBTP>
Cover: <present|missing>   Video: <present|missing>
Shorts planned: <n> (timestamps: <set|not_verified>)
AI disclosure: included
Technical QC: <PASS|…>   Owner listening approval: <yes|no>
```

### B6 — Hermes Notification Draft

No credential required: compose a plain-text message, print it as a copy-ready block, and also write it to `telegram_payload.txt`. Contents: song id + title, status, Drive folder link, download-link caveat if fallback, QC summary (measured vs not_verified), cover/video presence, playlist, planned release date, Shorts plan summary, AI-disclosure + synthetic-content reminder, Hermes' next steps (verify files → SHA-256 → ffprobe → duplicate-check on YouTube → upload → set playlists → schedule → mirror to Facebook if approved → record video ID/URL/publishAt/privacy → Thai report). **Plain text only; do not rely on Markdown italics** (underscores in `READY_FOR_UPLOAD` get eaten by Telegram `parse_mode: Markdown`).

---

## 📚 Catalog (`catalog.json`)

Claude can only check duplicates against metadata it has been given (brief §11). Keep a single catalog file in `01_BRIEF/`, updated at every status change:

```json
{
  "songs": [
    {
      "song_id": "", "final_title": "", "pillar": "", "series": "",
      "scene": "", "central_image": "", "hook": "", "key_images": [],
      "bpm": 0, "vocal_persona": "", "cover_palette": "",
      "status": "", "planned_release": "", "youtube_video_id": null,
      "released_date": null
    }
  ]
}
```

`youtube_video_id` and `released_date` are filled **only from what Hermes reports back** — never guessed. If the user pastes Hermes' report, update the catalog from it. If the catalog does not exist yet, create it from whatever the user supplies and tell them it's the seed.

## 📻 Playlist Architecture

Playlists are worlds, not dumps. Each song gets one primary playlist and, where fitting, one mood playlist:

Midnight Transit — Oatto Music · Latest Releases — Oatto Music · Emotional Songs — Oatto Music · Rainy Night Songs — Oatto Music · Autumn Glow — Oatto Music · Oatto Music Shorts · Instrumental for Quiet Nights

## Channel Rules (distilled)

**Make**: songs with identity; one hook that can be remembered; visuals from one coherent universe; steady cadence (~1 full song/week); 3–5 purposeful Shorts per song that point back to the full track; repeat only what listeners actually respond to (measured by repeat listening, not hunches).

**Never**: random-genre AI songs; imitation of living artists; many songs with the same title/image/mood; generic instrumentals without character; selling AI tools as the channel's identity; AI-slop volume.

## 🏷 Status Vocabulary

```
DRAFT
PROMPT_READY
LYRICS_READY
WAITING_FOR_AUDIO                (Stage A terminal)
AUDIO_RECEIVED
QC_PENDING
QC_PASS
QC_EXCEPTION_OWNER_AUTHORIZED
READY_FOR_UPLOAD                 (audio bytes verified in Drive)
READY_FOR_UPLOAD_VIA_URL         (controlled fallback only)
URL_DOWNLOAD_FAILED
UPLOADED_PROCESSING              (Hermes)
SCHEDULED                        (Hermes)
PUBLIC                           (Hermes)
BLOCKED
PUBLISHED_AND_MIRRORED           (Hermes)
```

Statuses from `UPLOADED_PROCESSING` onward are **Hermes-owned**; Claude writes them only when copying a Hermes report, and says so.

## ⚙️ Requirements

- Google Drive connector — `Oatto Music/` folder (Folder ID: *fill in once created*)
- `ffmpeg`/`ffprobe` for measured audio QC (check before assuming)
- Higgsfield MCP — optional, cover art only
- A music generator the user operates (Suno etc.) — **Claude does not generate the song**
- Hermes AI (พี่วอก) for YouTube/Facebook publishing
- No Telegram credential — Stage B drafts the notification for the user to send

## Related skills

- `oattomusic` — takes a finished Oatto song and produces cinematic Shorts footage. This skill is upstream: it produces the song package and the Hermes handoff; `oattomusic` can consume `song_package.json` (lyrics, mood, shorts_plan) once audio exists. They share the channel and the Hermes handoff but not code.
- `youtubeoat`, `meaw`, `longlifeheath` — sibling channel skills sharing the Fast Mode shape, the two/three invariants, and the Hermes split.

---

*Skill: oattosong | Oatto Music Song Planning & Packaging | Version 1.0*
*Created: 2026-10-06*
