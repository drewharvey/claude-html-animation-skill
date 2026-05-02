# Voice-over-to-video pipeline plan

This document is a hand-off brief for a Claude Code instance about to plan and implement one of the unbuilt skills in this pipeline. It contains the project context, per-skill specs, and the design questions left open. Read the whole file once before starting — the skills compose, so design choices in one affect the others.

## The broader goal

The user has a voice-over (mp3/wav) and wants animated visuals that align to it. Today this is a manual workflow: listen, decide what to show when, hand-author each clip, time-align, export, drop into the NLE. The pipeline collapses **everything up to the NLE step** into a Claude-driven flow, ending with **N drop-in MP4 clips** that the user assembles in their own video editing software (DaVinci, Premiere, Final Cut, etc.).

The orchestrator does **not** assemble a single output video, mux audio, or use ffmpeg. The user keeps creative control over transitions, audio mixing, additional B-roll, color grading, and any other post work in their NLE.

```
audio file
    ↓  whisper-transcribe skill
timestamped transcript (JSON segments with start/end seconds)
    ↓  orchestrator: storyboard segments + propose animations
proposed (start, end, idea) tuples covering the audio
    ↓  user reviews storyboard      ← APPROVAL CHECKPOINT
approved storyboard
    ↓  html-animation skill (per segment, sequential or batched)
N HTML animation files, each with h2v-duration matching its segment exactly
    ↓  user previews in browser; requests per-segment refinements   ← ITERATION LOOP
finalized HTML animations
    ↓  html-animation skill: h2v export (use --concurrency for batches)
N silent MP4 clips, named for ordered timeline drop-in
    ↓  user drags into their NLE alongside the voice-over track
finished video (assembled by the user)
```

## The three skills

| Skill | Status | Purpose |
|---|---|---|
| `html-animation` | **Exists** — this repo | Generate self-contained HTML/CSS/JS animations and drive `h2v` to export them as MP4. Already handles both single- and multi-animation runs correctly |
| `whisper-transcribe` | To build | Drive the OpenAI Whisper CLI to produce a timestamped transcript from an audio file |
| `voice-over-to-video` | To build | Thin orchestrator that wires the above two together — encodes the storyboard-then-execute workflow |

**Skills compose loosely.** There's no function-call mechanism between Claude Code skills; each is markdown guidance loaded into Claude's context when triggered. The orchestrator works by being explicit: it names the dependent skills, it uses language that matches their trigger descriptions, and it tells Claude to follow their conventions for the relevant steps. It does not duplicate their content.

## How Claude Code skills work (quick reference)

Each skill lives in its own directory with at minimum a `SKILL.md`:

```
~/.claude/skills/<skill-name>/SKILL.md
```

`SKILL.md` starts with YAML frontmatter that defines the trigger:

```markdown
---
name: <skill-name>
description: Short prose describing what the skill does and when to activate. Claude reads this on every prompt; words/phrases in the user's message that match this description activate the skill.
---

# Skill body — instructions Claude follows when activated
```

The body is loaded into context every time the skill triggers, so verbosity costs tokens on every invocation. Keep it prescriptive and terse.

Conventions established by this repo (recommend matching):

- One repo per skill, named `claude-<skill-name>-skill`, with `SKILL.md`, `README.md`, `LICENSE` at root.
- Install via clone or symlink into `~/.claude/skills/<skill-name>/` — see this repo's `README.md` for the pattern to mirror.
- Tone: prescriptive, comment-light, opinionated. Walk Claude through decisions before code patterns.
- Sync-audit pattern: when a skill consumes an external tool's contract, periodically re-fetch that tool's docs and compare against the skill's content to catch drift. See this repo's `CLAUDE.md` for the workflow.

---

## Skill brief: `whisper-transcribe`

### Goal

Take an audio file path; produce a timestamped transcript downstream tooling can consume.

### Trigger

Activate on: "transcribe this audio", "get a transcript with timestamps", "convert this voice memo to text", "what does this recording say", and similar.

Do not activate on: live transcription, music analysis (genre/beat detection), text-to-speech, audio editing.

### External dependency

OpenAI Whisper CLI — https://github.com/openai/whisper

- Local, Python-based, no API
- Install: `pip install -U openai-whisper` (also requires `ffmpeg` on PATH for audio decoding)
- Models: `tiny`, `base`, `small`, `medium`, `large` (and `*.en` English-only variants). Larger = more accurate, slower, more VRAM/RAM
- Output formats: `txt`, `vtt`, `srt`, `tsv`, `json`. **JSON is canonical** — per-segment text with `start` / `end` timestamps in seconds, the format the orchestrator consumes
- GPU via PyTorch CUDA (auto-detected); CPU fallback works but is slow at `medium`+

