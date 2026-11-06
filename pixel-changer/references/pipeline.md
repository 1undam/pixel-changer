# Pipeline reference

How each stage of `assets/pixel-changer.html` works, and why. Read the stage you are about to change.

## Contents

1. Background
2. Grid
3. Cell colors
4. Palette
5. Cleanup
6. Removed colors, pinned colors, overrides
7. Data the stages share

## 1. Background (`computeBackground`)

Produces `S.bg`, one byte per source pixel where 1 means background, and `S.bbox`, the bounding box of everything that is not background.

- Pixels with alpha under 128 are always background.
- If more than half of the border pixels are transparent, the image already has its own transparency and no color is removed.
- Otherwise, in 자동 mode, the background color is the most common border color, found with a 5-bit-per-channel histogram and averaged.
- A pixel matches when its RGB distance to that color is within "허용 오차".
- With "테두리와 이어진 배경만" on, matching is a flood fill from the border. This keeps a black outline that touches a black background, and eyes of the background color inside the subject.

If everything turns out to be background, nothing is removed. An empty result is worse than an unremoved background.

The source pane draws `S.srcViewCanvas`, the original with background pixels faded, so the user can see what was removed.

## 2. Grid (`computeGrid`, `detectGrid`)

Produces `S.xs` and `S.ys`, the cell boundaries in source pixels. They are strictly increasing integers, and `S.W = xs.length - 1`.

### Edge profiles

`edgeProfiles()` counts, for every column boundary, how many rows have a color change across it, and the same for rows. A change is an RGB distance above "색 경계로 볼 차이", or a transition between background and subject. Background-to-background pairs are ignored. The profiles are smoothed over ±1 px before scoring, because blur and jitter split one edge across two columns.

### Choosing the cell size

For each candidate size in "찾을 칸 크기 범위" (steps of 0.25 px below 12, then 0.5) and each offset, lines are laid at `origin + offset + k·size` and the mean profile value on those lines is measured.

The score is a t statistic:

```
score = (mean on lines − mean overall) / standard deviation × √(number of lines)
```

The `√n` term is what makes this work. A size twice the true one lands on the same edges with half as many lines, so it scores about 0.7 of the true size. A size half the true one puts every other line on empty space, which halves the mean, so it also scores about 0.7. An earlier version normalized by the profile maximum with no line count, and it picked 63 px on an image whose pixels were 9 px, because a handful of lines on the strongest edges looked perfect.

Width and height share one size, since pixels are square, and each axis gets its own offset. After the best score is found, a divisor of 2 or 3 replaces it only if it scores at least 95% as well.

"경계 일치 %" is a separate number for display: how far the mean on the lines sits between the overall mean and the maximum.

### Snapping (`snapLines`)

A uniform lattice drifts off an image whose pixels vary in width. Each interior line moves to the strongest edge within "최대 이동" of its ideal position, with nearer edges weighted higher. The ideal positions come from the global lattice, not from the previous snapped line, so errors do not accumulate. Lines are then forced to stay increasing, at least half a cell apart, and inside the image. Lines pushed past the end are dropped.

### Other modes

- **수동:** the lattice is built from the typed cell size and offset. Offsets are absolute image coordinates.
- **결과 크기 고정:** the bounding box is divided evenly into the requested columns and rows.

Snapping applies in all three modes when the cell is at least 3 px.

`latticeLines()` starts one line at or before the bounding box and ends one at or after it. The outer cells are mostly background and usually come out transparent.

## 3. Cell colors (`computeCells`)

Produces one color per cell in `S.cellRGB` and `S.cellLab`, and `S.cellA`, which is 1 for an opaque cell.

- A cell is transparent when its non-background share is under "투명 기준".
- Sampling uses the cell inset by "칸 테두리 무시" on each side, as long as at least 2 px remain. Border pixels are where neighboring colors bleed in.
- **살짝 모은 뒤 가장 많은 색** quantizes the samples into buckets of "색 묶기 간격" per channel, takes the fullest bucket, and returns the average of the pixels in it. It is the default because it ignores a minority color entirely where an average would mix it in.
- The other methods are the per-channel median, the center pixel, the plain average, and a Gaussian center-weighted average.

