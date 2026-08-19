+++
title = 'Turning a Repeated Prompt into a Claude Code Plugin'
date = 2026-08-19T09:00:00+01:00
draft = false
+++

I build a fair number of internal decks — sprint show-and-tells, architecture walkthroughs, the occasional "here's why this took three weeks" — and they all need to look like they came from the same place. For a while my process was to open [Claude Code](https://www.anthropic.com/claude-code), paste a long-winded description of our house style, and hope. It worked about as well as you'd expect. The type scale drifted, slide numbering restarted halfway through, and roughly every third attempt decided that six consecutive bullet-point slides were a reasonable thing to hand a room full of people.

So I turned the prompt into a Claude Code **skill**, and then packaged the skill as a **plugin**. The interesting part turned out to have very little to do with deck styling, and quite a lot to do with where you draw the line between what the model decides and what a plain script does.

The repo lives on an internal org, so there's no public source link this time — everything that matters is inlined below.

## Overview

- **A skill, not a prompt**: a single `SKILL.md` describing the workflow, which Claude loads on demand when I ask for slides.
- **One self-contained HTML file**: fonts embedded as base64, zero external requests — it renders identically on a machine with no route to the internet.
- **A closed set of components**: the model picks from a catalogue of ten or so styled components rather than inventing CSS.
- **Validation in the tool**: an 82-line Python assembler that refuses malformed input, instead of a prompt politely asking for well-formed input.
- **It checks its own work**: the deck is rendered headlessly, screenshotted, and read back before I ever see it.
- **Packaged as a plugin**: installable with `/plugin install`, with a plain shell installer as a fallback.

## What a skill actually is

A skill is a directory with a `SKILL.md` in it. The frontmatter is the whole trick — the `description` is what Claude matches against when deciding whether the skill is relevant, so it needs to enumerate the ways a person might ask:

```yaml
---
name: presentation-deck
description: Generate a Royal Navy / MDDB-styled HTML presentation deck — a single
  self-contained file with embedded fonts, no external requests, plus an optional
  1280x720 PDF. Use when the user asks to "make a deck", "build slides", "create a
  presentation", "put some slides together", wants to present work to a team or
  stakeholders, or runs /presentation-deck.
version: 1.0.0
---
```

The body is the workflow: establish the topic, read the component reference, plan the narrative, write the slides, build, verify. Six steps, and the ordering matters more than the prose. Notably, step three ends with *wait for the user's go-ahead on the outline before writing the full deck* — a 12-slide deck is expensive to regenerate and cheap to re-plan, so the approval gate sits before the writing, not after.

The instruction I'd least want to lose is in step one:

> **Never invent specifics.** Every number, flag, filename and command in the deck must be real. If a fact cannot be established, leave it out rather than guessing. A deck full of plausible-sounding invented detail is worse than a short one.

A deck is exactly the kind of artefact where a confident fabrication survives review, because nobody fact-checks a bullet point on a slide. Being told to go and read the git log, the ticket or the script's `--help` first — and to omit rather than guess — does more for the output than any amount of styling guidance.

## Judgement in the model, assembly in a script

This is the bit I'd carry to any other skill I write. There are two very different kinds of work in "make me a deck":

**Judgement.** What's the arc? Which of these facts earns a slide? Is this a process flow or a before/after? That's genuinely a language problem, and the model is good at it.

**Assembly.** Wrap the fragment in a template, inject the CSS, escape the title, count the slides, shell out to headless Chrome for a PDF. That's deterministic, tedious, and something a model can only get *slightly* wrong in a way you won't notice until you're presenting.

So the model writes nothing but the slide bodies, to a fragment file — no `<html>`, no `<head>`, no `<style>`, no `<script>`. A small Python script does the rest:

```python
deck = (
    TEMPLATE.read_text()
    .replace("{{DECK_TITLE}}", html.escape(args.title))
    .replace("/* INJECT_EXTRA_CSS */", "\n".join(css_parts))
    .replace("<!-- INJECT_SLIDES -->", slides)
)
```

Three string replacements. Standard library only, nothing to `pip install`. The reason this matters is subtractive rather than additive: because the assembler is boring, none of its failure modes are interesting. It cannot forget the print stylesheet, hallucinate a font stack, or drop the navigation JavaScript, because it isn't deciding any of those things.

![A slide from the generated deck showing a four-step process flow: agree the outline, write the fragment, build, then render and check](/images/presentation-deck-pipeline.png)

## A closed vocabulary beats an open one

The single biggest improvement in consistency came from writing a `COMPONENTS.md` and telling the skill to read it *in full* before writing any markup. It catalogues every available component with a copy-paste snippet: bulleted points, a card grid, a process flow, two-column before/after panels, big statistics, tag pills, a terminal block, check lists, verdict cards, a callout.

Each entry also carries its own limits, which is where the real value is:

```html
<div class="flow">
  <div class="step">
    <div class="k">Trigger</div>
    <div class="t">Labelled PR merged</div>
    <div class="d">major / minor / patch drives the bump</div>
  </div>
  <div class="arrow">&rarr;</div>
  <!-- ... -->
</div>
```

> `.k` = stage/actor label (uppercase accent), `.t` = what happens, `.d` = detail.
> 3–4 steps; more than 4 will wrap and look cramped.

"More than 4 will wrap and look cramped" is worth ten lines of aesthetic instruction. So is `4–5 items maximum` on the bullet list, and `Designed for exactly three` on the verdict cards. The model isn't being asked to develop taste — it's being told the constraints that taste would have produced, which is a far more reliable thing to communicate.

The other line that pulls its weight sits in the workflow rather than the catalogue: *vary the components — several identical slides in a row is the main way these decks go wrong.* Naming the failure mode explicitly turned out to be the fix for it.

## Guardrails belong in the tool, not the prompt

The deck template hides every slide except the one carrying `active`, and initialises from it. A fragment where nothing is marked `active` produces a technically valid HTML file that opens completely blank — which is a horrible thing to discover with a laptop plugged into a projector.

You could ask the model very nicely to always remember the `active` class. Better to make the build fail:

```python
slides = Path(args.slides).read_text()
if 'class="slide' not in slides:
    sys.exit("error: --slides file contains no .slide sections")
if "active" not in slides:
    sys.exit("error: no slide carries the 'active' class — the first slide must have it")
```

Two checks, four lines, non-zero exit. An agent that can read a failing command's output will fix its own mistake and move on without being told; a prompt-level instruction just quietly decays.

![Terminal recording of build.py assembling a five-slide deck and PDF, then rejecting a fragment where no slide is marked active and exiting 1](/images/presentation-deck-build.gif)

## One file, no network

Every deck comes out as a single HTML file with Lato 400/700/800 embedded as base64 woff2. That puts a deck at around 725 KB, which is an unfashionable number for something with five slides in it.

It's deliberate. Our work distributes data to ships over satellite links, and the machines a deck gets shown on are not reliably ones with a route to the public internet. A deck that fetches a webfont at load time is a deck that renders in Times New Roman at the worst possible moment. So the `COMPONENTS.md` conventions section says, flatly, *never add a webfont link* — and the 725 KB buys a file I can email, drop on a USB stick, or open on a locked-down machine and know exactly what it will look like.

![Title slide of a generated deck, dark navy gradient with the Royal Navy brandmark, reading "One file, no network."](/images/presentation-deck-title-slide.png)

Slides are 1280×720 and sized in `vw`/`vh` rather than pixels, so the same markup scales to any screen and to the PDF. `--pdf` prints through headless Chrome at 960×540pt — exactly 1280×720px at 96dpi.

## Making it look at its own work

Content on a slide is *clipped* at 720px, not scrolled. That's the right call for a presentation, but it means overflow is completely invisible unless something actually renders the thing. A model that has only ever seen its own markup has no idea it wrote one bullet too many.

So the final step of the skill is to render, screenshot, and look. The awkward part is that only the `active` slide is visible, so to inspect slide four you have to move the class and neutralise the template's `show(0)` initialiser:

```python
h = Path('deck.html').read_text()
h = h.replace(
    '<section class="slide title active">', '<section class="slide title">', 1
)
h = h.replace('<section class="slide">', '<section class="slide active">', 1)
Path('/tmp/preview.html').write_text(h.replace('show(0);', ''))
```

Then Chrome headless with `--screenshot --window-size=1280,720`, and read the PNG back:

```bash
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"$CHROME" --headless --disable-gpu \
  --screenshot=/tmp/preview.png --window-size=1280,720 /tmp/preview.html
```

The check is specific — nothing overflowing the frame, no colliding text, brandmark present, type scale sane — and so is the remedy: *fix anything wrong, usually by splitting a slide, not by shrinking text.* Without that last clause the instinct is to drop the font size a point and call it solved, which is how you end up with a deck nobody at the back can read.

![A slide from the generated deck showing a dark terminal block with the build command, a failed run exiting 1, and a successful run](/images/presentation-deck-terminal.png)

Every screenshot in this post came out of exactly that loop, which is a reasonable sanity check on whether the loop works.

## Packaging it as a plugin

A skill on its own lives in `~/.claude/skills/`. To make it installable, you add a `.claude-plugin/plugin.json`:

```json
{
  "name": "presentation-deck",
  "version": "1.0.0",
  "description": "Generate self-contained, offline-ready HTML presentation decks...",
  "keywords": ["presentation", "slides", "deck", "html", "royal-navy", "mddb"]
}
```

…and a `marketplace.json` alongside it, so the repo can serve as its own marketplace:

```json
{
  "name": "presentation-deck-marketplace",
  "plugins": [
    { "name": "presentation-deck", "category": "productivity", "source": "./" }
  ]
}
```

Which makes installation two lines:

```text
/plugin marketplace add <org>/presentation-deck
/plugin install presentation-deck
```

The gotcha is asset paths. A plugin's files are addressed relative to `${CLAUDE_PLUGIN_ROOT}`, so `SKILL.md` refers to its template as `${CLAUDE_PLUGIN_ROOT}/skills/presentation-deck/template.html`. Install the same skill *directly* — by copying it into `~/.claude/skills/` — and that variable simply isn't set, so every path in the file silently points at nothing.

Rather than making `SKILL.md` handle both cases, the plain installer rewrites the variable out at install time:

```python
skill = dest / "SKILL.md"
text = skill.read_text().replace(
    "${CLAUDE_PLUGIN_ROOT}/skills/presentation-deck", str(dest)
)
skill.write_text(text)
```

One file has one addressing scheme, and the installer resolves it. It also checks for `python3`, and warns — rather than fails — if it can't find Chrome, Edge or Chromium, since that only affects PDF export and screenshots. The HTML builds regardless.

## What I'd take to the next one

Three things generalise well beyond decks.

**Put the deterministic half in a real CLI.** `build.py` is a normal script with `--help` that I can run by hand, which means the skill is debuggable without an agent in the loop at all. If the deck is wrong, I can bisect: bad fragment, or bad assembly?

**Encode limits, not aspirations.** "3–4 steps; more than 4 will wrap" beat every attempt I made at describing what good looks like.

**Give it a way to see the result.** Rendering the output and reading it back caught more real problems than anything else in the workflow — and it's the step most easily left out, because everything appears to have worked without it.

None of this is specific to Claude Code, really. It's the same instinct as any good automation: work out which part of the job genuinely needs judgement, do that part properly, and refuse to let it anywhere near the part that doesn't.
