# Ideogram 4 on Apple Silicon (ComfyUI / MPS)

Running [Ideogram 4](https://huggingface.co/Comfy-Org/Ideogram-4) — the 9.3B open-weights DiT —
**locally on a Mac**: two flat, readable, 100%-core-node workflows on Ideogram's official recipe,
every Apple-Silicon-specific requirement and measurement I hit getting there, and a proper writeup
of the JSON caption grammar the model actually wants.

Ideogram 4's standout capability among locally-runnable models is **typography**: exact strings,
several text blocks at once, small print that stays legible, and bbox-level control over placement.
Every example here is a straight render — no upscaler, no retouching, no compositing.

## Examples

<!-- EXAMPLES -->

## What this adds over the official template

**ComfyUI already ships an Ideogram 4 template** (Templates → Image → *Text to Image (Ideogram v4)*),
and you should know that before deciding you need this repo. It's the authority on the recipe, and
the recipe here is taken from it rather than invented.

What it doesn't give you:

- It's built as a **subgraph** wrapping math and JSON-extraction nodes to drive preset selection.
  Great to use, awkward to read or modify. The workflows here are **flat and explicit** — every node
  visible, values typed in directly, so you can see exactly what the recipe is.
- It says **nothing about Apple Silicon**. Which checkpoint works, what it costs per step, what it
  peaks at in memory, why one of the three quantisations is a non-starter — all of that is below,
  measured on a real machine.
- There's **no seed-scan variant**. Ideogram 4 on a Mac is minutes per image, so scanning cheaply
  before committing matters more here than on a 4090.

## The two workflows

| File | What it's for |
|---|---|
| `Ideogram4_Mac.json` | Single render, **V4_DEFAULT_20** at 1024×1024. |
| `Ideogram4_Mac_Seedhunt.json` | Seed scan, **V4_TURBO_12**. Seed set to `increment` — set a batch count in the queue and walk away. |

Both carry two in-graph panels — model downloads with direct links and install paths, and a
recipe/gotchas panel — so you don't need this README open while you work.

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

Ideogram's own docs call **Quality 48 the default**. On a Mac that's a long wait, so these
workflows ship Default 20 and Turbo 12 as the practical tiers — go to Quality 48 for a keeper.

**Presets appear to preserve composition at a fixed seed — which is what makes the seedhunt
workflow worth having.** Turbo and Default use different `mu` (0.5 vs 0.0), yet in the A/B I ran
(the Dolomiti poster, seed 42, 1088×1920) they produced the same layout — identical peaks, cable car
and sun — with Default 20 differing only in refinement: cleaner letterforms and paper grain. So the
intended loop is scan cheaply at Turbo 12, then re-render the seed you liked at Default 20 or
Quality 48.

**Caveat, stated plainly: that's one caption and one seed.** I haven't swept it, so treat it as a
strong working assumption rather than a guarantee — and eyeball your final against the scan rather
than assuming they match.

**The resolution must not change between those two steps.** Latent shape determines the noise
tensor, so the same seed at a different size is a different image. Both shipped workflows therefore
default to the same 1088×1920, and `Ideogram4Scheduler`'s width/height are kept in sync with the
latent in both.

**Two things worth understanding:**

**Ideogram 4 does CFG with two separate models.** Rather than running one UNet twice per step, it
ships a distinct unconditional model and `DualModelGuider` evaluates both. That's why you download
two 9.28 GB checkpoints and why both are resident while sampling.

**`Ideogram4Scheduler` takes width and height** — the sigma schedule is resolution-aware, and the
timestep shift is derived from the resolution plus the preset's `mu`/`std`. That's why there's no
`ModelSamplingAuraFlow` in the chain. **Its width/height must match your latent**, and both must be
multiples of 16 (the official template snaps with `max(((a + 15) // 16) * 16, 256)`). Change them
together or your schedule no longer matches what you're rendering.

## Requirements

- Apple Silicon Mac — see the memory numbers below
- **ComfyUI 0.26+** (needs the `ideogram4` CLIP type, `DualModelGuider` and `Ideogram4Scheduler`)
- **[ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8)** —
  ComfyUI Manager → search "AppleSilicon-FP8", or clone into `custom_nodes/` and
  `pip install -r requirements.txt`, then restart. Tested here on **torch 2.11**.

Models, all from [Comfy-Org/Ideogram-4](https://huggingface.co/Comfy-Org/Ideogram-4):

| File | Size | Goes in |
|---|---|---|
| `diffusion_models/ideogram4_fp8_scaled.safetensors` | 9.28 GB | `ComfyUI/models/diffusion_models/` |
| `diffusion_models/ideogram4_unconditional_fp8_scaled.safetensors` | 9.28 GB | `ComfyUI/models/diffusion_models/` |
| `text_encoders/qwen3vl_8b_fp8_scaled.safetensors` | 10.59 GB | `ComfyUI/models/text_encoders/` |
| `vae/flux2-vae.safetensors` | 0.34 GB | `ComfyUI/models/vae/` |

### Why fp8, and why that custom node

Comfy-Org ships Ideogram 4 in three formats, and **none of them is a dense bf16 checkpoint**:

| Checkpoint | Size | On Apple Silicon in ComfyUI |
|---|---|---|
| `ideogram4_fp8_scaled` | 9.28 GB | ✅ the practical route, via the AppleSilicon-FP8 node |
| `ideogram4_int8_convrot` | 9.58 GB | ⚠️ needs a recent ComfyUI; no optimised MPS path |
| `ideogram4_nvfp4_mixed` | 5.49 GB | ⚠️ NVFP4 arithmetic is Blackwell-class NVIDIA; only emulated here |

MPS could not touch `Float8_e4m3fn` at all until recently, which is why Ideogram 4 had no local Mac
story in ComfyUI. **[ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8)**
(by pawel-mazurkiewicz) installs MPS-scoped Python patches at ComfyUI startup, decoding contiguous
fp8 through a 256-entry device lookup table with guarded CPU fallbacks for layouts and conversions
MPS won't do. Its default path decodes fp8 to bf16 and then runs a bf16 matmul rather than doing
native fp8 arithmetic — so it's a **compatibility and memory win, not a speed win**, which is the
author's own framing. (There's also an experimental opt-in Metal fp8 kernel behind `ASFP8_FP8_EXT=1`.)

**Being precise about what is and isn't possible**, since it's easy to overclaim here:

- On the **int8** and **nvfp4** builds: recent ComfyUI can fall back to dequantise-then-matmul paths,
  so "cannot run" would be too strong. What's true is there's no optimised quantised GEMM for them
  on Metal, so fp8_scaled is the route that's actually worth using. I have not benchmarked the others.
- Ideogram 4 on Apple Silicon **outside ComfyUI** is a separate and real option:
  [MFLUX](https://github.com/filipstrand/mflux) runs it through MLX. This repo is specifically about
  the ComfyUI route.
- Community bf16 conversions of the weights exist on HuggingFace; the official releases have none.

## Speed and memory, measured

M-series Mac, 48 GB unified memory, ComfyUI launched `--lowvram --disable-smart-memory`, models on
an external SSD. These are the sampler's own reported `s/it`:

| Resolution | Megapixels | s/it |
|---|---|---|
| 720×1280 | 0.92 | 11.0 |
| 896×1600 | 1.43 | 15.9 |
| 1088×1920 | 2.09 | 23.3 |

**Cost scales linearly with pixel count** — roughly 11–12 s/it per megapixel. There's no sweet spot
to hunt for and no cliff to avoid; you're trading minutes for resolution. In practice:

| | 1024×1024 (~13 s/it) | 1088×1920 (~26 s/it) |
|---|---|---|
| Turbo 12 | ~2.6 min | ~5.2 min |
| Default 20 | ~4.3 min | ~8.7 min |
| Quality 48 | ~10.4 min | ~21 min |

Add a **one-time model load of roughly 1–2 minutes** (~19 GB of UNet). Paid once if ComfyUI stays
warm — and paid on *every* render under `--disable-smart-memory`, which dominates a seed hunt.

**Be honest with yourself about the loop.** This is a minutes-per-image model on a Mac. Scan
compositions with the seedhunt workflow, then spend Quality 48 only on a seed you already want.

**Memory — this is the tight part.** Across a 14-render batch at 1088×1920 with `--lowvram`, peak
device memory ranged **36.8 to 44.1 GB of 48 GB**, including a ~14 GB baseline from macOS and other
apps. At the top of that range there were under 4 GB free. It completed every time, but that is not
much margin, and it's the reason I'd treat ~2 MP as a sensible ceiling rather than a starting point.

Two practical consequences:

- **Close things before a long batch.** The baseline is yours to control and it's a third of the budget.
- **Don't raise the MPS watermark to "fix" an OOM.** Unified memory is shared with the OS; budgeting
  yourself past the cap trades a failed render for a hung machine. Lower the resolution instead.

I have not tested a 32 GB Mac and won't guess — the two UNets alone are 18.6 GB before activations.

## MPS gotchas

Apple-Silicon backend behaviour, not Ideogram bugs — most bite any large DiT in ComfyUI on a Mac.

1. **`batch_size` must stay 1.** The batch dimension *inside* one render is what breaks on MPS: set
   it to 4 and only one frame denoises, the rest come out static. Queuing many *separate* jobs is
   fine — that's exactly what the seedhunt workflow does with `increment`. Loop seeds, don't batch.
2. **Never set a LoRA to strength 0.0 — bypass the node instead.** A zeroed patch is not a no-op on
   MPS; it NaNs the model to pure black. Right-click → Bypass (Ctrl+B).
3. **First render is slow, then fine.** Cold-loading ~19 GB takes minutes, especially off an
   external drive. Keep ComfyUI warm across a seed hunt.
4. **Cheap failure detection.** Check mean luminance and channel spread with PIL's `ImageStat`:
   near-black means gotcha 2; flat grey with no channel spread means the model refused (below).

## Prompting: it wants JSON, not prose

The single biggest quality lever, written up in **[CAPTIONS.md](CAPTIONS.md)**.

Short version: Ideogram 4 was trained on structured JSON captions. A plain-text prompt is
out-of-distribution — quietly worse images, weaker adherence, and more false-positive refusals, with
no error to tell you. Paste a JSON caption into `CLIPTextEncode`; the whole JSON string is the
prompt, no special node needed.

Refusals are trained into the model — it renders a flat grey card rather than failing. Refuses on
*some* seeds → a seed ladder rescues it (that's the seedhunt workflow). Refuses on *every* seed →
reword it; replacing vague filler with concrete description fixes most.

## Credits

Model weights by [Ideogram](https://ideogram.ai), packaged by
[Comfy-Org](https://huggingface.co/Comfy-Org/Ideogram-4), recipe from ComfyUI's official template.
The fp8-on-MPS work that makes this possible is
[pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8](https://github.com/pawel-mazurkiewicz/ComfyUI-AppleSilicon-FP8).
