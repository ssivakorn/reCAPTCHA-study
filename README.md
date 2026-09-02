# reCAPTCHA Study

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey?logo=creativecommons&logoColor=white)](LICENSE)
[![Validate Dataset](https://github.com/ssivakorn/reCAPTCHA-study/actions/workflows/validate.yml/badge.svg)](https://github.com/ssivakorn/reCAPTCHA-study/actions/workflows/validate.yml)
![Challenges](https://img.shields.io/badge/reCAPTCHA-1000%20challenges-green?logo=google&logoColor=white)


## Repository contents

- **`dataset/`**: 1,000 labeled Google reCAPTCHA visual challenges (662 Type A, 338 Type B), each with the instruction text, extracted keyword, tile images, and human-annotated ground-truth labels. Also available on [Kaggle](https://www.kaggle.com/datasets/ssivakorn/recaptcha-challenge).
- **`reCAPTCHA-solver/SKILL.md`**: the natural-language classification prompt (Claude Code skill) used for the paper's prompt-only evaluation. See [Reproducibility](#reproducibility).
- **`scripts/`**: dataset validation (`validate.py`).
- **`images/`**: example challenge figures.

## Intended Use

These materials are published for research purposes, including CAPTCHA
robustness/security research, accessibility research, and academic study
of visual challenge design. They are not intended for use in building or
operating automated systems that solve reCAPTCHA challenges to defeat
bot-detection in production (e.g. account creation abuse, scraping,
spam). Such use is likely to violate Google's reCAPTCHA and general
Terms of Service regardless of this repository's license, and is outside
the intended use of this release.

## Challenge Types

### Type A: Independent Image Tiles
The user receives a 3×3 grid of nine independent tiles and must select
all tiles containing the target keyword. Type A has two sub-variants:
- **Static:** All matches must be identified in a single pass
- **Dynamic:** Correctly clicked tiles refresh with new images; the
  challenge loops until no matches remain

<p align="center">
  <img src="images/type_a_challenge.png" alt="Type A challenge example" width="400"><br>
  <em>Example Type A challenge: "Select all images with a fire hydrant", nine independent tiles, three containing hydrants.</em>
</p>

### Type B: Single Image Grid
The user receives a single photograph split into a 4×4 grid of 16 tiles
and must select all tiles containing the target keyword. All matches must
be identified in a single pass with no tile refresh.

<p align="center">
  <img src="images/type_b_challenge.png" alt="Type B challenge example" width="400"><br>
  <em>Example Type B challenge: "Select all squares with traffic lights", a single photo split into a 4×4 grid.</em>
</p>


## Data Collection

Challenges were collected by visiting real-world websites with Google
reCAPTCHA deployed, each time using a fresh browser session with no
prior browsing history or cookies. Collection took place in mid-2026.

## Annotation

Each challenge was manually labeled by a human annotator using a custom
web interface. The annotator recorded which tiles constitute correct
answers based on the challenge instruction. Labels reflect strict
exact-match criteria consistent with reCAPTCHA's own acceptance standard.


## File Formats

### `info.json`
Each challenge folder contains an `info.json` file with the following fields:

```json
{
  "instruction": "Select all squares with\ntraffic lights\nIf there are none, click skip",
  "keyword": "traffic lights",
  "correct_answers": [1, 3, 5, 7]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `instruction` | string | Full instruction text shown to the user |
| `keyword` | string | Extracted target object keyword |
| `correct_answers` | list[int] | Zero-indexed tile indices that are correct answers |

### Tile Images
- **Type A:** 9 tiles (`tile_0.png` to `tile_8.png`), each an independent image
- **Type B:** 16 tiles (`tile_0.png` to `tile_15.png`), each a crop of `full_images.png`
- Tile indices follow left-to-right, top-to-bottom order

**Type A (3×3):**

```
| 0 | 1 | 2 |

| 3 | 4 | 5 |

| 6 | 7 | 8 |
```
**Type B (4×4):**

```
|  0 |  1 |  2 |  3 |

|  4 |  5 |  6 |  7 |

|  8 |  9 | 10 | 11 |

| 12 | 13 | 14 | 15 |
```

## Validation

`scripts/validate.py` checks every folder in `dataset/` against the schema documented above and runs automatically on every push via [GitHub Actions](.github/workflows/validate.yml). For each challenge it verifies:

- `info.json` is present and parses as valid JSON
- `instruction` and `keyword` are strings, and `correct_answers` is a list
- the tile file count is either 9 (Type A) or 16 (Type B)
- the tile files present exactly match `tile_0.png` ... `tile_N.png` for that count, with no gaps or extras
- every index in `correct_answers` is an integer within range for the folder's tile count

It does not validate image content (e.g. that a tile actually matches its label), only structural/schema correctness.

```
python3 scripts/validate.py
```

## Reproducibility

This dataset accompanies the paper *"Robot Visions: Breaking reCAPTCHA at Zero Cost and Zero Shot"* (ISC 2026).

[`reCAPTCHA-solver/SKILL.md`](reCAPTCHA-solver/SKILL.md) is the natural-language classification prompt (a Claude Code skill) used for the paper's prompt-only evaluation. It instructs a multimodal AI assistant to read each challenge's tiles and classify them for the target keyword. It contains no code and no browser automation, and it operates on the offline challenges in `dataset/`.

The reported prompt-only result (98% on a 50-challenge sample) used Claude Sonnet 4.6 via Claude Code in its default configuration; extended thinking was not enabled.

To limit misuse, the end-to-end online solver (which additionally automates browser interaction) and the solver source code are not released.

## License

Copyright (c) 2026. This dataset is released under [CC BY-NC 4.0](LICENSE) (Attribution-NonCommercial). You may share and adapt it with attribution, for non-commercial purposes only.
