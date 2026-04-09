---
title: CellPlateVision — Architecture, Analysis & Outlook
description: Architecture, segmentation approaches, tool integration, and outlook for automated Petri dish cell growth detection
created: 2026-04-09
updated: 2026-04-09
category: architecture
validated_links: 2026-04-09
---

## 1. Problem Statement

Manual inspection of round Petri dish cell cultures is subjective and
non-reproducible. Automated growth detection requires dish boundary
identification, cell-vs-background segmentation, confluence estimation,
and traceable experiment documentation.

## 2. Architecture

```text
Image (phone / macro camera / microscope)
    |
    v
Circle detection (Hough transform)
    -> crop to dish interior
    |
    v
Segmentation
├── Classical: Otsu / adaptive threshold (fast, no GPU)
├── Cellpose cyto3 model (handles overlap, GPU optional)
├── Fiji: threshold + watershed (interactive / macro batch)
└── VLM zero-shot (API-based validation layer)
    |
    v
Confluence = cell_pixels / dish_pixels
    -> e.g. 45%
    |
    v
Classification: confluence > threshold -> "growing"
    |
    v
Log to eLabFTW (image + result via REST API)
```

## 3. Segmentation Approach Comparison

| Approach              | Complexity | Accuracy    | Overlap          | Setup                 |
| --------------------- | ---------- | ----------- | ---------------- | --------------------- |
| Classical CV (Otsu)   | Low        | Good        | None             | Python + OpenCV       |
| Fiji thresh+watershed | Low        | Good        | Watershed split  | Fiji only             |
| Cellpose (cyto3)      | Medium     | High        | Instance seg.    | Python + GPU optional |
| VLM zero-shot         | Low        | Moderate    | None             | API key only          |
| LandingLens SDK       | Low        | Medium-High | Depends on model | Python + API key      |

## 4. Dish Detection

Round Petri dish boundary detection via [OpenCV][opencv]:

- `cv2.HoughCircles` on grayscale + Gaussian blur
- Fallback: contour detection + `cv2.minEnclosingCircle`
- Output: center (x, y) + radius -> circular ROI mask

## 5. Overlapping Cells

At higher confluence, cells overlap and classical thresholding merges them.

Strategies by increasing complexity:

1. **Ignore overlap** — for "growing yes/no", total coverage area suffices
2. **Watershed** ([Fiji][fiji] / scikit-image) — distance transform + watershed
   splits merged binary regions at constriction points
3. **[Cellpose][cellpose] instance segmentation** — trained on touching/overlapping
   cells, outputs per-cell masks directly
4. **[Omnipose][omnipose]** — better for elongated / bacterial morphologies with overlap

For the initial growth detection goal, strategy 1 or 2 is sufficient.

## 6. Physical Marking

Optional physical markers to aid image registration across timepoints:

| Method              | Purpose                  | Caveat                          |
| ------------------- | ------------------------ | ------------------------------- |
| Grid inserts        | Spatial ref (e.g. ibidi) | Cost, may affect cell adhesion  |
| Fiducial dots       | Dish exterior alignment  | Must not obstruct imaging field |
| Scratch marks       | Wound healing standard   | Damages cell monolayer          |
| Printed grid dishes | Built-in coordinates     | Limited dish size options       |

For initial prototype: **no physical marking needed**. Dish edge itself
serves as the reference boundary.

## 7. Fiji Usage

[Fiji][fiji] serves as the interactive / batch tool for this project:

- `Image > Adjust > Threshold` (Otsu auto) for cell-vs-background
- `Process > Binary > Watershed` for splitting overlapping regions
- `Analyze > Analyze Particles` for cell count + area measurements
- `Process > Find Edges` for dish boundary enhancement
- IJ macro language for batch processing multiple dish images
- [TrackMate-Cellpose][trackmate-cellpose] plugin for deep learning segmentation within Fiji

## 8. Landing.ai / LandingLens (Optional)

No-code alternative via [LandingLens SDK][landinglens-sdk]:

- Upload dish images, annotate cells vs background, train model
- Supports cell colony counting, [anomaly detection][landinglens] (2026)
- Python SDK for programmatic inference (illustrative):

```python
from landingai.predict import Predictor

predictor = Predictor(endpoint_id="<id>", api_key="<key>")
result = predictor.predict(image)
confluence = sum(r.area for r in result.predictions) / dish_area
```

- Cloud-based — no local GPU needed
- Best for teams without CV/ML expertise

## 9. eLabFTW Integration

[eLabFTW][elabftw] serves as the experiment metadata and data management layer:

- Link dish images to experiments via [REST API][elabftw-guide]
- Attach analysis results (segmentation masks, confluence CSVs)
- Full changelog and revision tracking for reproducibility
- [EAP4EMSIG][eap4emsig] paper proposes direct eLabFTW integration

### Integration Pattern

```text
POST /api/v2/experiments/{id}/uploads  — attach result files
PATCH /api/v2/experiments/{id}         — update body with summary
GET /api/v2/experiments/{id}           — retrieve metadata
```

## 10. Outlook

### Multi-well plate support

- Multi-well plate navigation ([PlateViewer][plateviewer], well-level aggregation)
- Complex per-cell segmentation ([TrackMate][trackmate] tracking)
- Fluorescence channel analysis

### Further extensions

- Time-lapse growth curves (confluence % over hours/days)
- VLM-powered natural language queries ("is this dish contaminated?")
- Automated alerting when confluence exceeds threshold
- Multi-modal analysis (brightfield + fluorescence)

## 11. References

[cellpose]: https://github.com/MouseLand/cellpose
[eap4emsig]: https://arxiv.org/html/2504.00047v1
[elabftw]: https://www.elabftw.net/
[elabftw-guide]: https://doc.elabftw.net/user-guide.html
[fiji]: https://fiji.sc/
[landinglens]: https://landing.ai/landinglens
[landinglens-sdk]: https://pypi.org/project/landingai/
[omnipose]: https://github.com/kevinjohncutler/omnipose
[opencv]: https://opencv.org/
[plateviewer]: https://github.com/embl-cba/plateviewer
[trackmate]: https://imagej.net/plugins/trackmate/
[trackmate-cellpose]: https://imagej.net/plugins/trackmate/detectors/trackmate-cellpose-sam
