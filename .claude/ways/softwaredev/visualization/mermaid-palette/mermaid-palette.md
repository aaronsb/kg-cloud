---
description: kg-cloud dual-theme Mermaid palette, light and dark mode safe diagram colors, opaque node fills with per-fill text colors
vocabulary: mermaid diagram flowchart palette color light dark mode contrast cloudflare railway topology fill stroke
scope: agent, subagent
refire: 0.15
---
<!-- epistemic: convention -->
# kg-cloud Mermaid Palette Way

Project-local refinement of the global Mermaid Way. When you add or edit a
Mermaid diagram in this repo, use this exact palette so it reads cleanly in
both GitHub themes.

## The principle that makes it dual-theme

GitHub paints Mermaid against a white *or* near-black page depending on the
viewer's theme. Anything translucent or default-colored fails on one of them.
Two rules fix it:

1. **Node fills are fully opaque, with text color chosen per fill.** The text
   sits on the node, not the page, so contrast is theme-independent. Dark text
   (`#1a1a1a`) on bright fills (orange/amber); white (`#ffffff`) on deep fills
   (violet/slate).
2. **Subgraph titles/borders use mid-tones, never full-saturation brand
   colors.** Subgraph labels paint over the *page* background. Full-saturation
   Cloudflare orange (`#f6821f`) fails on white (~2.9:1); the mid-tone amber
   `#d97706` clears ~3:1 on both. Use a translucent fill (`...1a` alpha) so the
   container tints without hiding nodes.

## The palette

```
Subgraph (title + border, mid-tone, ~3:1 both modes):
  Cloudflare / edge   stroke+color #d97706   fill #f6821f1a
  Railway / core      stroke+color #8b5cf6   fill #7c3aed1a

Node fills (opaque) + text:
  Cloudflare compute  fill #f6821f  text #1a1a1a  stroke #9a3412   (Pages, Worker)
  Object store (R2)   fill #fbbf24  text #1a1a1a  stroke #92400e
  Railway compute     fill #7c3aed  text #ffffff  stroke #4c1d95   (API)
  Database            fill #6d28d9  text #ffffff  stroke #4c1d95
  Neutral / external  fill #475569  text #ffffff  stroke #94a3b8   (Browser)
```

Brand cue: orange family = Cloudflare, violet family = Railway, slate = neutral.
Keep that mapping consistent across every diagram in the repo.

## Always

- `<br/>` for line breaks in labels, never `\n` (GitHub renders literal `\n`).
- `&lt;env&gt;` for literal angle brackets in labels.
- Validate before committing: `mmdc -i diagram.mmd -o /tmp/x.svg` (a clean SVG
  means the syntax parses).

Reference implementation: the topology diagram in this repo's `README.md`.
