# Changelog

## v0.1.3-experimental — 2026-09-17

- Add backward-compatible auto/manual tile count and direction controls to the H3 sampler.
- Manual dropdown offers 2, 4, 6, 8 (default 2); auto retains the original 1–8 search.
- Keep English as the homepage and update the separate Chinese README, including VRAM estimation caveats.
- Show actual spatial plan, estimated memory and errors in a read-only node panel.
- Keep overlaps automatic and preserve the full-audio mask restriction.
- 47 CPU tests and frontend callback tests with a stub DOM passed; real model and live frontend verification pending.

## v0.1.2-experimental — 2026-09-16

- Allow high-resolution tiling for the validated audio-only mask case: video mask all 1, audio mask all 0.
- Automatically select spatial direction and tile count from available workspace; 1–8 tiles are possible.
- Forward the complete audio latent and H3 audio conditioning to every video tile.
- Keep rejecting partial/soft video masks, partial/soft audio masks, dynamic mask schedules, and ControlNet with the H3 tiling path.
- Add tile-stitching and masked-tiling regression tests; 34 CPU tests pass.
- Add separate English and Chinese documentation; English README is the repository homepage.
- Add a clickable GIF preview and publish the maintainer-provided `_00011-audio.mp4` as a public Release asset.
- Publish the additional maintainer-provided `Selflift Avatar demo.mp4` as a public demo Release asset.

## v0.1.1-experimental

- Connect H3 native audio conditioning and model-input audio injection instead of only restoring audio after prediction.
- Separate the high-resolution noisy resume state from the clean inpaint anchor.
- Preserve dual-stream masks, multi-channel masks, and constant image-sized SolidMask audio compatibility.
- 28 CPU tests passed; no systematic lip-sync benchmark.

## v0.1.0-experimental

- Independent Avatar H3, Image, and TST node registrations without overriding the original plugin.
- H3 dual-stream masks, multi-channel video masks, packed masks, and static audio constraints.
- H3 audio-scale correction and the masked pure pixel-anchor fix.
