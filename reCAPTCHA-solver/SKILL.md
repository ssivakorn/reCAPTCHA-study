---
name: challenge-solver
description: Solve reCAPTCHA image challenges by classifying tiles (Type A and Type B)
---

# challenge-solver

Solve reCAPTCHA image challenges by classifying tiles.

## Arguments

- First argument: path to a challenge directory or a parent folder containing multiple challenges
- Optional `--threshold N`: coverage percentage threshold for Type B challenges (default: 0, i.e. any visible coverage counts)

## Instructions

1. **Determine scope**: If the path contains `info.json`, solve that single challenge. Otherwise, find all subdirectories with `info.json` and solve each that has `correct_answers` for comparison.

2. **Read the challenge**: Load `info.json` to get `keyword`, `instruction`, and `correct_answers`.

3. **Detect challenge type** from the instruction text:
   - **Type A** (independent images per tile): instruction contains "images" (e.g., "Select all images with a fire hydrant")
   - **Type B** (single image split into tiles): instruction contains "squares" (e.g., "Select all squares with stairs")
   - Fallback: 9 tiles = Type A, 16 tiles = Type B

4. **Solve based on type**:

   **Type A**: For each `tile_N.png`, use a two-step judgment rather than jumping straight to yes/no; a tile is independent of its neighbors, so judge it on its own content only.
   1. **List the objects first, generically.** Before thinking about the keyword at all, write out a flat list of the concrete objects/things visible in the tile — plain nouns, one item per thing a person would point at (e.g. "shopfront", "potted plants", "AC unit", "bicycle"), not a narrative sentence. Listing items individually, without describing how prominent or where each one is, keeps you from folding a judgment about importance into the noticing step. It also naturally keeps out things that aren't the object itself — a shadow, a reflection, the pole something is mounted on — since those get listed as what they are, not as the object.
   2. **Then check whether the keyword appears in that list** (or is clearly one of the listed items), and judge it as one of:
      - **Clear** — the object is genuinely there, *and* it's a prominent item in the tile: sized and placed the way the other clearly-listed objects are, not noticeably smaller or more peripheral than them. A person glancing at the tile would name it as one of the tile's main things, not something they'd have to point out specifically.
      - **Ambiguous** — the object is genuinely present (you're not guessing) but small, distant, or incidental relative to the rest of the scene, or you only noticed it by zooming in and reasoning hard about it. Being *certain* it's there does not make it Clear — confidence and prominence are different questions; a small object you're 100% sure about is still Ambiguous, not Clear, if it isn't one of the tile's main subjects.
      - **Absent** — not there.

   Then decide the tile set from those judgments. In this dataset, Type A challenges overwhelmingly (96%+) have **exactly 3** correct tiles — treat 3 as the expected count, not just a threshold:
   - Take every tile judged **Clear**.
   - **If that gives 3 or more tiles, use exactly that set — do not add any Ambiguous tiles**, even ones that seem plausible on a second look. A challenge with several clear matches doesn't need borderline ones added, and reCAPTCHA's ground truth consistently leaves these out when clear matches are already sufficient.
   - **If fewer than 3 tiles are Clear, you are very likely missing a real match — don't just include an Ambiguous tile if it happens to seem likely, actively go looking for it.** Rank the Ambiguous tiles by how likely each is to be genuine, and promote your best candidate(s), one at a time, until you reach 3 total. Only stop short of 3 if you've genuinely run out of Ambiguous candidates — every remaining tile is clearly Absent with nothing left to reconsider — not because none of them individually felt "more likely than not."

   Return the list of tile indices that contain the object.

   **Keyword-specific note — "bus".** reCAPTCHA's ground truth for "bus" is broader than the classic transit/coach/school-bus image: it also counts large, boxy, integrated-cab vehicles that a person would naturally call something else at first glance — motorhomes/RVs, box trucks, and step vans included. When listing objects for a tile under the "bus" keyword, don't stop at "truck" or "RV" as a final answer — note the vehicle's actual shape (a single continuous boxy body with an integrated cab and side windows, as opposed to a flatbed, dump bed, tanker, or small pickup with a separate open cargo area) and lean toward counting it as the target if it has that bus-like silhouette, even when your first instinct names it as a different vehicle type.

   **Type B**: Do not judge tile boundaries by eyeballing pixel coordinates on the downscaled `full_images.png` composite — this is the single biggest source of error, because a small misjudgment of where a row/column boundary falls silently shifts a tile's content by a whole row or column. Instead, reassemble the individual `tile_N.png` crops into their 4x4 grid (numbered left-to-right, top-to-bottom) with gridlines and index labels drawn on; the crops are ~2.8x higher resolution than the composite and this reassembly makes each tile's true content and boundaries unambiguous. For any tile you're still unsure of, open its own `tile_N.png` crop directly and judge it in isolation from its neighbors. Then, for each tile, estimate what percentage of its area is covered by the keyword object and select tiles where coverage >= threshold argument (default 0, i.e. any visible coverage at all qualifies).

   Five conventions that reCAPTCHA's ground truth follows, and that fix the dominant Type B errors:
   - **Any coverage counts.** With the default threshold of 0, select a tile the moment any recognizable part of the object appears in it, however small — a corner, an edge, a sliver clipped from a neighboring tile. Under-selecting a boundary tile the object only partially enters is the single most common mistake. When genuinely uncertain about a borderline tile, prefer to include it rather than omit it. (Still exclude tiles with no visible trace of the object at all — over-selecting empty tiles also fails.)
   - **Shadow alone does not count.** A tile showing only the object's cast shadow or reflection, with no part of the object's own body visible in that tile, is not a match — do not select it. Only select a tile once some physical part of the object itself (not just its shadow) appears in it.
   - **A second instance's inclusion is genuinely inconsistent — judge it case by case, don't apply a blanket default.** When more than one instance of the object is visible (a second rider, vehicle, or signal that is smaller, farther away, or cut off at the edge), reCAPTCHA's ground truth sometimes includes it and sometimes doesn't, with no reliable rule found so far for which way a given case goes — testing has confirmed both "always include extra instances" and "always exclude distant ones" wrong about as often as each was right. Don't default hard in either direction. What's not in question: a tile with a clear lookalike that isn't the target object (stray lane paint that isn't the crosswalk, a different vehicle type) should still be excluded; the uncertainty is specifically about genuine additional instances of the correct object.
   - **Trace the rider's column upward, all the way to the top of their head.** For a ridden vehicle (motorcycle, bicycle), reCAPTCHA labels the whole object-plus-rider blob, and the dominant error here is under-counting *upward*: a rider's helmet/head is very often the tile content nearest the top edge of the frame, one or more rows above where the vehicle body itself sits, and it is easy to stop tracing the object too early and miss those top tiles. The test for whether a head/helmet tile belongs to the object is *continuity*, not whether the vehicle happens to also be visible in that same tile: if the tile sits directly above another tile that is part of this same rider-and-vehicle (their shoulders, torso, or the vehicle continuing the same column downward with no gap), include it, even though only the head/helmet appears in it and nothing of the vehicle does. Only exclude a head/helmet tile when it is genuinely detached — separated by background with no such continuation below it (e.g. a distant, different rider's head that doesn't connect down into this vehicle's own tiles).
   - **Exclude the mount, not just the object's own body.** Select only the object's own visible body — not the pole, arm, bracket, or other hardware it is mounted on (e.g. a traffic light's housing counts, the signal pole and mounting arm above it do not), and not the ground-level shadow or undercarriage tiles directly beneath a vehicle once its visible body has ended. For "stairs" specifically, this means a tile showing only the handrail/railing with no visible tread or step surface is usually not counted. Note that this convention is inconsistently applied in practice (a tile clearly touched by a wheel, or a handrail-only stairs tile, is sometimes still included in the ground truth despite the above) — treat it as a useful default, not a guarantee, and don't be surprised when a seemingly-identical tile goes the other way in a different challenge.

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
