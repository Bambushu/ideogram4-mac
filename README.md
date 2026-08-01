# Ideogram 4 on Apple Silicon (ComfyUI / MPS)

Running [Ideogram 4](https://huggingface.co/Comfy-Org/Ideogram-4), the 9.3B open weights DiT,
locally on a Mac. Two flat readable workflows on Ideogram's official recipe, one of them pure core
nodes and one that lets you draw the layout, every Apple Silicon requirement and measurement I hit
getting there, and a proper writeup of the JSON caption grammar the model actually wants.

Ideogram 4's standout capability among locally runnable models is typography: exact strings, several
text blocks at once, small print that stays legible, and bbox level control over placement. Every
example here is a straight render. No upscaler, no retouching, no compositing.

## Examples

| | |
|---|---|
| ![magazine_cover](examples/magazine_cover.png) | ![travel_poster](examples/travel_poster.png) |
| ![neon_signage](examples/neon_signage.png) | ![packaging](examples/packaging.png) |
| ![chalkboard_menu](examples/chalkboard_menu.png) | ![swiss_type](examples/swiss_type.png) |

## What this adds over the official template

ComfyUI already ships an Ideogram 4 template (Templates, Image, *Text to Image (Ideogram v4)*), and
you should know that before deciding you need this repo. It is the authority on the recipe, and the
recipe here is taken from it rather than invented.

What it does not give you:

- It is built as a subgraph wrapping math and JSON extraction nodes to drive preset selection. Great
  to use, awkward to read or modify. The workflows here are flat and explicit, with every node
  visible and values typed in directly, so you can see exactly what the recipe is.
- It says nothing about Apple Silicon. Which checkpoint works, what it costs per step, what it peaks
  at in memory, why one of the three quantisations is a non starter: all of that is below, measured
  on a real machine.
- There is no visual way to place your layout. Ideogram 4's whole strength is putting exact text in
  exact places, and the official template still has you typing bbox integers by hand.

## The workflows

| File | What it is for | Needs |
|---|---|---|
| `Ideogram4_Mac.json` | Single render, **V4_DEFAULT_20** at 1088x1920, about 8 minutes. | core nodes only |
| `Ideogram4_Mac_PromptBuilder.json` | Same recipe, but you **draw the layout** instead of hand writing bboxes. | + [KJNodes](https://github.com/kijai/ComfyUI-KJNodes) |

Both ship with the caption that produced the Dolomiti poster above, so queueing any of them
unmodified reproduces a known good result before you start changing things. Drop the resolution to
1024x1024 if you want roughly half the render time.

Each carries two in graph panels, one with model downloads and install paths and one with the
recipe and the Mac gotchas, so you do not need this README open while you work.

### Which one to reach for

**Start with the prompt builder if you are doing layout work**, which is most of what this model is
good for. It uses `Ideogram4PromptBuilderKJ` from KJNodes: you drag regions on a canvas, mark each
one as `obj` or `text`, and it assembles the JSON caption for you. That removes the single most
error prone part of the format, since it owns the `[ymin, xmin, ymax, xmax]` 0-1000 convention that
is very easy to get backwards by hand. It can also use your last render as the canvas background, so
you place the next layout on top of what you actually got.

It also wires its `width` and `height` outputs into both `Ideogram4Scheduler` and the latent, so
those two cannot drift apart. That is a real gotcha in the core-nodes-only graph, where you have to
remember to change both.

The cost is one custom node. If you want a graph that runs on a stock ComfyUI with nothing extra
installed, `Ideogram4_Mac.json` is that graph and always will be.

**A note on seed hunting, since it is the reflex from other models.** With most models you roll
seeds to find a composition. Ideogram 4 inverts that: composition is declared in the caption's
bboxes, so rolling seeds to discover a layout is fighting the model instead of using it. Change the
caption, not the seed. Where a seed ladder does still earn its place is escaping refusals, and for
that you just set the seed widget to `increment` and queue a few.

## The recipe (from ComfyUI's official template)

| Setting | Value |
|---|---|
| UNet (conditional) | `ideogram4_fp8_scaled.safetensors`, dtype `default` |
| UNet (unconditional) | `ideogram4_unconditional_fp8_scaled.safetensors`, dtype `default` |
| Text encoder | `qwen3vl_8b_fp8_scaled.safetensors`, **type = `ideogram4`** |
| VAE | `flux2-vae.safetensors` |
| Sigmas | **`Ideogram4Scheduler`** (steps, width, height, mu, std) |
| Guidance | `DualModelGuider` cfg **7.0** |
| Guidance tail | `CFGOverride` cfg **3.0**, start **0.7**, end **1.0** |
| Sampler | **euler** |
| Negative | `ConditioningZeroOut` of the positive |

Official presets, embedded in the template:

| Preset | Steps | mu | std |
|---|---|---|---|
| `V4_QUALITY_48` | 48 | 0.0 | 1.50 |
| `V4_DEFAULT_20` | 20 | 0.0 | 1.75 |
| `V4_TURBO_12` | 12 | 0.5 | 1.75 |

Ideogram's own docs call Quality 48 the default. On a Mac that is a long wait, so these workflows
ship Default 20 and Turbo 12 as the practical tiers. Go to Quality 48 for a keeper.

**Presets seem to preserve composition at a fixed seed.** Turbo and Default use different `mu` (0.5 against 0.0), yet in the A/B I ran (the
Dolomiti poster, seed 42, 1088x1920) they produced the same layout, with Default 20 differing only
in refinement: cleaner letterforms and paper grain. So the intended loop is to scan cheaply at
Turbo 12, then re render the seed you liked at Default 20 or Quality 48.

There is a real limit to that, though. Rendering the magazine cover above at both Default 20 and
Quality 48 on the same seed shows the layout holding exactly, with masthead, headline, deck, cover
line and footer all identical, while the subject's fine pose shifts slightly between them. Those two
presets share `mu` and differ in `std`, so preset changes are not free. Treat composition as stable
enough to pick a seed by, but expect fine detail to move, and eyeball the final rather than
assuming.

**The resolution must not change between those two steps.** Latent shape determines the noise
tensor, so the same seed at a different size is a different image. Both shipped workflows therefore
default to the same 1088x1920, and `Ideogram4Scheduler`'s width and height are kept in sync with the
latent in both.

Two more things worth understanding.

**Ideogram 4 does CFG with two separate models.** Rather than running one UNet twice per step, it
ships a distinct unconditional model and `DualModelGuider` evaluates both. That is why you download
two 9.28 GB checkpoints and why both are resident while sampling.

**`Ideogram4Scheduler` takes width and height.** The sigma schedule is resolution aware, and the
timestep shift is derived from the resolution plus the preset's `mu` and `std`. That is why there is
no `ModelSamplingAuraFlow` in the chain. Its width and height must match your latent, and both must
be multiples of 16 (the official template snaps them with `max(((a + 15) // 16) * 16, 256)`). Change
them together or your schedule no longer matches what you are rendering.

## Requirements

- Apple Silicon Mac. See the memory numbers below.
- **ComfyUI 0.26+**, which is where the `ideogram4` CLIP type, `DualModelGuider` and
  `Ideogram4Scheduler` live.
- **[ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8)**.
  ComfyUI Manager, search "AppleSilicon-FP8", or clone into `custom_nodes/` and
  `pip install -r requirements.txt`, then restart. Tested here on torch 2.11.

Models, all from [Comfy-Org/Ideogram-4](https://huggingface.co/Comfy-Org/Ideogram-4):

| File | Size | Goes in |
|---|---|---|
| `diffusion_models/ideogram4_fp8_scaled.safetensors` | 9.28 GB | `ComfyUI/models/diffusion_models/` |
| `diffusion_models/ideogram4_unconditional_fp8_scaled.safetensors` | 9.28 GB | `ComfyUI/models/diffusion_models/` |
| `text_encoders/qwen3vl_8b_fp8_scaled.safetensors` | 10.59 GB | `ComfyUI/models/text_encoders/` |
| `vae/flux2-vae.safetensors` | 0.34 GB | `ComfyUI/models/vae/` |

### Why fp8, and why that custom node

Comfy-Org ships Ideogram 4 in three formats, and none of them is a dense bf16 checkpoint:

| Checkpoint | Size | On Apple Silicon in ComfyUI |
|---|---|---|
| `ideogram4_fp8_scaled` | 9.28 GB | the practical route, via the AppleSilicon-FP8 node |
| `ideogram4_int8_convrot` | 9.58 GB | needs a recent ComfyUI, and has no optimised MPS path |
| `ideogram4_nvfp4_mixed` | 5.49 GB | NVFP4 arithmetic is Blackwell class NVIDIA, so only emulated here |

MPS could not touch `Float8_e4m3fn` at all until recently, which is why Ideogram 4 had no local Mac
story in ComfyUI.
**[ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8)** by
pawel-mazurkiewicz installs MPS scoped Python patches at ComfyUI startup, decoding contiguous fp8
through a 256 entry device lookup table with guarded CPU fallbacks for layouts and conversions MPS
will not do. Its default path decodes fp8 to bf16 and then runs a bf16 matmul rather than doing
native fp8 arithmetic, so it is a compatibility and memory win rather than a speed win, which is the
author's own framing. There is also an experimental opt in Metal fp8 kernel behind `ASFP8_FP8_EXT=1`.

Being precise about what is and is not possible, since it is easy to overclaim here:

- On the int8 and nvfp4 builds, recent ComfyUI can fall back to dequantise then matmul paths, so
  saying they cannot run would be too strong. What is true is that there is no optimised quantised
  GEMM for them on Metal, so fp8_scaled is the route actually worth using. I have not benchmarked
  the others.
- Ideogram 4 on Apple Silicon outside ComfyUI is a separate and real option.
  [MFLUX](https://github.com/filipstrand/mflux) runs it through MLX. This repo is specifically about
  the ComfyUI route.
- Community bf16 conversions of the weights exist on HuggingFace. The official releases have none.

## Speed and memory, measured

M series Mac, 48 GB unified memory, ComfyUI launched `--lowvram --disable-smart-memory`, models on
an external SSD. These are the sampler's own reported `s/it`:

| Resolution | Megapixels | s/it |
|---|---|---|
| 720x1280 | 0.92 | 11.0 |
| 896x1600 | 1.43 | 15.9 |
| 1088x1920 | 2.09 | 23.3 |

Cost scales linearly with pixel count, roughly 11 to 12 s/it per megapixel. There is no sweet spot
to hunt for and no cliff to avoid. You are simply trading minutes for resolution. Pick your
resolution from the time you are willing to spend.

| | 1024x1024 (about 13 s/it) | 1088x1920 (about 23 s/it) |
|---|---|---|
| Turbo 12 | about 2.6 min | about 5 min |
| Default 20 | about 4.3 min | about 8 min |
| Quality 48 | about 10 min | about 19 min |

Add a one time model load of roughly 1 to 2 minutes for about 19 GB of UNet. It is paid once if you
leave ComfyUI warm, and paid on every render under `--disable-smart-memory`, which dominates a seed
hunt.

Be honest with yourself about the loop. This is a minutes per image model on a Mac. Scan
compositions at Turbo 12, then spend Quality 48 only on a seed you already want.

**Memory is the tight part.** Across a 14 render batch at 1088x1920 with `--lowvram`, peak device
memory ranged from 36.8 to 44.4 GB out of 48, including a baseline of roughly 14 GB from macOS and
other apps. At the top of that range there was under 4 GB free. It completed every time, but that is
not much margin, and it is the reason I would treat 2 MP as a sensible ceiling rather than a
starting point.

Two practical consequences:

- Close things before a long batch. The baseline is yours to control and it is a third of the budget.
- Do not raise the MPS watermark to fix an OOM. Unified memory is shared with the OS, so budgeting
  yourself past the cap trades a failed render for a hung machine. Lower the resolution instead.

I have not tested a 32 GB Mac and will not guess. The two UNets alone are 18.6 GB before activations.

## MPS gotchas

These are Apple Silicon backend behaviours rather than Ideogram bugs, and most will bite any large
DiT in ComfyUI on a Mac.

1. **`batch_size` must stay 1.** The batch dimension inside one render is what breaks on MPS. Set it
   to 4 and only one frame denoises while the rest come out static. Queuing many separate jobs is
   fine: set the seed widget to `increment` and raise the queue batch count. Loop seeds, do not
   batch.
2. **Never set a LoRA to strength 0.0. Bypass the node instead.** A zeroed patch is not a no op on
   MPS. It NaNs the model to pure black. Right click, Bypass, or Ctrl+B.
3. **First render is slow, then fine.** Cold loading about 19 GB takes minutes, especially off an
   external drive. Keep ComfyUI warm across a seed hunt.
4. **Cheap failure detection.** Check mean luminance and channel spread with PIL's `ImageStat`.
   Near black means gotcha 2. Flat grey with no channel spread means the model refused, which is
   covered below.

## Prompting: it wants JSON, not prose

This is the single biggest quality lever, and it is written up in **[CAPTIONS.md](CAPTIONS.md)**.

Short version: Ideogram 4 was trained on structured JSON captions. A plain text prompt is out of
distribution, which gets you quietly worse images, weaker adherence, and more false positive
refusals, with no error to tell you. Paste a JSON caption into `CLIPTextEncode` instead. The whole
JSON string is the prompt, and no special node is needed.

Refusals are trained into the model, so it renders a flat grey card rather than failing. If a
caption refuses on some seeds, a seed ladder rescues it: set the seed widget to `increment` and
queue a few. If it refuses on every seed, reword it, because replacing vague filler with concrete
description fixes most cases.

## Credits

Model weights by [Ideogram](https://ideogram.ai), packaged by
[Comfy-Org](https://huggingface.co/Comfy-Org/Ideogram-4), recipe from ComfyUI's official template.
The fp8 on MPS work that makes this possible is
[pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8).
