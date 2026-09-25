# html-animation

A Claude Code skill that does two things: it gets Claude to produce high-quality, self-contained HTML/CSS/JS animations, and it lets you export those animations as MP4 videos. Install it once and it triggers automatically whenever you ask Claude to create an animation, motion graphic, or animated visualization.

The HTML output is the iteration surface — standalone files you preview and refine in a browser. Video export is handled by the [html-to-video](https://github.com/drewharvey/html-to-video) CLI (`h2v`), which the skill knows how to drive. Every animation it produces ships `h2v`-ready, with the duration and theme metadata the renderer needs baked in, so exporting is a one-line ask.

The normal loop: ask Claude to create an animation (or a batch of them), iterate in the browser until they look right, then ask Claude to export the finals as video.

## What it does

When you ask Claude Code to create an animation — "animate a deployment sequence," "show a progress bar filling up," "visualize data flowing through a pipeline" — this skill activates and guides Claude to produce animations with:

- A proper color system using CSS custom properties (no hardcoded hex values)
- A single palette by default, with multiple switchable themes (light/dark, brand variants, custom palettes) added when you ask for them
- Staggered entrance animations instead of everything appearing at once
- Smooth easing curves and intentional timing
- Layered surfaces with depth (borders, shadows, elevation)
- Minimal on-screen text (visuals tell the story, not captions)
- A consistent controls bar (Reset, plus a theme picker when the animation has multiple themes)
- A clean, self-contained single HTML file with no external dependencies
- `h2v`-ready metadata, so any animation can be exported to MP4, MOV (including transparent/alpha), WebM, or GIF on request

## Install

The skill's `name:` field is `html-animation`, so it must be installed at `~/.claude/skills/html-animation/` regardless of what this repo's directory is called.

These instructions assume macOS or Linux with a POSIX shell (zsh, bash). On Windows, use WSL or substitute the equivalent PowerShell commands (`New-Item -ItemType SymbolicLink`, etc.).

If you've never installed a Claude Code skill before, the skills directory may not exist yet. Create it once:

```bash
mkdir -p ~/.claude/skills
```

### For personal use (all projects)

The simplest install — clone directly into your skills directory with the correct target name:

```bash
git clone https://github.com/drewharvey/claude-html-animation-skill.git ~/.claude/skills/html-animation
```

If you want to edit the skill while it's installed, clone anywhere and symlink it into the skills directory. **The `cd` step matters** — without it, `$(pwd)` won't point at the cloned repo and the symlink will be broken:

```bash
git clone https://github.com/drewharvey/claude-html-animation-skill.git
cd claude-html-animation-skill
ln -s "$(pwd)" ~/.claude/skills/html-animation
```

If you'd rather use an absolute path (no `cd` required):

```bash
ln -s /absolute/path/to/claude-html-animation-skill ~/.claude/skills/html-animation
```

Either way, the symlink means edits to `SKILL.md` take effect immediately — Claude Code reads the file fresh each time the skill is invoked.

To verify the symlink is good, run `ls -la ~/.claude/skills/html-animation/` — you should see `SKILL.md` listed. If you get "No such file or directory," the symlink is broken (it points at a path that doesn't exist); `rm` it and try again.

### For a specific project

Put the skill inside your project's `.claude/skills/` directory. Anyone who clones the repo gets the skill automatically:

```bash
mkdir -p your-repo/.claude/skills/html-animation
cp SKILL.md your-repo/.claude/skills/html-animation/
```

### After installing

**Start a new Claude Code session.** Skills are registered when a session starts — an existing session won't pick up a new install. Once registered, edits to `SKILL.md` take effect on the next skill invocation without needing another restart.

## Update

How you update depends on how you installed.

**Symlinked install** — pull in your clone; the symlinked install picks up the new `SKILL.md` immediately:

```bash
cd /path/to/claude-html-animation-skill   # wherever you cloned
git pull
```

**Direct clone into skills directory** — pull from inside the install location:

```bash
cd ~/.claude/skills/html-animation
git pull
```

**Project-scoped install (copied `SKILL.md`)** — copy the updated file in and commit:

```bash
# from the cloned repo:
cp SKILL.md /path/to/your-repo/.claude/skills/html-animation/
```

## Uninstall

**Symlinked install** — remove the link only (your clone stays put):

```bash
rm ~/.claude/skills/html-animation
```

**Direct clone into skills directory** — delete the installed copy:

```bash
rm -rf ~/.claude/skills/html-animation
```

**Project-scoped install** — delete the skill directory from the project and commit so teammates' next pull removes it on their side too:

```bash
rm -rf your-repo/.claude/skills/html-animation
```

## Verify it works

Start a new Claude Code session and try:

```
Create an animation showing a grid of servers going offline one by one
```

Claude should produce a single HTML file with a dark palette, staggered animations, CSS custom properties for all colors, an `h2v-duration` meta tag, and a Reset button in the top-right controls bar. The file should open in your browser automatically; if it doesn't, Claude prints the path so you can open it yourself.

To check theming, ask for variants: `...with a light and dark mode`. The controls bar should now include a swatch picker.

To check export (requires `h2v`, see [Video export](#video-export)): `Export that animation to video`.

## What the skill covers

The `SKILL.md` instructs Claude on:

**Design thinking** — Define the story arc, mood, what earns its place on screen, and (if the animation may become a video) the recording method before writing code.

**Recording method** — Two ways to author an animation for export. *Play-driver* uses the normal web idiom (CSS keyframes, transitions, `setTimeout`) and `h2v` records it by slowing the page clock; it's quick to author, but timing can drift under load. *Seek-driven* exposes a deterministic `window.seek(ms)` function that `h2v` calls once per frame; it's frame-perfect and safe at high concurrency, at the cost of interpolating motion by hand. When export is in scope, Claude asks which you want and defaults to seek.

**Color system** — A single palette using CSS custom properties with semantic color assignments (accent, success, warning, error) and surface layering for depth. A cinematic dark palette is the default starting point.

**Multiple themes** — Opt-in. When you ask for light/dark, brand variants, or any set of palettes, Claude adds per-theme palette blocks on a `data-theme` attribute, an `h2v-themes` meta tag, a swatch picker in the controls bar, `sessionStorage` persistence across Reset, a `postMessage` listener for iframe embedding, and a guard that disables theme transitions during recording.

**Typography** — System fonts for UI text, monospace for data and labels. No external font loading. Clear size hierarchy.

**Layout** — Centered viewport, fixed pixel dimensions, generous spacing, consistent border-radius scale, subtle shadows.

**Motion** — Entrance animations (opacity + translateY), transitions for state changes, proper easing curves (no `linear`), staggered timing, and a hold at the end state. The same timing values apply to seek-driven animations, just applied through `seek(ms)` instead of CSS and timers.

**Sequencing** — Phased `setTimeout` orchestration with commented timing structure (play-driver), or a timeline of eased segments inside `seek(ms)` (seek-driven).

**File structure** — Complete HTML template with the palette, `h2v-duration` meta, and a controls bar marked `data-h2v-hide` so it's hidden during capture.

**Multi-animation runs** — Requests like "create 5 animations for X" produce one file per animation in a dedicated directory, with descriptive names and a shared visual style.

**Video export** — How to drive `h2v`: install check, single-file and bundle export, per-theme export, custom viewports (`h2v-viewport`), quality presets, alpha/transparent `.mov` for compositing, other codecs (h264, ProRes, WebM), GIF, and concurrency. Export only runs when you explicitly ask for it.

**Previewing** — Opens a newly created file automatically, reminds you to refresh after edits, and uses `h2v review` for multi-file runs (a single live-reloading page, or a portable snapshot with `--out`). Say "don't open" or "just save" to skip.

**Common mistakes** — Explicit list of what to avoid: too much text, everything appearing at once, no end state, flat layouts, inconsistent radii, no visual hierarchy, hardcoded colors, missing hover states.

## Video export

Export is handled by [html-to-video](https://github.com/drewharvey/html-to-video) (`h2v`), which isn't on npm yet. You need Node 18+, `ffmpeg`, and Chrome/Chromium. Install from source:

```bash
git clone https://github.com/drewharvey/html-to-video.git
cd html-to-video
npm install
npm install -g .
```

If `h2v` is missing when you ask for an export, Claude offers to install it and waits for your confirmation first, since `npm install -g` changes your global environment.

Once it's installed, ask in plain language: "export to video", "render all themes as mp4", "export with a transparent background for After Effects", "make it a gif", "quick draft render". Output goes to `./output/` by default. Recording is slow (play-driver animations take about 6× their duration), so Claude never exports as part of creating an animation.

## Development

### File structure

```
claude-html-animation-skill/
├── SKILL.md       # The skill instructions (the only file required at install)
├── README.md      # This file
├── AGENTS.md      # Notes for agents editing this repo (sync audits, conventions)
├── CLAUDE.md      # Points Claude Code at AGENTS.md
└── LICENSE        # MIT
```

Only `SKILL.md` ends up at `~/.claude/skills/html-animation/` when installed — the rest is repo-level material for people working on the skill.

### Editing the skill

Edit `SKILL.md` directly. If you installed via symlink, changes take effect in your current Claude Code session without restarting.

Key areas you might want to customize:

- **Default color palette** — The skill includes a neutral blue accent. Change this to match your brand or preference.
- **Typography** — If you prefer a specific font stack, update the typography section.
- **Animation timing** — The defaults (300–600ms transitions, 60–150ms staggers) produce a balanced feel. Speed them up for snappier output, slow them down for more cinematic motion.
- **File template** — The controls bar (Reset, plus the theme picker for multi-theme animations) is baked into every animation. Modify the template if you want different controls.
- **h2v-derived content** — Install steps, CLI flags, defaults, and the authoring contract in `SKILL.md` are copied from h2v's docs and drift as h2v changes. See the sync-audit task in `AGENTS.md`.

### Testing changes

After editing `SKILL.md`, test with a few prompts:

```
Create an animation showing a progress bar that starts fast then stalls
```

```
Animate a file uploading with a percentage counter
```

```
Create a visualization of network packets flowing between three servers
```

Check that the output follows your updated guidelines. If Claude ignores a specific instruction, make it more prominent in the `SKILL.md` or add it to the "common mistakes" section as something to avoid.

### Skill triggering

The skill's `description` field in the YAML frontmatter controls when Claude activates it. The current description triggers on: animation, animated visualization, motion graphic, animated diagram, animated explainer, animated transition, and variations like "show me how X works" when motion adds value.

If the skill triggers too often or not enough, edit the `description` field. Making it more specific narrows triggering; making it broader widens it.

## Forking and sharing your own version

If you fork this skill, modify it, and want others to install your version instead of the upstream, point them at your fork.

### GitHub

Push your fork, then share the install command — substitute your GitHub username for `<your-username>`:

```bash
git clone https://github.com/<your-username>/claude-html-animation-skill.git ~/.claude/skills/html-animation
```

### Zip file

For distributing offline or outside GitHub. From the parent directory:

```bash
zip -r claude-html-animation-skill.zip claude-html-animation-skill/
```

Recipients unzip and rename the directory to match the skill name:

```bash
unzip claude-html-animation-skill.zip
mv claude-html-animation-skill ~/.claude/skills/html-animation
```

## Limitations

This skill improves the baseline quality of Claude's animation output, but it doesn't give Claude eyes. In claude.ai, the artifact preview creates an instant visual feedback loop — you see the animation and can say "the timing is off" or "the colors are wrong." In Claude Code, you preview in a browser and describe issues back to Claude. The skill closes the quality gap on first output but the iteration loop is still slower in CLI.

For best results: preview each animation in your browser, and when something needs fixing, describe the issue specifically — "the second element never appears," "the fade-in is too slow," "the red is too bright in light mode." Specific feedback produces specific fixes.

## License

MIT