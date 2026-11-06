---
name: pixel-mosaic
description: Turn an AI-generated or ordinary image into clean pixel art with a bundled single-file browser tool (automatic pixel-grid detection, background removal, per-cell color picking, palette reduction, color remove and pin, manual painting, PNG export). Use this whenever the user wants to pixelate an image, clean up "pixel-looking" AI art that has uneven pixel sizes, blurry edges or hundreds of colors, convert a picture into a sprite or dot art, snap an image to a grid with a limited palette, or mentions 도트, 픽셀 아트 변환, 모자이크, "AI 그림 픽셀로 정리" or a Chickxel-style pixelate feature, even if they never ask for a tool by name. Also use it to explain or tune the tool's settings, to diagnose a bad conversion, or to change the tool's code.
---

# Pixel Mosaic

A single HTML file, `assets/pixel-mosaic.html`, that converts an image into pixel art in the browser. Nothing is uploaded and there is no build step or dependency. The user loads an image, the tool finds the pixel grid, and the result appears next to the source for cell-by-cell comparison.

It exists because AI image models draw "pixel art" that is not on a grid: the pixels are 7 to 9 px wide instead of a constant size, edges are blurred, and there are hundreds of near-duplicate colors. Downscaling such an image gives mush. This tool recovers the grid the image was meant to have and picks one color per cell.

## Giving the tool to the user

Do not regenerate the tool from memory. The bundled file is tested, and a rewrite will lose behavior that took several rounds to get right. Hand over the file as it is.

- **If you can publish an Artifact:** copy `assets/pixel-mosaic.html` into your working directory and publish that copy. The file already follows the Artifact page contract: it has no doctype, `html` or `body` tags, the `<title>` is at the top, and every color is a theme token with light and dark values.
- **Otherwise:** copy the file into a folder the user chooses and tell them to open it in a browser by double-clicking it. It works offline. Only the two web fonts need a connection, and system fonts take over without them.

Tell the user which one to prefer for saving. Inside the Artifact viewer, downloads and clipboard image copy can be blocked, so "PNG 만들기" shows the image for a right-click save. The local file has no such limit.

The user supplies the image in the page: the "이미지 열기" button, paste with Ctrl+V, or drag and drop. The page opens with a built-in sample so its behavior is visible before any image is loaded.

## What the user sees

The UI is in Korean.

- **Left pane, 결과:** the result as a plain W×H image. Every result pixel is drawn the same size.
- **Right pane, 원본:** the source with the detected cell boundaries drawn on it. These lines are uneven on purpose, because they follow the real color edges. If the lines sit on the edges of the color blocks, the grid is right.
- **Settings column (⚙ 설정):** five numbered stages that run in order. Changing a value re-runs the pipeline when "값 바꾸면 자동" is on, or when "▶ 정리하기" is pressed.
- **Bottom:** the color swatches, the color-decision panel for the clicked cell, and export.

The two panes zoom and pan together and are linked by cell index: clicking a result pixel marks the source cell it came from, and the reverse.

## Tuning a bad conversion

Start from the status line at the top right. It reports the detected cell size, the grid origin and "경계 일치 %", which is how well the grid lines sit on real edges. About 30% or more usually means the grid was found.

| Symptom | Likely cause | What to change |
|---|---|---|
| Result is a few huge blocks, or far too many tiny pixels | Wrong cell size was picked | Narrow "찾을 칸 크기 범위" around the real size. Raise the minimum if it picked a fraction, lower the maximum if it picked a multiple. |
| Grid lines drift off the color blocks | Detection is counting noise as edges, or missing soft edges | Raise "색 경계로 볼 차이" for noisy JPEG-like images. Lower it to 15–20 for pastel, low-contrast art. |
| Cells look right in most places but a few are half-width | Lines were pulled onto an inner detail | Lower "최대 이동 (칸의 %)", or turn off "칸 경계를 실제 색 경계에 맞춤" for art whose pixels are already uniform. |
| Auto detection keeps failing | Image has too few edges, or is a photo | Switch "칸 크기 기준" to 수동 and type the cell size and offset, or to 결과 크기 고정 for a target size such as 64×64. "찾은 값을 수동으로" copies the auto result into the manual fields for fine adjustment. |
| Background is still there, or parts of the subject vanished | Background tolerance is off | Adjust "허용 오차". Keep "테두리와 이어진 배경만" on so outline pixels that match the background color survive. |
| Colors look muddy at block edges | Cell samples include blurred border pixels | Raise "칸 테두리 무시 (%)", and keep the method on "살짝 모은 뒤 가장 많은 색". |
| Too many near-identical colors, or important small colors lost | Palette size | Change "색 수", or use "OKLCH 묶기" to let the color count follow the image. For a lost accent color, pin it with `+` instead of raising the count. |
| A shadow turned into the wrong hue | Nearest color by distance crossed hue families | Keep "명암 계열 우선" on, or raise "색상 무게". |
| Stray single pixels | Noise survived | Turn on "외톨이 픽셀 지우기" in stage 5. Lower its contrast limit if it erases eyes or highlights. |