## 4. Palette (`computePalette`, `assignCells`)

The palette is built from cell colors, not source pixels. Noise has already been voted out by then, and the palette has far fewer points to cluster.

`colorEntries(q)` merges cell colors into quantized entries, each with a count, an average RGB and an OKLab value. The methods cluster those entries:

- **k-평균 (OKLab):** deterministic. It starts from the most common entry and adds the farthest entry weighted by √count, then runs up to 30 iterations.
- **OKLCH 묶기:** greedy, most common first. An entry joins a center when lightness, chroma and hue all fall within the ranges, and hue is ignored for near-greys. The ranges decide the color count.
- **중앙 분할:** weighted median cut in OKLab.
- **원래 색 유지:** every distinct cell color, up to 256.

"팔레트 색" decides what represents a cluster. **원본 색** uses the most common member, a color that really occurs in the image. **평균 색** uses the weighted mean, which lets blurred edge cells drag a white toward grey.

Near-identical results are merged. With "어두운 색을 외곽선 색 하나로" on, palette colors at or below the lightness limit collapse into the most used one.

### Assignment (`choosePalette`)

Each opaque cell takes a palette color.

- The distance is OKLab, or the hue-weighted form `√(ΔL² + ΔC² + w²·ΔH²)`, where `w` is "색상 무게".
- With "명암 계열 우선" on, the cell first looks for palette colors of its own family: hue within 20° and chroma within 0.08, or both near-grey. Among those it takes the one nearest in lightness, provided that color is not more than 2.2 times farther than the plain nearest color. This keeps a darker patch of skin from becoming gold because gold happened to be closer in lightness.

`S.base` holds the result and `S.baseWhy` records whether the family rule decided it. The color-decision panel reads both.

## 5. Cleanup (`cleanup`)

Works on palette indices and writes `S.clean` and `S.cleanWhy`.

- **외톨이 픽셀 지우기:** a pixel whose opaque neighbors, three or more, all share one other color takes that color. It is left alone when the contrast with them exceeds the limit, so pupils and highlights survive.
- **1px 구멍 메우기:** a transparent cell with four opaque neighbors takes their most common color.

Both read the pre-cleanup grid, so one fix cannot trigger the next.

## 6. Removed colors, pinned colors, overrides (`finalFor`, `computeFinal`)

These run after everything else and never re-run the palette.

- **Removed colors (`S.removed`):** a cell whose color was removed takes the nearest remaining palette color, measured from its own cell color.
- **Pinned colors (`S.fixed`):** a cell switches to a pinned color only when its cell color is closer to that than to its current color. Pinned colors use indices from `palette.length` upward.
- **Overrides (`S.overrides`):** painted cells, keyed by cell index, holding an RGB triple or `null` for transparent. They store colors rather than palette indices so they survive a re-run. They are cleared when the result size changes.

`S.finalIdx` and `S.finalWhy` hold the outcome before overrides. `cellFinalRGB(k)` is the one place that answers "what color is this cell".

## 7. Data the stages share

| Field | Meaning |
|---|---|
| `S.src`, `S.srcW`, `S.srcH` | Source `ImageData` and size |
| `S.bg`, `S.bbox` | Background mask and subject bounding box |
| `S.xs`, `S.ys`, `S.W`, `S.H` | Cell boundaries and result size |
| `S.cellRGB`, `S.cellLab`, `S.cellA` | Per-cell color and opacity |
| `S.palette`, `S.palLab` | Palette colors |
| `S.base`, `S.clean`, `S.finalIdx` | Palette index per cell after assignment, cleanup, and post-processing |
| `S.resImg`, `S.resCanvas` | The W×H result that is displayed and exported |
| `S.set` | Every setting |
| `S.found` | Last auto-detected cell size and grid origin |

Cell index is `k = j·W + i`.
