# Ideogram 4 speaks JSON, not prose

This is the part that is not in any model card, and it is the difference between Ideogram 4 being
disappointing and being the best open typography model you can run.

**Ideogram 4 was trained on structured JSON captions.** A plain text prompt is out of distribution.
You do not get an error. You get quietly worse images, weaker prompt adherence, and a sharp rise in
false positive refusals, where the model draws a flat grey "blocked" card for completely benign
subjects.

Send it JSON instead and the same idea renders better, follows layout instructions, and stops
refusing things it has no business refusing.

## The shape

Exactly these top level keys, in this order. Order matters, because it is part of what the model was
trained on.

```json
{
  "high_level_description": "One paragraph: subject, action, setting, mood, format.",
  "style_description": {
    "aesthetics": "the overall design/photographic register",
    "lighting": "direction, quality, what it does to the subject",
    "photo": "...photoreal look...",
    "medium": "what it physically is, print, photograph, lithograph",
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

Paste that into the `CLIPTextEncode` node. That is it. No special node, no preprocessor. The whole
JSON string is the prompt.

### `style_description`, pick a lane

It carries either `photo` or `art_style`. Never both, never neither. The key order differs between
the two paths:

| path | key order |
|---|---|
| photoreal | `aesthetics`, `lighting`, `photo`, `medium`, `color_palette` |
| illustration / design | `aesthetics`, `lighting`, `medium`, `art_style`, `color_palette` |

`color_palette` is optional, up to 16 uppercase `#RRGGBB` values. It is a strong steer. A tight five
colour palette is most of what makes a screen print look like a screen print.

### `bbox` is `[y_min, x_min, y_max, x_max]`

Note the ordering: y comes first. Integers 0 to 1000 regardless of your actual render resolution, so
the same caption works at any size or aspect ratio.

It is strong guidance rather than a hard constraint. The model will honour the broad placement and
proportion of a box, and will quietly adjust to make the composition work.

### `elements`

Key order within an element: `type`, `bbox`, `text`, `desc`, `color_palette`. Element level
`color_palette` is capped at 5.

There are two ways to specify rendered type, and both work. I have verified both produce correct
strings.

**A, a dedicated `text` element.** The literal string lives in its own `text` field, and `desc`
carries only the styling:

```json
{"type": "text", "bbox": [650, 90, 790, 910], "text": "DOLOMITI",
 "desc": "heavy condensed geometric sans serif, all capitals, tightly letterspaced, deep blue ink"}
```

**B, an `obj` element with the string quoted inside `desc`.** This is what ComfyUI's official
Ideogram 4 template does. Every element in its example caption is `type: "obj"`, with strings in
single quotes:

```json
{"type": "obj", "bbox": [128, 149, 354, 810],
 "desc": "Massive 3D puffy inflatable white typography spelling 'COMFY', stretching across the upper half"}
```

That template also repeats the rendered strings in `high_level_description`, which is worth copying,
because it reinforces what the model should be spelling.

Which to use: A gives you cleaner separation and one bbox per typographic level, which is what you
want for posters and covers with real hierarchy. B is what the official example demonstrates and
reads more naturally when the type is an object in the scene, like signage, packaging, or a torn
paper banner. Use A for layout driven design, B for type embedded in a photographed thing.

## Writing text elements well

This is where the model earns its reputation, so it is worth being deliberate.

**Put the string in `text` and the styling in `desc`.** Do not describe the string in `desc`, and do
not put styling hints inside `text`, because `text` is rendered literally.

**Describe the typeface by character, not by name.** "heavy condensed geometric sans serif, all
capitals, tightly letterspaced" works. Naming a specific font is unreliable, and naming a foundry's
font gets you an approximation at best.

**Give every text block its own element, one per typographic level.** A masthead, a headline, a deck
and a footer are four elements with four bboxes. That is how you get real hierarchy, because each
gets its own size, weight, colour and position.

Embedded newlines do work within one element. `"ROASTED IN\nSMALL BATCHES"` on the packaging example
renders correctly as two centred lines. Use them for a single block that happens to wrap. Just do
not use them to fake hierarchy, because everything in one element shares one styling.

**Say the colour and the alignment.** Unspecified type drifts toward centred black.

**Small print works.** Genuinely small, multi word lines render legibly. This is the single biggest
gap between Ideogram 4 and every other open model, so it is worth leaning on.

## Spatial relationships need to be concrete too

`bbox` places things. It does not describe how two elements relate, and this is where captions
quietly fail.

