---
title: CellPlateVision — Feasibility Assessment
description: Gate decision for scaffolding the automated Petri dish confluence estimation prototype
issue: Lambda-Biolab/CellPlateVision-Prototype#2
created: 2026-04-23
status: assessment
---

## Executive Summary

**Verdict: GO-WITH-CAVEATS**

The core pipeline (Hough circle detection → Otsu segmentation → confluence ratio → eLabFTW REST) is technically sound for a research prototype using round Petri dishes under controlled imaging. All primary dependencies are open-source with permissive licenses. The main caveats are imaging-condition sensitivity, the optional LandingLens path requiring a paid account, and the need for representative sample images before parameter tuning can begin.

---

## Per-Component Feasibility

### 1. Dish Detection — OpenCV Hough Circle Transform

**Verdict: Feasible with tuning required**

`cv2.HoughCircles` is well-established for detecting circular objects in grayscale images. For standard round Petri dishes (35/60/100 mm) photographed from above:

- Works reliably when the dish edge has sufficient contrast against the background.
- Sensitive to three parameters: `dp` (accumulator resolution), `minDist`, `param1`/`param2`. These need per-setup calibration.
- The fallback (contour detection + `minEnclosingCircle`) provides a safety net for low-contrast edges.
- Multi-dish layouts on a single image require pre-filtering by radius range per plate size.

**Confidence: High** for single-dish, fixed-imaging-setup scenarios. Medium for ad-hoc phone photography without a fixed rig.

### 2. Segmentation — Otsu + Cellpose / Fiji

**Verdict: Mature pipeline, appropriate toolchain**

| Backend | Assessment |
|---|---|
| Otsu thresholding | Proven, fast, zero dependencies beyond OpenCV/scikit-image. Adequate for binary growing/not-growing at low-medium confluence. Degrades when cell-to-background contrast is low (e.g. subconfluent or stained cultures). |
| Cellpose (cyto3) | State-of-the-art for cell segmentation, handles overlap via instance seg. GPU optional. Model is freely available under BSD 3-clause license. `pip install cellpose` — no registration required. Adds ~500 MB model weight download. |
| Fiji / IJ macro | Interactive validation path is low-risk. `TrackMate-Cellpose` plugin bridges both. No new risk here. |
| Watershed (scikit-image) | Standard, dependency already implicit in any scipy/skimage install. |

The tiered approach (classical first, Cellpose on demand) is architecturally correct — start simple, escalate only when accuracy demands it.

**Confidence: High**

### 3. Confluence Estimation — `cell_pixels / dish_pixels`

**Verdict: Valid heuristic for the stated goal**

For a binary "is culture growing?" classifier, total covered area divided by dish area is the standard confluence metric used in commercial systems (IncuCyte, Celigo). Key considerations:

- The ratio must exclude the dish border mask (the Hough circle ROI crop handles this correctly as designed).
- Otsu can misclassify scratches, condensation, or medium color variation as cells — morphological opening (erosion + dilation) should be applied post-threshold to remove speckle noise.
- At very low confluence (<5%) or near 100%, Otsu often fails; Cellpose or a minimum-area filter improves reliability.
- No independent count of individual cells is required for the growth classification goal — area-based confluence is sufficient and well-validated in literature.

**Recommendation:** Add a morphological cleanup step (2–3 px erosion) to the Otsu path before computing the ratio.

**Confidence: High** for the stated use case.

### 4. eLabFTW REST API Integration

**Verdict: Feasible, API is stable**

eLabFTW v4.x exposes a versioned REST API (`/api/v2/`). The three operations used in the design are all part of the stable v2 surface:

- `POST /api/v2/experiments/{id}/uploads` — file attachment (image, CSV result)
- `PATCH /api/v2/experiments/{id}` — body/metadata update
- `GET /api/v2/experiments/{id}` — metadata retrieval

Authentication is via Bearer token (API key generated per user in the eLabFTW UI). The token model is simple and compatible with a Python `requests`-based client. No OAuth flow required for self-hosted instances.

**Caveat:** The exact endpoint behaviour depends on the deployed eLabFTW version. Self-hosted instances may lag behind the upstream release. The integration should target the v2 API with version-check validation at startup.

**Confidence: High** for self-hosted eLabFTW 4.x. Verify server version before coding the client.

