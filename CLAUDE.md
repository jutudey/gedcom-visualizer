# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A browser-based GEDCOM ancestor tree visualizer. No build system, no npm, no external dependencies. Everything runs client-side.

## How to run

Serve the directory with any local HTTP server (required for `fetch('sample.ged')` to work):

```bash
python3 -m http.server 8080
# then open http://localhost:8080/Gedcom%20Visualizer.html
```

Opening `Gedcom Visualizer.html` directly as a `file://` URL will work except the sample file auto-load will silently fail (CORS).

## File roles

| File | Purpose |
|------|---------|
| `Gedcom Visualizer.html` | The sole source file. All CSS, HTML, and JS live here. |
| `sample.ged` | Demo GEDCOM file fetched on page load. |
| `uploads/` | User-uploaded GEDCOM files (not tracked). |

## Architecture of `Gedcom Visualizer.html`

The file is a single self-contained HTML page. The JavaScript is organized into numbered sections:

**1. GEDCOM parser** (`parseGedcom`)
Two-pass approach: first builds a hierarchical record tree from raw lines, then extracts two `Map`s:
- `indis: Map<xref, { id, name:{full,given,surname}, sex, birth, death, famc[], fams[] }>`
- `fams: Map<xref, { id, husb, wife, chil[], marr }>`

CONT/CONC continuation lines are folded into their parent node's value in the first pass.

**2. Tree builder** (`buildAncestorTree`)
Recursive traversal starting from a proband ID, up to `maxGens` generations. Returns a binary tree of `{ person, gen, father, mother }` nodes.

**3. Layout engine** (`layoutTree`)
Horizontal left-to-right layout: proband in column 0, parents in column 1, grandparents in column 2, etc. Two rendering modes per generation:
- Normal: father above mother in the same column, joined by a bracket line
- Stacked (`STATE.stack[g] = true`): parents overlay the child's column vertically (child sits between them), connected by straight vertical lines at 75% across the child card

All per-generation sizing is driven by 8-element arrays in `STATE` (`colWidth`, `nodeW`, `nodeHeight`, `vGap`, `wrap`, `armLen`).

**4. Renderer** (`render`)
Builds an SVG element. Person cards are rendered via `<foreignObject>` containing HTML `<div class="node-card">` elements (not SVG shapes). This allows CSS text wrapping. The SVG is mounted into `#viewport` inside `#stage`.

**5. Pan/zoom**
Pointer capture on `#stage` for drag-to-pan. Wheel zoom around cursor using exponential scaling. Transform applied to `#viewport` via CSS `transform: translate() scale()`.

**6. PNG/SVG export**
Attempts canvas rasterization (2×). Falls back to downloading the SVG directly if `foreignObject` causes canvas tainting (common in sandboxed contexts).

**7. Sidebar, search, and file loading**
`STATE` is the single source of truth. `render()` is called on every state change. `setProband(id)` switches the focused person and re-renders. The proband popover (hover) shows spouses and children and allows navigating to them.

## Key state object

```js
STATE = {
  data: { indis, fams, head },  // parsed GEDCOM
  probandId: string,             // currently focused person's xref
  maxGens: 2–25,
  showYears, colorSex, showSpouses,
  // per-generation arrays (index 0 = proband column, length 25):
  colWidth[], nodeW[], nodeHeight[], vGap[], wrap[], armLen[], stack[],
  view: { x, y, scale }         // pan/zoom state
}
```

## CSS design system

All styles are inline in the `<style>` block. Colors use `oklch()`. Key custom properties:

```css
--bg, --panel, --ink, --ink-soft, --ink-faint
--rule, --card, --card-bd, --hairline
--accent, --accent-soft
--male-tint, --female-tint, --proband
```

Node cards use `.node-card` with optional `.male`, `.female`, `.proband` modifiers. The proband card gets a green border (`--proband`).
