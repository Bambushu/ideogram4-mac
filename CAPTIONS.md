# Ideogram 4 speaks JSON, not prose

This is the part that isn't in any model card, and it's the difference between Ideogram 4 being
disappointing and being the best open typography model you can run.

**Ideogram 4 was trained on structured JSON captions.** A plain-text prompt is out-of-distribution.
You don't get an error — you get quietly worse images, weaker prompt adherence, and a sharp rise in
false-positive refusals (the model draws a flat gray "blocked" card for completely benign subjects).

Send it JSON instead and the same idea renders better, follows layout instructions, and stops
refusing things it has no business refusing.

## The shape

Exactly these top-level keys, **in this order**. Order matters — it's part of what the model was
trained on.

```json
{
  "high_level_description": "One paragraph: subject, action, setting, mood, format.",
  "style_description": {
    "aesthetics": "the overall design/photographic register",
    "lighting": "direction, quality, what it does to the subject",
    "photo": "...photoreal look...",
    "medium": "what it physically is — print, photograph, lithograph",
    "color_palette": ["#F2E8D9", "#2E2A26"]
  },
  "compositional_deconstruction": {
    "background": "What fills the frame behind everything. No watermark, no caption text.",
    "elements": [
      {"type": "obj",  "bbox": [200, 100, 880, 900], "desc": "..."},
      {"type": "text", "bbox": [80, 100, 200, 900], "text": "EXACT STRING",
       "desc": "font, weight, colour, placement"}
    ]
  }
}
```

Paste that into the `CLIPTextEncode` node. That's it — no special node, no preprocessor. The whole
JSON string *is* the prompt.

### `style_description` — pick a lane

It carries either `photo` **or** `art_style`. Never both, never neither. The key order differs
between the two paths:

| path | key order |
|---|---|
| photoreal | `aesthetics`, `lighting`, `photo`, `medium`, `color_palette` |
| illustration / design | `aesthetics`, `lighting`, `medium`, `art_style`, `color_palette` |

`color_palette` is optional: up to 16 uppercase `#RRGGBB` values. It's a strong steer — a tight
5-colour palette is most of what makes a screen-print look like a screen-print.

### `bbox` — `[y_min, x_min, y_max, x_max]`

Note the ordering: **y first**. Integers `0–1000` regardless of your actual render resolution, so
the same caption works at any size or aspect ratio.

It's strong guidance, not a hard constraint. The model will honour the broad placement and
proportion of a box, and will quietly adjust to make the composition work.

### `elements`

Key order within an element: `type`, `bbox`, `text`, `desc`, `color_palette`.
Element-level `color_palette` is capped at 5.

There are **two ways to specify rendered type**, and both work. I've verified both produce correct
strings:

**A — a dedicated `text` element.** The literal string lives in its own `text` field, `desc`
carries only the styling:

```json
{"type": "text", "bbox": [650, 90, 790, 910], "text": "DOLOMITI",
 "desc": "heavy condensed geometric sans serif, all capitals, tightly letterspaced, deep blue ink"}
```

**B — an `obj` element with the string quoted inside `desc`.** This is what ComfyUI's official
Ideogram 4 template does — every element in its example caption is `type: "obj"`, with strings in
single quotes:

```json
{"type": "obj", "bbox": [128, 149, 354, 810],
 "desc": "Massive 3D puffy inflatable white typography spelling 'COMFY', stretching across the upper half"}
```

That template also **repeats the rendered strings in `high_level_description`**, which is worth
copying — it reinforces what the model should be spelling.

**Which to use.** A gives you cleaner separation and one bbox per typographic level, which is what
you want for posters and covers with real hierarchy. B is what the official example demonstrates and
reads more naturally when the type *is* an object in the scene — signage, packaging, a torn paper
banner. Use A for layout-driven design, B for type embedded in a photographed thing.

## Writing text elements well

This is where the model earns its reputation, so it's worth being deliberate.

**Put the string in `text` and the styling in `desc`.** Don't describe the string in `desc` and
don't put styling hints inside `text` — `text` is rendered literally.