### 5. LandingLens SDK (Optional Path)

**Verdict: Viable but gated on subscription**

The `landingai` Python package is available on PyPI (Apache-2.0 SDK license). However:

- Inference requires a Landing.ai cloud account and a trained model endpoint.
- Free tier exists but is limited; production-scale inference requires a paid plan (pricing not public, requires sales contact as of early 2026).
- Model training requires annotated images uploaded to their platform — no local training.
- This path adds a cloud dependency and potential data-residency concern for sensitive biological data.

**Recommendation:** Treat LandingLens as a stretch goal / no-code demo path only. Do not block the main pipeline on it. Document the endpoint-ID and API-key requirements clearly so a user can opt in.

---

## Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Imaging variability (lighting, camera angle, condensation) causes Hough / Otsu failures | High | High | Require fixed imaging rig (e.g. a 3D-printed phone holder over a light pad) as a documented prerequisite; provide parameter-tuning notebook |
| R2 | Cellpose model unavailable offline / slow download in lab network | Medium | Medium | Bundle or pre-download `cyto3` weights; document offline install path; classical Otsu always available as fallback |
| R3 | eLabFTW API version mismatch on self-hosted instance | Medium | Medium | Version-check at startup; document minimum supported eLabFTW version (≥4.0) |
| R4 | LandingLens pricing or data residency blocks adoption | Medium | Low (optional path) | Keep LandingLens fully optional; no core functionality should depend on it |
| R5 | Hough circle detection fails for non-round plates (e.g. rectangular well plates) | Low (out of stated scope) | Medium | Scope is explicitly round Petri dishes; document as out-of-scope; note contour fallback handles mild ellipticity |
| R6 | Cell confluency metric inaccurate at extreme values (<5% or >95%) | Medium | Low | Document operating range; recommend Cellpose backend for edge cases |

---

## Prerequisites Before Scaffolding

The following must be resolved before Python code scaffolding begins:

1. **Sample images (REQUIRED):** At least 5–10 representative dish images covering:
   - Different confluence levels (0%, ~30%, ~70%, ~100%)
   - Different plate sizes used in the lab (35 mm, 60 mm, 100 mm)
   - The actual camera/rig setup that will be used in production
   Without these, parameter calibration for `HoughCircles` and Otsu cannot be validated.

2. **eLabFTW server access (REQUIRED):** Confirm the deployed eLabFTW version and that API v2 is enabled. A test API key and a staging experiment ID are needed to validate the integration before it is wired into the pipeline.

3. **Target platform (REQUIRED):** Confirm whether the pipeline will run on:
   - A dedicated lab workstation (affects GPU/CPU decision for Cellpose)
   - A shared server / Docker container
   - Researcher laptops (affects dependency isolation strategy)

4. **Cellpose GPU availability (NICE TO HAVE):** If GPU is available, `cellpose` installs with CUDA support. If CPU-only, inference is slower (~10–30s per image) — acceptable for a prototype but worth documenting.

---

## Next Steps if GO

1. Obtain sample images meeting the prerequisites above.
2. Scaffold Python project: `pyproject.toml`, `src/cellplatevision/`, `tests/`, `Makefile`.
3. Implement and test dish detection module against sample images with a parameter sweep notebook.
4. Implement Otsu segmentation + morphological cleanup + confluence ratio.
5. Validate eLabFTW API calls against staging instance.
6. Wire pipeline end-to-end with a CLI entry point.
7. Add Cellpose backend as an opt-in flag (`--backend cellpose`).
8. Document imaging rig setup requirements for reproducible results.

---

## Open Questions for User

1. **Imaging setup:** Is there an existing fixed rig, or will researchers photograph dishes ad-hoc (phone, different angles, ambient light)? This is the single highest-impact variable for pipeline reliability.
2. **eLabFTW version:** What version is deployed? Is API v2 enabled? Is it self-hosted or cloud?
3. **Dish types in scope:** Are rectangular multi-well plates (6-well, 12-well, 24-well) expected now or in the near-term? They require a different detection approach.
4. **Cellpose licensing context:** The BSD 3-clause Cellpose license is permissive for research. If this moves toward a commercial product, confirm IP clearance with the institution.
5. **LandingLens interest level:** Is this a genuine alternative path the team wants to pursue, or is it a placeholder? Clarifying this avoids dead-end scaffolding.