### Behavior contract

- **Install check first.** Mirror the `command -v h2v` pattern from this repo's `SKILL.md` (see `### Before running: check it's installed`). Don't `pip install` without confirmation — it mutates the user's environment.
- **Model selection.** Default to `base` (balanced speed/quality). Bump to `small` / `medium` if the user mentions accuracy, non-English language, or noisy/accented audio. Don't default to `large` — slow, rarely needed.
- **Always emit JSON.** Pass `--output_format json` so downstream consumers (the orchestrator) get structured data instead of formatted text.
- **Output directory hygiene.** Whisper writes to cwd by default; pass `--output_dir` to keep the user's working directory clean.
- **Language hint.** If the user mentions a language, pass `--language <code>`; otherwise let Whisper auto-detect (slower but robust).
- **Sync audit.** Periodically re-fetch the Whisper README and compare CLI surface (model names, flags, JSON schema) against this skill's content. Same pattern as this repo's `CLAUDE.md`.

### Open questions to resolve during planning

- Single-command surface (`transcribe <file>`) vs. multi-flag exposure (model, language, format, output dir)? Lean toward minimal surface in `SKILL.md`, expand only when needed.
- Long-file handling: chunk? warn about runtime? Whisper handles long files internally but a 2-hour podcast on `medium` CPU is *very* slow.
- WhisperX (forced-alignment word-level timestamps) vs. plain Whisper? More accurate per-word but adds dependency complexity. Probably skip unless the orchestrator turns out to need word-level timing.

### Useful starting reading

- Whisper README: https://github.com/openai/whisper
- This repo's `SKILL.md` — the `## Video export` → `### Before running: check it's installed` section is the install-check pattern to mirror
- This repo's `CLAUDE.md` — sync-audit workflow

---

## Skill brief: `voice-over-to-video`

### Goal

A **thin orchestrator** that turns a voice-over audio file into a directory of timing-accurate MP4 clips the user can drop into their NLE alongside the original audio. The skill itself does almost no original work — its job is to wire the existing skills together correctly.

This is explicitly **not** a renderer. It does not produce a single assembled MP4, it does not mux audio, it does not use `ffmpeg`. The user assembles the final video in their own editing software.

### Trigger

Activate on: an audio file is referenced or attached **and** the user asks for animations, B-roll, visuals, or a video to accompany it. Phrases like "make animations to go with this voice-over", "give me ideas for visuals that sync to this narration", "I have a voiceover, generate animations for it", "create b-roll for this audio".

Do not activate on: requests to make a single animation with no audio (that's the `html-animation` skill alone) or to just transcribe (that's `whisper-transcribe` alone).

### Dependencies

This skill is a workflow wrapper around two other skills. Both must be installed:

- `whisper-transcribe` skill at `~/.claude/skills/whisper-transcribe/`
- `html-animation` skill at `~/.claude/skills/html-animation/`

Plus the underlying CLIs those skills require: `whisper` (and Python/`pip`), `h2v` (and `node`/`ffmpeg`).

### Behavior contract — the workflow phases

**Phase 0 — Dependency check.** Before doing anything substantive, verify both dependent skills are installed (`ls ~/.claude/skills/whisper-transcribe/SKILL.md` and same for html-animation). If either is missing, stop and tell the user where to install it from. Don't attempt to fall back or duplicate the missing skill's content.

**Phase 1 — Transcribe.** Defer to the `whisper-transcribe` skill. The orchestrator's body should reference it explicitly ("use the `whisper-transcribe` skill to generate a JSON transcript with timestamps") — language that matches whisper-transcribe's trigger description so its body activates and its conventions apply (model selection, install check, output format).

**Phase 2 — Storyboard (the only real work this skill does).** Read the JSON transcript. Segment the audio into *scenes* — usually larger than a Whisper segment (which is sentence-ish). Use semantic clustering: a 30-second explanation of one concept is probably one scene, not seven. Output a proposed list of `(start_time, end_time, animation_idea)` tuples covering the full audio with no gaps. The animation_idea is a short description ("a grid of servers lighting up one by one", "a progress bar filling and stalling at 80%") not full implementation detail.

**Phase 3 — Storyboard approval (HARD CHECKPOINT).** Show the proposed storyboard to the user. Wait for explicit approval, modification, or revision requests before proceeding to generation. Do not generate animations off an unapproved storyboard — generation is the expensive step and a misaligned storyboard wastes minutes of wall time and a lot of tokens.

