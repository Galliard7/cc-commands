---
description: "Create stunning HTML slide decks — supports company branding via URL extraction"
---

# /slides — HTML Presentation Generator

You are creating a presentation using the **frontend-slides** skill. This command extends the skill with brand-aware style generation.

The frontend-slides skill is installed at `~/.claude/skills/frontend-slides/`. Its `SKILL.md` defines the full phase workflow (Phases 0–5). **Follow that workflow exactly**, with the modifications below.

## Argument Parsing

The user may invoke this command with optional arguments:

- `/slides` — no args, run the standard frontend-slides workflow
- `/slides <topic>` — pre-fill Phase 1 with the topic (still ask remaining questions)
- `/slides --brand <url>` — extract brand identity from the URL and use it in Phase 2
- `/slides --brand <url> <topic>` — both brand extraction and topic

## Brand Extraction (when `--brand` is provided)

**Insert this step between Phase 1 and Phase 2.** This replaces the mood-based style discovery with brand-driven style generation.

### Step B1: Fetch and Analyze

Use WebFetch to retrieve the provided URL. Extract:

1. **Colors** — Primary, secondary, accent, background, text colors. Look for CSS custom properties (`:root` vars), dominant colors in the design, logo colors, button/CTA colors.
2. **Typography** — Font families used for headings and body. Check `font-family` declarations, Google Fonts / Typekit imports.
3. **Visual Tone** — Is it minimal? Bold? Corporate? Playful? Techy? Note spacing density, use of whitespace, imagery style, border radius patterns.
4. **Logo** — If visible and identifiable, note it for potential inclusion.

### Step B2: Generate Brand Style Spec

Create a style specification in the format of STYLE_PRESETS.md entries:

```
**Vibe:** [derived from visual tone]
**Typography:**
- Display: [heading font from site] ([weight])
- Body: [body font from site] ([weight])
**Colors:**
:root {
    --bg-primary: [extracted];
    --bg-secondary: [extracted];
    --text-primary: [extracted];
    --text-secondary: [extracted];
    --accent: [extracted];
    --accent-secondary: [extracted];
}
**Signature Elements:** [derived from the site's design language]
```

### Step B3: Show Brand Preview

Generate **one** single-slide HTML preview using the extracted brand spec — save to `.claude-design/slide-previews/brand-preview.html` and open it.

Ask the user (header: "Brand Style"):
> Here's how your slides will look based on [company]'s brand. Options:
> - "Looks great" — proceed to Phase 3
> - "Adjust colors/fonts" — user provides tweaks
> - "Ignore brand, show me presets instead" — fall back to standard Phase 2

### Step B4: Proceed

If approved, skip Phase 2 entirely and go straight to Phase 3 using the brand style spec.

## Output Location

Save presentations to the current working directory unless the user specifies otherwise. Use a descriptive filename: `{topic-slug}-deck.html`.

## After Generation

After Phase 5 delivery, remind the user:
- To iterate on the deck, just describe changes — CC will edit the HTML directly
- To re-brand for a different company: `/slides --brand <new-url>`
- The HTML is self-contained — works offline, can be shared as a single file, or hosted on GitHub Pages
