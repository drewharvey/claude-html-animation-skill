# html-animation

A Claude Code skill that produces high-quality, self-contained HTML/CSS/JS animations. Install it once and it triggers automatically whenever you ask Claude to create an animation, motion graphic, or animated visualization.

## What it does

When you ask Claude Code to create an animation — "animate a deployment sequence," "show a progress bar filling up," "visualize data flowing through a pipeline" — this skill activates and guides Claude to produce animations with:

- A proper color system using CSS custom properties (no hardcoded hex values)
- Light and dark theme support via a toggle button
- Staggered entrance animations instead of everything appearing at once
- Smooth easing curves and intentional timing
- Layered surfaces with depth (borders, shadows, elevation)
- Minimal on-screen text (visuals tell the story, not captions)
- A consistent controls bar (Reset, Theme toggle)
- A clean, self-contained single HTML file with no external dependencies

## Install

### For personal use (all projects)

```bash
# Clone the repo
git clone https://github.com/yourname/html-animation.git

# Symlink into your Claude Code skills directory
ln -s "$(pwd)/html-animation" ~/.claude/skills/html-animation
```

The symlink means edits to `SKILL.md` take effect immediately — Claude Code watches for file changes without needing a restart.

Alternatively, copy instead of symlink:

```bash
cp -r html-animation ~/.claude/skills/html-animation
```

### For a specific project

Put the skill inside your project's `.claude/skills/` directory. Anyone who clones the repo gets the skill automatically:

```bash
mkdir -p your-repo/.claude/skills
cp -r html-animation your-repo/.claude/skills/html-animation
```

### For claude.ai (web interface)

Download the `html-animation.skill` file from the releases and upload it through **Customize > Skills > + Create skill** in the claude.ai interface.

## Verify it works

Start a new Claude Code session and try:

```
Create an animation showing a grid of servers going offline one by one
```

Claude should produce a single HTML file with a dark theme, staggered animations, CSS custom properties for all colors, and a theme toggle in the top corner. Open the file in a browser to preview.

## What the skill covers

The `SKILL.md` instructs Claude on:

**Design thinking** — Define the story arc, mood, and what earns its place on screen before writing code.

**Color system** — A full dark/light palette using CSS custom properties with semantic color assignments (accent, success, warning, error) and surface layering for depth.

**Typography** — System fonts for UI text, monospace for data and labels. No external font loading. Clear size hierarchy.

**Layout** — Centered viewport, fixed pixel dimensions, generous spacing, consistent border-radius scale, subtle shadows.

**Motion** — Entrance animations (opacity + translateY), CSS transitions for state changes, proper easing curves (no `linear`), staggered timing, and a hold at the end state.

**Sequencing** — Phased `setTimeout` orchestration with commented timing structure.

**File structure** — Complete HTML template with theme variables, controls bar, theme toggle, and `postMessage` listener for embedding in iframes.

**Common mistakes** — Explicit list of what to avoid: too much text, everything appearing at once, flat layouts, inconsistent radii, hardcoded colors, missing hover states.

## Development

### File structure

```
html-animation/
├── SKILL.md       # The skill instructions (this is the only required file)
└── README.md      # This file
```

### Editing the skill

Edit `SKILL.md` directly. If you installed via symlink, changes take effect in your current Claude Code session without restarting.

Key areas you might want to customize:

- **Default color palette** — The skill includes a neutral blue accent. Change this to match your brand or preference.
- **Typography** — If you prefer a specific font stack, update the typography section.
- **Animation timing** — The defaults (300–600ms transitions, 60–150ms staggers) produce a balanced feel. Speed them up for snappier output, slow them down for more cinematic motion.
- **File template** — The controls bar and theme toggle are baked into every animation. Modify the template if you want different controls.

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

## Sharing

### GitHub

Push this repo and others install with:

```bash
git clone https://github.com/yourname/html-animation.git ~/.claude/skills/html-animation
```

### Zip file

```bash
zip -r html-animation.zip html-animation/
```

Share the zip. Recipients unzip into `~/.claude/skills/`:

```bash
unzip html-animation.zip -d ~/.claude/skills/
```

### claude.ai upload

Package as a `.skill` file (just a zip with a different extension):

```bash
cd .. && zip -r html-animation.skill html-animation/
```

Upload through the claude.ai web interface under **Customize > Skills**.

## Limitations

This skill improves the baseline quality of Claude's animation output, but it doesn't give Claude eyes. In claude.ai, the artifact preview creates an instant visual feedback loop — you see the animation and can say "the timing is off" or "the colors are wrong." In Claude Code, you preview in a browser and describe issues back to Claude. The skill closes the quality gap on first output but the iteration loop is still slower in CLI.

For best results: preview each animation in your browser, and when something needs fixing, describe the issue specifically — "the second element never appears," "the fade-in is too slow," "the red is too bright in light mode." Specific feedback produces specific fixes.

## License

MIT