**Phase 4 — Generate.** For each approved tuple, defer to the `html-animation` skill, passing the segment duration as the target. The `html-animation` skill already handles 1-to-N animation requests correctly — for N≥2 it produces a clean directory of files with descriptive names. Two constraints the orchestrator must communicate downstream:

- **Exact duration.** Each animation's `<meta name="h2v-duration">` must equal the segment's `(end - start)` to sub-second precision. `4.7s` not `5s`. The clip's duration is what makes it slot into the user's NLE timeline correctly.
- **No end hold.** The `html-animation` skill defaults to a 1–2s hold at the end. For voice-over-aligned clips this is wrong — the next segment must start exactly when the previous ends. Tell `html-animation` to skip the final hold (or compress it to zero) for these animations.
- **Visual continuity.** `html-animation`'s `## Multi-animation runs` section already prescribes matching palette, typography, and motion language across a set. Reinforce this — the clips will be edited together, so they must read as one coherent piece.

**Phase 5 — Iteration (HARD REQUIREMENT).** After generation, the user previews HTML files in the browser and may request per-segment refinements: *"redo segment 5 with a different idea", "make segment 3's color match segment 2", "the timing on 7 feels rushed"*. The orchestrator supports cheap per-segment regeneration without re-running phases 1–3. Loop until the user says they're done.

**Phase 6 — Export.** Defer to the `html-animation` skill's export pattern — `h2v export <directory>` with `--concurrency` sized to the user's RAM (see `html-animation`'s SKILL.md for the table). Output is N silent MP4 files named in transcript order so they sort correctly when the user multi-selects and drops them onto an NLE timeline.

### File naming convention

The export step needs to produce files that sort correctly in alphabetical/Finder order so a user can multi-select all of them in their file browser and drag them onto the NLE timeline in the right sequence. Use a numeric prefix matching the storyboard order: `01-<descriptive-name>.mp4`, `02-<descriptive-name>.mp4`, etc. The descriptive name comes from the storyboard's animation idea, not from the transcript text.

### Out of scope (do not implement)

- Single assembled output MP4
- Audio muxing / merging the voice-over track with the video
- Drift management between video clips and audio (the user's NLE handles this)
- Music or SFX track support
- `ffmpeg` integration of any kind
- Live preview of the assembled timeline (use `h2v review` if a preview is wanted, but it shows clips side-by-side, not on a timeline)

If the user asks for any of the above, tell them this skill stops at producing per-segment clips; assembly is intentionally their NLE's job.

### Open questions to resolve during planning

- **Long-segment pacing.** When a transcript segment runs >15s, single-shot animations get boring. Does the orchestrator sub-divide the segment into multiple animations, or instruct `html-animation` to use sub-phasing within a single longer animation?
- **Silence handling.** Whisper's segments correspond to speech; gaps between segments are pauses. Does the orchestrator extend the previous animation through the pause, generate an "ambient" animation, or ask?
- **Visual direction inheritance.** Does the orchestrator ask the user upfront for a palette/mood (then pass it to every `html-animation` invocation), infer from the transcript content, or let `html-animation` pick its defaults? The first probably wins for coherence — but only ask once at the start, not per-segment.
- **Re-storyboard after audio change.** If the user re-records the voice-over and re-runs the skill, can the orchestrator diff old vs new transcripts and only regenerate changed segments? Probably overkill for v1; just regenerate from scratch.

### Useful starting reading

- This repo's `SKILL.md` — especially `## Multi-animation runs` (visual continuity, file naming) and `## Video export` (`h2v-duration` semantics, `h2v export` with `--concurrency`, batch handling)
- This repo's `CLAUDE.md` — sync-audit workflow, link-stability convention
- h2v: https://github.com/drewharvey/html-to-video — `docs/authoring.md` (meta tags, theme contract) and `docs/cli.md` (CLI flags, output paths, parallel recording)
- Whisper: https://github.com/openai/whisper — JSON output schema specifically

---

## Project-level decisions still open

- **Inline Whisper or extract it?** This document assumes extraction (3-skill model). Break-even is whether the user can name a second consumer for transcription (subtitles, meeting notes, podcast indexing). If yes, extract. If voice-over-to-video is the only ever consumer, fold it inline and drop the third skill.
- **Repo strategy.** Each skill in its own repo (matching this project) keeps install/sync-audit uniform; a monorepo with subdirectories is also defensible. Pick based on whether you expect the skills to evolve at very different rates.