**Describe the typeface by character, not by name.** "heavy condensed geometric sans serif, all
capitals, tightly letterspaced" works. Naming a specific font is unreliable and naming a foundry's
font gets you an approximation at best.

**Give every text block its own element.** A masthead, a headline, a deck and a footer are four
elements with four bboxes, not one block of text with line breaks. Separate elements are how you
get real typographic hierarchy, and they're far more reliable than embedded newlines.

**Say the colour and the alignment.** Unspecified type drifts toward centred black.

**Small print works** — genuinely small, multi-word lines render legibly. This is the single
biggest gap between Ideogram 4 and every other open model, so it's worth leaning on.

## Concrete beats generic — this is also a refusal fix

Filler prose is out-of-distribution. Phrases like "a polished professional setting" or "a
high-quality modern scene" measurably degrade output *and* attract refusal cards, even for
something as innocent as a coffee mug.

Describe what is actually there. The prompt's real content is the only content.

The same rule fixes most false refusals: sparse role-based people ("a retired couple") refuse far
more often than the same people described concretely ("a man in his seventies with a white beard
and a navy fisherman's sweater"). Vagueness reads as suspicious to the filter; specificity doesn't.

## When it refuses anyway

The refusal is trained into the model — **it draws a flat gray card as the image**. Nothing in
ComfyUI errors out. Detect it cheaply by mean luminance and channel spread: a refusal card is flat
and grey (low saturation, low variance), where a real flat-colour design still has channel spread.

Recovery depends on how deep the refusal basin is:

- **Refuses on *some* seeds** → run a seed ladder. Most captions escape within a handful of seeds.
  This is exactly what the seedhunt workflow is for.
- **Refuses on *every* seed** → reword the caption. Make the vague parts concrete; that alone
  rescues most of them.

## A worked example

The `LONGEST` poster in `examples/` — one enormous word, three supporting lines at different sizes,
and an object overlapping a letter:

```json
{
  "high_level_description": "A bold graphic poster where an enormous single word fills most of the frame, with a small swimmer silhouette cutting across the counter of one letter.",
  "style_description": {
    "aesthetics": "Swiss International Style poster design, ruthless grid, enormous type as the primary image, vast flat colour field",
    "lighting": "flat even studio light with no modelling, a single hard-edged shadow beneath the swimmer",
    "medium": "offset lithograph on uncoated stock, faint paper tooth, one visible ink overprint",
    "art_style": "1960s Swiss graphic design, Helvetica-like grotesque, asymmetric grid, two-colour print",
    "color_palette": ["#0F3B63", "#F4F1E8", "#E85D28"]
  },
  "compositional_deconstruction": {
    "background": "a single flat deep blue field filling the frame edge to edge, no gradient, no watermark",
    "elements": [
      {"type": "text", "bbox": [180, 40, 620, 960], "text": "LONGEST",
       "desc": "one enormous word in a tight grotesque sans, all capitals, cream ink, letterforms cropped hard by the left and right frame edges, filling the upper two thirds"},
      {"type": "obj", "bbox": [330, 380, 470, 640],
       "desc": "a small orange silhouette of a front-crawl swimmer mid-stroke, angled, overlapping the enclosed counter of the letter O"},
      {"type": "text", "bbox": [700, 40, 760, 560], "text": "THE LONGEST SWIM",
       "desc": "a medium all-capitals grotesque line in orange, left aligned to the same margin as the hero word"},
      {"type": "text", "bbox": [790, 40, 840, 620], "text": "A FILM BY ANNIKA REUSS",
       "desc": "a small all-capitals grotesque line in cream, left aligned, widely letterspaced"},
      {"type": "text", "bbox": [900, 40, 940, 480], "text": "IN CINEMAS FROM 14 MARCH",
       "desc": "very small all-capitals grotesque text in cream, left aligned at the bottom margin"}
    ]
  }
}
```

Note what's doing the work: the art path (no `photo` key), a three-colour palette that forces the
two-ink look, one text element per typographic level, and `desc` describing letterforms rather than
naming a font.