Explain a setting in terms of what it does to the picture. Users of this tool are artists and game developers, and most do not want the math unless they ask.

## Controls worth pointing out

- **Remove a color:** right-click a swatch or a result pixel in the 선택 tool. Only the pixels of that color change, each to the nearest remaining color for its own source cell. Nothing else is recomputed, so the color count drops by exactly one. Right-click again, or "모두 되살리기", restores it.
- **Pin a color:** press `+` at the end of the swatches, then click a source cell. That cell's color joins the result as a fixed color, and only the cells closer to it than to their current color switch. Shift+click keeps picking, Esc stops.
- **Color decision:** click any cell to see each stage's color in order (cell color, palette, cleanup, color removal, pinned color, manual override), the three nearest palette colors with their distances, and the color distribution inside the cell.
- **Editing:** 칠하기 paints with the brush color, 지우개 makes cells transparent, right-click or a long press picks the color under the pointer, Ctrl+Z undoes a stroke.
- **Navigation:** wheel zooms, drag pans in the 선택 tool, the middle button pans in any tool. On a tablet, two fingers pan and pinch.

## Changing the tool

Everything is in the one file: styles, markup, then a single script wrapped in an IIFE. The script is divided by banner comments, in pipeline order:

`state` → `color math (OKLab)` → `image loading` → `1. background` → `2. grid` → `3. cell colors` → `4. palette` → `5. cleanup` → `6. removed colors / fixed colors / overrides` → `pipeline` → `palette strip` → `inspector` → `view / drawing` → `settings binding` → `export` → `sample`.

`recompute()` runs stages 1 to 5 and then `computeFinal()`. Read `references/pipeline.md` before changing a stage. It explains each algorithm and why it is built the way it is.

Keep these properties, because each one was a bug or a complaint before it was a rule:

- **The result is uniform.** The result pane draws `S.resCanvas`, a plain W×H image, scaled into the bounding box of the source grid. Only the source pane uses the uneven `S.xs` / `S.ys` boundaries. Drawing the result with the uneven boundaries makes the output look like a different, broken feature.
- **Color removal and pinned colors are post-processing.** They live in `finalFor()` and never re-run the palette. Users rely on other pixels staying exactly as they were.
- **Settings are the source of truth.** `S.set` holds every value and the inputs are filled from it at startup. Reading state from the inputs instead leads to a page that shows one number and computes with another.
- **A new setting needs only markup.** Add a control with `data-set="key"` and a default in `S.set`. The binding code picks it up.
- **`[hidden]{display:none!important}` stays in the stylesheet.** The Artifact host supplies that rule, but a local file does not, and without it the export overlay covers the page.
- **`fitView()` must tolerate a pane with no size yet.** It defers to the next resize when the pane is not laid out.

### Testing a change

The script exposes its state as `window.__pm`, so a browser check can read results directly, for example `__pm.W`, `__pm.H`, `__pm.palette.length` and `__pm.found`.

Load the page over `http://localhost`, since some browser tools refuse `file://`. On the built-in sample, a healthy run reports a 9 px cell, a 33×43 result and 9 colors, with no console errors. Then click a result pixel, right-click a swatch, and pin a color with `+`, and confirm each changes what it should.

## Limits

Say so plainly when a request falls outside what the tool does. It has no preset candidates with scores, no automatic parameter search, no 1 px outline fitting, no sprite-sheet frame splitting, no dithering, and no batch or command-line mode. Images larger than 1600 px on the long side are scaled down on load.

The tool was written independently, with the workflow of Chickxel's "AI 그림 픽셀로 정리" feature as its reference. It shares no code with Chickxel and is not affiliated with it.