I asked for "a small orange silhouette of a front crawl swimmer, angled, overlapping the enclosed
counter of the letter O". Across two seeds the text rendered perfectly every time and the swimmer
came out as an orange bird beside the word. The model had no trouble with the typography. It had
trouble with "the enclosed counter of the letter O", which is a relationship rather than a thing.

Rewriting the same element as a self contained description, with the pose spelled out limb by limb,
its own bbox in clear space, and an explicit statement of what surrounds it, fixed it:

```
"a small solid orange silhouette of a single swimmer seen directly from above, face down, one arm
 reaching forward over the head and the other trailing back along the body, legs straight behind,
 surrounded by empty flat blue with no water and no other objects"
```

The general rule: describe each element as if it were alone in the frame, and let `bbox` do the
positioning. If you catch yourself writing "overlapping", "behind the", or "tucked into", that
phrase is doing work the model will not reliably do.

Know which failure you are looking at, too. A seed ladder rescues refusals and composition luck. It
will not rescue an underspecified `desc`, and you will just pay for the same mistake at a new seed.
If every seed fails the same way, stop laddering and rewrite the element.

## Concrete beats generic, and it doubles as a refusal fix

Filler prose is out of distribution. Phrases like "a polished professional setting" or "a high
quality modern scene" measurably degrade output and attract refusal cards, even for something as
innocent as a coffee mug.

Describe what is actually there. The prompt's real content is the only content.

The same rule fixes most false refusals. Sparse role based people, like "a retired couple", refuse
far more often than the same people described concretely, like "a man in his seventies with a white
beard and a navy fisherman's sweater". Vagueness reads as suspicious to the filter. Specificity does
not.

## When it refuses anyway

The refusal is trained into the model, so it draws a flat grey card as the image. Nothing in ComfyUI
errors out. Detect it cheaply by mean luminance and channel spread: a refusal card is flat and grey,
with low saturation and low variance, where a real flat colour design still has channel spread.

Recovery depends on how deep the refusal basin is:

- Refuses on some seeds: run a seed ladder. Most captions escape within a handful of seeds. This is
  exactly what the seedhunt workflow is for.
- Refuses on every seed: reword the caption. Make the vague parts concrete, which alone rescues most
  of them.

## A worked example

The `LONGEST` poster in `examples/`. One enormous word, three supporting lines at different sizes,
and a figure in clear space beside it:

```json
{
  "high_level_description": "A bold graphic poster where an enormous single word fills most of the frame, with a small swimmer beside it.",
  "style_description": {
    "aesthetics": "Swiss International Style poster design, ruthless grid, enormous type as the primary image, vast flat colour field",
    "lighting": "flat even studio light with no modelling, a single hard edged shadow beneath the swimmer",
    "medium": "offset lithograph on uncoated stock, faint paper tooth, one visible ink overprint",
    "art_style": "1960s Swiss graphic design, Helvetica-like grotesque, asymmetric grid, two colour print",
    "color_palette": ["#0F3B63", "#F4F1E8", "#E85D28"]
  },
  "compositional_deconstruction": {
    "background": "a single flat deep blue field filling the frame edge to edge, no gradient, no watermark",
    "elements": [
      {"type": "text", "bbox": [180, 40, 620, 960], "text": "LONGEST",
       "desc": "one enormous word in a tight grotesque sans, all capitals, cream ink, letterforms cropped hard by the left and right frame edges, filling the upper two thirds"},
      {"type": "obj", "bbox": [650, 620, 780, 940],
       "desc": "a single swimmer seen directly from above, printed as one completely flat solid orange shape with no shading, like a stencil or pictogram, fully clothed in a plain long sleeved swimsuit and a swim cap, face down with one arm reaching forward over the head and the other trailing back, legs straight behind, surrounded by empty flat blue"},
      {"type": "text", "bbox": [700, 40, 760, 560], "text": "THE LONGEST SWIM",
       "desc": "a medium all capitals grotesque line in orange, left aligned to the same margin as the hero word"},
      {"type": "text", "bbox": [790, 40, 840, 620], "text": "A FILM BY ANNIKA REUSS",
       "desc": "a small all capitals grotesque line in cream, left aligned, widely letterspaced"},
      {"type": "text", "bbox": [900, 40, 940, 480], "text": "IN CINEMAS FROM 14 MARCH",
       "desc": "very small all capitals grotesque text in cream, left aligned at the bottom margin"}
    ]
  }
}
```

Note what is doing the work: the art path with no `photo` key, a three colour palette that forces the
two ink look, one text element per typographic level, `desc` describing letterforms rather than
naming a font, and the swimmer described as a self contained shape in its own clear space.
