# selflift-Avatar

**Current public development version: `v0.1.2-experimental`** · [中文说明](README_CN.md) · [Changelog](CHANGELOG.md) · [v0.1.2 release](https://github.com/slmonker/selflift-Avatar/releases/tag/v0.1.2-experimental)

An independent experimental SelfLift branch for ComfyUI / MiniMax H3. It focuses on H3 audio-video latent masks, preserving source audio during sampling, and a narrowly supported high-resolution tiling path. It registers separate node IDs, so it can coexist with the original SelfLift plugin.

> **Experimental, unofficial project.** The v0.1.2 code has passed 34 CPU tests and has been exercised in the maintainer's H3 workflow. This is not a systematic benchmark of lip-sync accuracy, speed, or every model/plugin combination. Results can vary by character, audio, prompt, sampler, latent upscaler, and seed.

## What changed in v0.1.2

- Allows high-resolution tiling when the video is fully generated (`video mask = 1` everywhere) and the source audio is fully preserved (`audio mask = 0` everywhere).
- Automatically chooses the spatial tile count and direction from available workspace; the current planner can select 1–8 tiles.
- Every spatial tile receives the complete audio latent and complete H3 audio conditioning. Audio is not split by the visual tile boundary.
- Keeps rejecting partial video preservation, partial/soft audio masks, and other mask combinations that have not been validated with tiling.
- Adds tests for tile stitching, full-audio preservation, H3 audio mask forwarding, invalid mask combinations, and the no-mask path.

## Demo

**H3 audio-driven demo:** [Download / watch `_00011-audio.mp4`](https://github.com/slmonker/selflift-Avatar/releases/download/v0.1.2-experimental/selfliftAvatar_00011-audio.mp4)

This demo was provided by the maintainer and is published as a Release asset. The repository does not include model weights. The video is an example of one workflow, not a universal lip-sync benchmark.

## Installation

You need a working MiniMax H3 ComfyUI setup and the models, VAE, LoRA, and optional latent upscaler required by your workflow. This project adds no dependencies and does not include model weights.

### Git installation

```bash
git clone https://github.com/slmonker/selflift-Avatar.git ComfyUI/custom_nodes/selflift-Avatar
```

To update an existing Git installation:

```bash
cd ComfyUI/custom_nodes/selflift-Avatar
git pull --ff-only
```

Restart the ComfyUI backend after updating. A browser refresh alone does not reload Python nodes.

### ZIP installation

Download the latest ZIP from the [Releases page](https://github.com/slmonker/selflift-Avatar/releases), back up the old `selflift-Avatar` folder outside `custom_nodes`, replace it with the new folder, and restart ComfyUI. Do not keep two copies of this plugin in `custom_nodes`.

The original `comfyui-SelfLift` folder can remain installed; this project uses different node IDs.

## Nodes

| Node ID | Display name |
| --- | --- |
| `SelfLiftAvatarH3Sampler` | selflift-Avatar Sampler (MiniMax H3) |
| `SelfLiftAvatarImageSampler` | selflift-Avatar Sampler (Image) |
| `SelfLiftAvatarH3TST` | selflift-Avatar H3 TST |

The sampler uses the standard `sampler` and `sigmas` inputs. Use the standard Euler sampler with `s_churn=0`.

## Audio-driven H3 workflow

A typical source-audio path is:

```text
source audio → trim to target clip → H3 audio VAE encode
                                             ↓
SolidMask(value=0) → SetLatentNoiseMask
                                             ↓
H3 video latent ───────────────→ concatenate AV latent
                                             ↓
                         selflift-Avatar H3 sampler
```

- Audio mask `0` preserves the source audio; audio mask `1` allows audio generation. Values between 0 and 1 are soft constraints.
- The encoded audio, video frame range, and final output FPS must refer to the same clip.
- If the final video node uses the original audio directly, hearing the correct soundtrack does not prove that H3 used it to drive the mouth during generation. Evaluate the generated frames.
- The sampler is not a standalone post-production lip-sync tool.

## High-resolution tiling

Set `highres_tiling=true` to enable the automatic high-resolution spatial tiling planner. The planner chooses the direction and number of tiles based on available workspace; it may choose one tile, which means no actual split.

### Supported masked tiling mode

The currently validated masked mode is deliberately narrow:

```text
video mask: all 1 → generate the complete video
 audio mask: all 0 → preserve the complete source audio
```

Every video tile receives the complete audio latent and complete H3 audio conditioning. The tiles are stitched in the video spatial dimension; audio is not spatially tiled.

### Still rejected with tiling

- Any video mask containing 0 or soft values.
- Partial or soft audio masks.
- Dynamic `denoise_mask_function` schedules.
- ControlNet with the H3 tiling path.

Disable `highres_tiling` for those workflows. Removing the validation would not make those combinations correct because their masks would also need to be cropped, transformed, and aligned with each tile.

Look for these console messages to confirm the planner:

```text
[selflift-Avatar plan] automatic high-resolution tiling enabled
[SelfLift tiling plan] axis=W tiles=4
```

`axis=H` or `axis=W` is the selected spatial direction. `tiles=N` is the automatically selected tile count.

## Mask support without tiling

- Regular image/video tensors: BHW, BCHW, and BCTHW; one channel or the actual latent channel count.
- H3 `NestedTensor(video_mask, audio_mask)` with separate video/audio handling.
- Packed masks whose flattened size matches the AV latent streams.
- Audio masks in T, BT, BST, and BCST layouts; dimensions of 1 broadcast.
- Spatial video masks are resized to the latent grid. Audio time is not guessed or automatically resampled.
- A constant image-sized `SolidMask` attached to audio, such as `[1,1,928,1664]`, is recognized as a constant audio constraint and broadcast to the audio latent.

Mask semantics: **0 = preserve, 1 = generate, 0–1 = soft constraint**. A meaningful preserve result requires an initialized latent containing the content to preserve.

## Native H3 audio conditioning

The sampler passes the packed mask into ComfyUI's native H3 conditioning path before sampling. It uses H3's `audio_denoise_mask` and input-side `scale_latent_inpaint` behavior instead of only restoring audio after the model prediction. The high-resolution resume state and the clean audio anchor are kept separate so residual sampling noise is not treated as source audio.

This fixes mask/conditioning transport. It does not guarantee perfect lip sync. The external H3 latent upscaler has temporal convolutions and does not directly receive the target audio; the low-resolution prefix and the number of high-resolution correction steps can also affect mouth detail.

For a controlled comparison, keep the same model, audio clip, prompt, frame count, FPS, and seed. Compare the native baseline, the current SelfLift route, different `transition_step` values, and the `upscaler_model=none` pixel-anchor route one variable at a time.

## Testing

Run with the ComfyUI Python environment:

```bash
python tests/test_avatar.py
```

The current suite contains 34 CPU tests covering AV packing, H3 audio scaling, full/partial masks, constant audio SolidMask conversion, native audio conditioning, two-stage sampling, automatic tile planning, tile stitching, complete-audio forwarding to every tile, invalid masked-tiling combinations, and the unmasked tiling path. The tests do not load a large H3 checkpoint or measure final video lip-sync quality.

## Source and licensing

This is based on a local snapshot of [facok/comfyui-SelfLift](https://github.com/facok/comfyui-SelfLift). The upstream documentation is preserved under [`docs/upstream/`](docs/upstream/), and source hashes are recorded in [`PROVENANCE.json`](PROVENANCE.json).

The upstream repository did not declare a license at the time this project was prepared. This repository does not add a license on behalf of the upstream author and does not imply permission to redistribute upstream code or model weights beyond the applicable rights. Research papers, third-party code, and model checkpoints remain the responsibility of their respective authors.

## Rollback

[v0.1.1-experimental](https://github.com/slmonker/selflift-Avatar/releases/tag/v0.1.1-experimental) remains available for comparison. You can also reconnect the original SelfLift node in the workflow. To uninstall, stop ComfyUI and move this folder out of `custom_nodes`.
