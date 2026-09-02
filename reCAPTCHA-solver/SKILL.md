---
name: challenge-solver
description: Solve reCAPTCHA image challenges by classifying tiles (Type A and Type B)
---

# challenge-solver

Solve reCAPTCHA image challenges by classifying tiles.

## Arguments

- First argument: path to a challenge directory or a parent folder containing multiple challenges
- Optional `--threshold N`: coverage percentage threshold for Type B challenges (default: 10)

## Instructions

1. **Determine scope**: If the path contains `info.json`, solve that single challenge. Otherwise, find all subdirectories with `info.json` and solve each that has `correct_answers` for comparison.

2. **Read the challenge**: Load `info.json` to get `keyword`, `instruction`, and `correct_answers`.

3. **Detect challenge type** from the instruction text:
   - **Type A** (independent images per tile): instruction contains "images" (e.g., "Select all images with a fire hydrant")
   - **Type B** (single image split into tiles): instruction contains "squares" (e.g., "Select all squares with stairs")
   - Fallback: 9 tiles = Type A, 16 tiles = Type B

4. **Solve based on type**:

   **Type A**: Read each `tile_N.png` image. For each tile, determine if it contains the keyword object. Return the list of tile indices that contain the object.

   **Type B**: Read `full_images.png`. The image is divided into a grid (typically 4x4 for 16 tiles, numbered left-to-right, top-to-bottom). For each tile region, estimate what percentage of the tile area is covered by the keyword object. Select tiles where coverage >= threshold argument (default 10%).

5. **Report results**:
   - Print the challenge type detected
   - Print predicted tile indices
   - For Type B, print per-tile coverage percentages
   - If `correct_answers` exists, compare and print CORRECT/WRONG
   - For batch mode, print summary: `X/Y correct (Z%)`

## Grid Layout Reference

```
3x3 (Type A):          4x4 (Type B):
| 0 | 1 | 2 |          |  0 |  1 |  2 |  3 |
| 3 | 4 | 5 |          |  4 |  5 |  6 |  7 |
| 6 | 7 | 8 |          |  8 |  9 | 10 | 11 |
                        | 12 | 13 | 14 | 15 |
```
