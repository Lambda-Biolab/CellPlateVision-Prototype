# CellPlateVision

![CellPlateVision — Automated cell growth detection on Petri dishes.](.github/social-preview.png)

> **Status: Prototype / Research**

Automated cell culture growth detection on round Petri dishes using
image-based confluence estimation and electronic lab notebook integration.

## Scope

- Detect whether cell cultures grow on round Petri dishes (35/60/100 mm)
- Pipeline: image capture -> dish detection -> segmentation -> confluence -> ELN export
- Growth classification via confluence threshold
- Segmentation backends: [OpenCV][opencv] classical, [Cellpose][cellpose], [Fiji][fiji], VLM, [LandingLens][landinglens] (optional)
- Experiment tracking via [eLabFTW][elabftw] REST API

## Stack

| Layer               | Tool                                     |
| ------------------- | ---------------------------------------- |
| Dish detection      | [OpenCV][opencv] Hough circle transform  |
| Segmentation        | Otsu, [Cellpose][cellpose], [Fiji][fiji] |
| Overlap handling    | Watershed, [Cellpose][cellpose] inst.seg |
| ELN                 | [eLabFTW][elabftw]                       |
| Alt. no-code (opt.) | [LandingLens SDK][landinglens-sdk]       |
| Alt. interactive    | [Fiji][fiji] (manual / macro batch)      |

## Docs

See [docs/outline.md](docs/outline.md) for architecture, analysis, and outlook.

## License

Apache-2.0 — see [LICENSE](LICENSE)

## References

[cellpose]: https://github.com/MouseLand/cellpose
[elabftw]: https://www.elabftw.net/
[fiji]: https://fiji.sc/
[landinglens]: https://landing.ai/landinglens
[landinglens-sdk]: https://pypi.org/project/landingai/
[opencv]: https://opencv.org/
