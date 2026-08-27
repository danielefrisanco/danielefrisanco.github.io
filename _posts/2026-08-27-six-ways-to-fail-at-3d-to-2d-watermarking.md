---
layout: post
title: "Six Ways to Fail at 3D-to-2D Watermarking"
date: 2026-08-27 09:00:00 +0200
description: "Feasibility research done before a hackathon: what breaks when you try to recover a watermark from a screenshot of a rendered 3D model, and why the failed experiments turned out to be the thing worth having."
tags: [Watermarking, 3D, Provenance, Cultural Heritage, Research]
---

Feasibility research done before a hackathon: what breaks when you try to recover a
watermark from a screenshot of a rendered 3D model, and why the failed experiments
turned out to be the thing worth having.

## The problem

I had an idea for Hackit!4EU — in the context of [Twin It! 3D](https://www.europeana.eu/en/galleries/15694-twin-it-a-pan-european-collection-of-heritage-3-d-models), Europeana's pan-European collection of heritage 3D models — a provenance platform for cultural heritage assets. Before committing to it on the day, I wanted to know whether the core mechanism was actually possible. So I spent the run-up building the thing to find out.

The use case is concrete. A museum digitises an artifact into a GLB and distributes it for research or licensing. A game studio takes that model and ships it without attribution. The museum wants to prove the model came from them, and their only available tool is the one any player has: take a screenshot.

That constraint is the whole problem. The museum cannot access the studio's files. They have pixels from one rendered frame. Provenance has to be recoverable from that alone.

This turns out to be much harder than it sounds, because rendering is a destructive, lossy, one-way transform. Consider what a GPU actually does to your watermark between the file and the screenshot:

- **Geometry is discarded.** Only the visible surface from one viewpoint survives. Back faces, occluded interiors, and anything outside the frustum are gone.
- **UV mapping warps texture space.** Your carefully placed 2D texture signal gets projected through perspective onto an arbitrary surface, then resampled per-fragment.
- **Lighting overwrites pixel values.** Diffuse and specular terms, tone mapping, and gamma correction all rewrite the exact RGB values a texture watermark depends on.
- **Mipmapping destroys high frequencies.** The GPU picks a downsampled texture level per fragment based on screen-space derivatives. Fine-detail signal is averaged away before it ever reaches a pixel.
- **Depth is flattened.** A 3D signal becomes a 2D projection with no inverse.

A watermark has to survive all of that simultaneously, without knowing the viewpoint, the lighting rig, or the renderer.

![Pipeline diagram: a GLB file's geometry, UV layout, textures and normal maps pass through a GPU render that discards hidden faces, warps UVs to screen space, rewrites RGB with lighting and averages away detail with mipmaps, leaving a screenshot of flat pixels with no depth and no UV. The watermark must be recoverable from that screenshot with no access to the file.](/assets/images/posts/render-destroys-signal.png)

## The six approaches, and how each one died

I implemented and tested six methods end-to-end against real renders. Here's the scorecard up front:

| # | Approach | Signal location | File | Screenshot | What killed it |
|---|----------|-----------------|:----:|:----------:|----------------|
| 1 | Neural watermark in base color texture | Texture pixels | &#10003; | &#10007; | 29 bit errors in a 61-bit payload (error correction fixes 5). Decoder never trained on rendered input |
| 2 | Same, replicated across every texture | Texture pixels | &#10003; | &#10007; | Same domain mismatch, more compute |
| 3 | FM concentric rings in normal map | Surface lighting response | 0 err | &#10007; | Gradient variance uniform (~40 std at all radii) — needs directional light the deployer doesn't control |
| 4 | Notches cut into mesh silhouette | Object contour | — | &#10007; | Ornate heritage contours indistinguishable from notches |
| 5 | High-contrast QR stamped on texture | Visible texture pattern | — | &#10007; | UV atlases fragment the code across patches; finders never intact |
| 6 | Fixed-viewpoint ("canonical view") watermark | 2D render, baked back to texture | &#10007; | &#10007; | 2D leg works; bake-back attenuates residual 2.549&rarr;0.382, below 8-bit floor, pastes screen onto UV space |
| — | Deep 3D-to-2D (Yoo et al., CVPR 2022) | Learned end-to-end | &#10003; | &#10003; | Nothing — needs GPU training on thousands of meshes; no weights released |

The four hiding places are easier to see than to describe. Each approach puts the signal somewhere different inside the same model:

![Four diagrams showing where a signal can hide inside the same model: imperceptible noise in texture pixels, concentric rings bending the surface normals, bits notched into the mesh silhouette, and a bold visible stamp — each annotated with the render stage that destroys it.](/assets/images/posts/four-hiding-places.png)

The rest of this section is the detail behind each row.

### 1. TrustMark in the base color texture

TrustMark (Adobe, ICCV 2025) is a strong neural image watermark. The obvious first move: embed it in the model's largest `baseColorTexture` and decode from the screenshot.

Result: 29 bit errors in a 61-bit payload. TrustMark's BCH_5 error correction can fix 5. The decoder wasn't degraded — it was outputting essentially random bits, because it had never seen 3D-rendered input during training.

I tried every mitigation I could think of:

```
WM_STRENGTH 1 -> 5 -> 50             -> still 20+ bit errors (50 visibly ruins texture)
Tiled watermark (16 tiles/texture)   -> every tile dies through the render
TrustMark bbox detector + localize   -> finds regions, decodes garbage
42-crop sliding window               -> no crop helps, signal is gone everywhere
Multi-shot logit averaging (5 fr.)   -> confidence dropped 0.12 -> 0.08
```

That last row is the tell. I averaged raw pre-threshold decoder logits across frames — mathematically the right way to pull signal out of noise. The confidence went down. I wasn't averaging a weak signal against noise; I was averaging noise against noise. The signal wasn't attenuated, it was absent.

TrustMark was trained against 2D attacks: JPEG, crop, resize, brightness. 3D rendering is not in that distribution. This is a domain mismatch, not a tuning problem.

### 2. The same watermark in every texture

Embedding into all textures instead of just the largest one. Same failure, more compute. Broadening a distribution mismatch doesn't fix it.

### 3. Normal map steganography (FM concentric rings)

This one I still think is the most interesting idea, and it's where I spent the most time.

The insight: a texture watermark is pixel values, which the renderer overwrites. A normal map watermark changes how the surface interacts with light. The renderer has no choice but to compute that physically — the signal rides on the lighting model rather than fighting it.

I encoded a 24-bit ID as frequency-modulated concentric rings in the normal map's B (Z) channel. Bit 1 → 18px ring spacing, bit 0 → 8px. Concentric rings are rotationally invariant, so spinning the model doesn't change them, and a circle under perspective becomes a mathematical ellipse that `cv2.fitEllipse` can un-warp.

File round-trip: 0 bit errors. Verified on a standard glTF test asset and on a 39MB scan of Petäjävesi Old Church.

Screenshot decode: complete failure. Radial gradient variance across the render was uniform (~40 std at all radii). There was no ring-shaped peak in the gradient signal — the pattern simply wasn't converting into visible lighting bands.

Debugging it surfaced a satisfying string of real bugs — a double-peak from the cosine bump profile creating two Sobel edges per ring; an encoder/decoder mismatch on `MAX_RING_RADIUS` that made every bit past the midpoint garbage; a capacity overflow (48 bits × 18px = 864px against a 460px radius budget); and the embed writing rings into every texture including base color, fixed by reading `mat.normalTexture.index` from the glTF material metadata.

Fixing all of them made the GLB path clean and the screenshot path no better. The root cause was physical, not algorithmic: `BUMP_AMPLITUDE` was 0.45 against ambient-dominated scene lighting. Ambient light produces no shadows from normal perturbation — only directional light does. The signal needed a lighting condition I didn't control.

### 4. Silhouette notches

If texture and normals both get destroyed, encode in the one thing every renderer must reproduce: the outer contour. I cut 8 bits as deep notches (`NOTCH_DEPTH = 0.55` of local radius) into the mesh silhouette, facing the canonical camera, and decoded by measuring the half-width profile of the thresholded silhouette.

The notches rendered correctly. Decoding failed anyway, for a reason specific to this domain: heritage models have complicated silhouettes. My two test assets were a scanned bust and the Heidentor, a Roman gate in Carnuntum, Austria. The bust's jawline and the gate's cornices produce natural dips in the width profile that are indistinguishable from intentional notches. There's no threshold separating signal from architecture. On a smooth sphere it would work fine — which is exactly why testing on real Europeana assets mattered.

### 5. A visible QR stamp in the texture

Abandon subtlety entirely: stamp a high-contrast 9×9 QR-style block grid, with three finder squares for perspective correction, directly into the base color texture. Visible by design — visibility being the price of surviving a render.

It failed for a reason I hadn't anticipated. Heritage textures are tightly packed UV atlases. A photogrammetry scan's texture isn't a neat unwrapped layout; it's hundreds of irregular shells jammed together. A QR stamped at any single location gets fragmented across many small, scattered, non-adjacent surface patches on the actual 3D model. The finder squares never appear intact in a render, so rectification never starts.

![A QR-style code stamped intact into one region of a flat texture atlas, shown mapped onto a rendered 3D model where it fragments into small scattered squares across the surface, so the finder squares never survive intact and the decoder can never begin rectifying the grid.](/assets/images/posts/qr-uv-atlas-fragmentation.png)

### 6. Canonical-view watermarking

The last idea, and the one I most wanted to work. Sidestep "survive arbitrary rendering" by fixing the viewpoint. Render server-side from the exact camera the Three.js viewer opens at (`position: [3,2,3]`, `fov: 50`), TrustMark that 2D image, then bake the watermarked pixels back into the texture. At detection time the museum photographs from roughly the same angle, and the watermark only needs to survive screenshot capture and mild angle deviation — things TrustMark handles well.

I re-ran this one while writing this article, to isolate exactly where it breaks. I embedded the asset ID `de8e56` into the 51MB Heidentor model and tried three ways of getting it back out:

```
A) canonical re-render decode         -> None
B) decode from canonical render image -> None
C) decode after JPEG q85              -> None
```

Then I tested the 2D leg in isolation — TrustMark the render, decode it immediately, before any bake-back:

```
PURE 2D: decode straight from TrustMarked render -> de8e56  OK
control: decode from clean render                -> None    OK
```

The 2D premise is sound. Correct payload recovered, clean negative control. The chain breaks precisely at the bake-back step, for two measurable reasons:

- **The residual is attenuated below the quantization floor.** The watermark residual has mean absolute value 2.549 (max 45.0). The bake-back blends at `BLEND = 0.15`, giving 0.382 — under 1, i.e. below what an 8-bit channel can even represent. The watermark is largely rounded away on write.
- **The residual lands in the wrong place.** My bake-back blends a screen-space 512×512 residual onto five 1024×1024 UV-space textures with no actual UV projection — a shortcut I'd knowingly left in, with a comment next to it admitting a proper implementation would need to raytrace the UVs. Screen space and UV space are not the same space. The residual scatters across the atlas essentially at random.

![Four-step canonical-view chain: render from a fixed camera, watermark the 2D image (which decodes correctly, recovering de8e56), bake back (residual 2.549 falls to 0.382, below the 8-bit floor, and screen space is not UV space), then screenshot, which fails to decode. The first half is proven sound; the second needs real inverse-UV projection.](/assets/images/posts/canonical-view-chain.png)

That's the honest post-mortem: not a flawed premise, but an unimplemented inverse-UV projection plus a blend factor set below the representable minimum.

## The one method that actually works, and why it wasn't an option

There is exactly one published method that genuinely recovers a watermark from a screenshot of a rendered mesh:

> *Deep 3D-to-2D Watermarking: Embedding Messages in 3D Meshes and Extracting Them from 2D Renderings* — Yoo, Chang, Luo, Stava, Liu, Milanfar, Yang (Google Research), CVPR 2022 · [arXiv:2104.13450](https://arxiv.org/abs/2104.13450)

The reason it works is the reason I couldn't use it. It trains an encoder, a differentiable renderer, and a decoder jointly, end-to-end, against randomised viewpoints and lighting. The network learns where to place the signal so that rendering won't destroy it — discovering empirically what I was trying to reason out analytically.

Every one of my six methods is a hand-designed guess at "where does signal survive a render?" The neural approach doesn't guess. It puts the renderer inside the training loop and optimises through it.

Out of scope for a hackathon because: it needs GPU training over thousands of meshes, no pretrained model has been released, and it's days-to-weeks of work. You cannot `pip install` it. Knowing that in advance was worth the whole exercise — discovering it at hour six of the event would have been considerably worse.

The rest of the literature is a useful negative result in its own right. I reviewed classical mesh watermarking (Deep3DMark, AAAI 2024), point-cloud methods (SVD + PointNet++), curvature-domain and reversible oblique-photography schemes — and Gaussian-splatting work (GS-Marker, GuardSplat) which does do 3D-to-2D via differentiable rendering, but only for splats, while Twin It! ships GLB meshes. Every non-neural method extracts from the 3D file, not from rendered pixels. They answer a different question: useful if the studio hands back the GLB, useless when all you have is a screenshot.

## Reframing the idea before the event

By the end of the run-up, the honest read was: the one method that works is out of reach, and all six of my alternatives are dead. The original pitch — "a watermark that survives anything, recoverable from any screenshot" — was not something I could build in a weekend. Better to learn that at my desk than on stage.

The tempting move is to go anyway and demo under conditions quietly rigged to pass, letting the audience assume it generalises. But a demo that works only under conditions you're concealing isn't a result; it's a liability that transfers to whoever trusts it next. So instead of dropping the idea or overselling it, I reframed what it should be — into two parts:

1. **A chain demo under stated conditions.** Not "a watermark that survives everything," but a working demonstration of the full provenance chain: upload a GLB → embed a watermark → register a digital passport (source URL, institution, timestamp, methods applied) in a database → view in a Three.js viewer → capture a screenshot → upload it to a provenance endpoint → recover the record. Every link is implemented. The recovery link works under controlled conditions — a fixed canonical viewpoint, known lighting, no re-export of the model — and the honest framing is that those conditions are part of the claim, not fine print omitted from it.
2. **A documented benchmark of what actually breaks.** A methods-against-attacks matrix: six geometry attacks (decimation to 50% and 25%, vertex noise at 0.1% and 0.5%, Laplacian smoothing) and eight image attacks (JPEG at quality 90/60/30, resize to 50%/25%, centre-crop to 80%, colour jitter). Each combination is tested three ways — against the raw texture, against a rendered screenshot, and against a lossless file round-trip — which separates "the watermark broke" from "the render broke it." Plus the failure analysis above, with the numbers attached.

The second part is the more useful artifact. Anyone picking this up now knows that TrustMark loses 29 of 61 bits through a render, that normal-map rings need directional light the deployer doesn't control, that silhouette encoding dies on ornate heritage geometry, that QR stamps shatter on UV atlases, and that canonical-view needs a real inverse-UV projection and a blend factor above the quantization floor. That's maybe two days of dead ends they can skip.

## The takeaway

Hackathons reward demos, which creates pressure toward the rigged demo — pick the input where it works, don't mention the rest. Doing the feasibility work beforehand is the cheapest defence against that pressure: you arrive already knowing where the walls are, so you're choosing your scope rather than discovering it under a deadline at 3am.

Negative results are cheap to produce and expensive to rediscover. "TrustMark loses 29 of 61 bits through a Three.js render" took me a day to establish and takes you ten seconds to read. That asymmetry is the entire argument for writing them down.

There's also a research-quality point. Testing on real Europeana heritage assets rather than a benchmark sphere is what surfaced the two most interesting failures — ornate silhouettes defeating contour encoding, and packed UV atlases shattering texture stamps. Neither shows up on clean synthetic geometry. Both are fundamental to the actual domain.

And "why did it fail" is worth strictly more than "it failed." Canonical-view isn't a dead end: the 2D leg provably works, and I know the exact two things standing between it and a working method. That's a starting point, not a tombstone.
