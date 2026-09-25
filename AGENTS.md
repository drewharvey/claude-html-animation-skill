# Project Guidance

This is a Claude Code skill that acts as a pipeline:

1. Generates HTML animations that conform to the [h2v](https://github.com/drewharvey/html-to-video) authoring contract.
2. Drives h2v CLI commands to export those animations to MP4 or bulk-preview them in a browser.

h2v is the upstream this skill depends on. Many pieces of information in `SKILL.md` — meta tags, attributes, install commands, CLI flag examples, default values, `h2v review` behavior — were copied or derived from h2v's docs.

## Tasks

### Sync audit (review h2v for drift)

**Why:** h2v is under active development. Content in `SKILL.md` that was copied from h2v's docs drifts as h2v changes. Drift means the skill instructs Claude using stale or wrong information — broken commands, wrong defaults, missing features, divergent conventions. A periodic sync audit catches the drift before it leaks into user-facing animations.

**How:**

1. Fetch h2v's current docs from https://github.com/drewharvey/html-to-video. Start with the README; follow links to `docs/` as needed.
2. Walk `SKILL.md` identifying every piece of h2v-derived content. Categories to check:
   - Authoring contract: `h2v-duration` and `h2v-themes` meta tags, `data-h2v-hide`, `data-h2v-recording`, theme conventions, output filename suffixes
   - Install instructions
   - CLI examples and flag descriptions
   - Default values (viewport, fps, slowdown, output path)
   - `h2v review` behavior
   - Concurrency / memory guidance
3. Compare each piece against the current upstream. Note drifts with `file:line` references.
4. Surface findings to the user before editing. Don't apply changes unprompted.
5. When fixing drifts, fix the drifted *content* in this project. Don't retarget cross-repo links to deeper paths — see [Linking to h2v](#linking-to-h2v) below.

**Out of scope:** Original guidance authored by this skill — animation design principles, color systems, motion standards, typography, common-mistakes lists. None of that depends on h2v and none of it needs upstream review.

## Conventions

### Linking to h2v

When `SKILL.md` or `README.md` cross-references h2v, link to the top-level README at https://github.com/drewharvey/html-to-video — not to specific files under `docs/`. h2v's doc structure can change; deep links break when files are renamed or merged. The README is a stable entrypoint and the extra navigation hop is negligible. Cross-repo fallback lookups are infrequent enough that directness isn't worth the breakage risk.

If a sync audit finds drift, fix the drifted content here — don't migrate the link.
