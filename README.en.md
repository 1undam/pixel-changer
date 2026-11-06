# pixel-changer

[한국어](README.md) | **English** | [日本語](README.ja.md)

A browser tool that turns AI-generated "pixel-looking" art or ordinary images into clean pixel art, packaged as a Claude skill (`pixel-changer`) so Claude can use it.

## What it does

Pixel art drawn by AI image models is not actually on a grid. The pixels are 7 to 9 px wide instead of one constant size, the edges are blurred, and there are hundreds of near-identical colors. Downscaling such an image turns it to mush. This tool finds the grid the picture was meant to have and picks one color for each cell.

- **Automatic grid detection**: counts where colors change to find the cell size and origin, then moves each line onto a real edge.
- **Background removal**: treats the most common border color as the background.
- **Cell color**: most common color, median, center pixel, average, or center-weighted average.
- **Palette**: k-means, OKLCH grouping, median cut, or keep the original colors. All work in the OKLab color space.
- **Cleanup**: remove isolated pixels, fill 1 px holes.
- **Remove and pin colors**: changes only the affected cells, without recomputing the rest.
- **Color decision**: click a cell to see how its color was chosen, stage by stage.
- **Manual editing**: paint, erase, eyedropper, undo.
- **PNG export**: from ×1 to ×16.

Nothing to install or build, and your image is never uploaded anywhere.

> The tool's interface is in Korean. This guide gives each Korean label with its meaning.

## Getting started

### Use the tool on its own

Download [`pixel-changer/assets/pixel-changer.html`](pixel-changer/assets/pixel-changer.html) and open it in a browser. It works without an internet connection.

### Install as a Claude skill

Copy the `pixel-changer` folder into a skills folder.

| Scope | Location |
|---|---|
| All your projects | `~/.claude/skills/pixel-changer/` |
| One project | `<project>/.claude/skills/pixel-changer/` |

Once installed, a request such as "turn this picture into pixel art" makes Claude bring out the tool, and Claude can tell you which value to change when a conversion looks wrong.

## User guide

### 1. Load an image

There are three ways.

- The **이미지 열기** (Open image) button at the top
- Paste with **Ctrl+V** anywhere on the page
- **Drag and drop** an image file onto the page

The image is converted as soon as it is loaded. The page opens with a sample picture so you can see how it behaves first.

### 2. Reading the screen

| Area | Contents |
|---|---|
| Left, **결과** (Result) | The converted pixel art. Every pixel is drawn the same size. |
| Right, **원본** (Source) | The source with the detected cell boundaries drawn on it. The lines are uneven because they follow the real color edges. |
| Left column, **설정** (Settings) | Settings for stages 1 to 5. The ⚙ 설정 button folds and unfolds it. |
| Bottom, **색** (Colors) | Swatches of the colors used in the result |
| Bottom, **색 판정** (Color decision) | How the clicked cell got its color |
| Bottom, **내보내기** (Export) | Make a PNG |

**How to tell it worked**: if the grid lines on the source sit on the edges of the color blocks, the grid was found correctly. The status line at the top right shows **경계 일치 %** (edge match). 30% or more is usually fine.

The two panes zoom and pan together. Clicking a cell on one side marks the matching cell on the other.

### 3. Settings, stage by stage

Settings apply from top to bottom. With **값 바꾸면 자동** (Auto on change) checked, the picture is converted again as soon as a value changes. With it unchecked, press **▶ 정리하기** (Clean up) to apply.

**Stage 1. 배경 지우기 (Remove background)**

| Setting | Meaning |
|---|---|
| 방식 (Mode) | 자동 (Auto) removes the most common color on the border. |
| 허용 오차 (Tolerance) | How different a color may be and still count as background. Raise it if background remains, lower it if the picture is eaten away. |
| 테두리와 이어진 배경만 (Only background connected to the border) | When on, the same color inside the picture, such as pupils or outlines, is kept. |

A PNG that is already transparent keeps its own transparency.

**Stage 2. 픽셀 격자 (Pixel grid)**

| Setting | Meaning |
|---|---|
| 칸 크기 기준 (Cell size basis) | **자동** (Auto) finds it from color edges. **수동** (Manual) takes a cell size and offset you type. **결과 크기 고정** (Fixed output size) takes a target size such as 64×64. |
| 찾을 칸 크기 범위 (Cell size range to search) | The range of candidates for how many source pixels one dot covers. Narrow it when the wrong size is picked. |
| 색 경계로 볼 차이 (Edge threshold) | How different two neighboring pixels must be to count as an edge. Raise it for noisy images, lower it to 15–20 for pastel, low-contrast art. |
| 칸 경계를 실제 색 경계에 맞춤 (Snap cell boundaries to real edges) | Moves evenly spaced lines onto nearby real edges. Needed for AI art whose dots vary in size. |
| 최대 이동 (Maximum shift) | The farthest one line may move, as a percentage of the cell size. |
| 찾은 값을 수동으로 (Copy found values to manual) | Copies the automatic result into the manual fields for fine adjustment. |

**Stage 3. 칸 색 고르기 (Cell color)**

| Setting | Meaning |
|---|---|
| 방식 (Method) | The default, **살짝 모은 뒤 가장 많은 색** (most common color after light grouping), picks the color that covers the most of the cell. An average can mix colors and look muddy. |
| 색 묶기 간격 (Grouping step) | How similar two colors must be to count as the same color |
| 칸 테두리 무시 (Ignore cell border) | Skips the blurred pixels at the edge of each cell. Raise it when colors look muddy. |
| 투명 기준 (Transparency threshold) | A cell becomes transparent when the picture covers less of it than this. |

**Stage 4. 팔레트 (Palette)**

| Setting | Meaning |
|---|---|
| 방식 (Method) | **k-평균** (k-means): you set the number of colors. **OKLCH 묶기** (OKLCH grouping): similar colors are grouped and the count follows the image. **원래 색 유지** (Keep original colors): no reduction. |
| 색 수 (Color count) | Number of colors in the result |
| 팔레트 색 (Palette color) | **원본 색** (Original color) uses a color that really occurs in the picture. **평균 색** (Average color) uses the mean of each group. |
| 색 거리, 색상 무게 (Color distance, hue weight) | How strongly a difference in hue counts when choosing the nearest color |
| 명암 계열 우선 (Prefer shade family) | Prefers lighter and darker shades of the same color over a different hue. Keeps shadows from turning into the wrong color. |
| 어두운 색을 외곽선 색 하나로 (Merge dark colors into one outline color) | Collapses near-black colors into one. |

**Stage 5. 정리 (Cleanup)**

| Setting | Meaning |
|---|---|
| 외톨이 픽셀 지우기 (Remove isolated pixels) | Removes single stray dots. High-contrast dots such as pupils are kept. |
| 1px 구멍 메우기 (Fill 1 px holes) | Fills a single transparent cell enclosed on all four sides. |

### 4. Working with colors

Use the **색** (Colors) swatch row at the bottom.

- **See where a color is used**: left-click a swatch or a result pixel. The cells using that color are highlighted for a moment.
- **Remove a color**: right-click a swatch or a result pixel. Only the cells of that color change, each to the nearest remaining color, and everything else stays as it was. Right-click again or press **모두 되살리기** (Restore all) to undo.
- **Pin a color**: press **+** at the end of the row, then click a cell on the source. That cell's color joins the result as a pinned color with a blue border. Use it to bring back a small accent color the palette dropped. Shift+click keeps picking, Esc stops. Right-click a pinned swatch to release it.

### 5. Color decision

With the **선택** (Select) tool, click a result pixel or a source cell. The **색 판정** panel shows:

- The color chosen at each stage, in order: cell color → palette → cleanup → color removal → pinned color → manual override
- The three nearest palette colors and their distances
- The distribution of colors inside the cell
- **직접 지정** (Set directly): click a swatch to set just that cell to a color or to transparent

Use it to find out why a cell got its color and which setting to change.

### 6. Editing by hand

Switch tools at the top to edit the result directly.

| Tool | Action |
|---|---|
| **선택** (Select) | Click a cell to see its color decision, drag to pan |
| **칠하기** (Paint) | Click a swatch to choose the brush color, then click or drag over result cells. |
| **지우개** (Eraser) | Makes result cells transparent. |
| **되돌리기** (Undo) | Undoes one stroke. Ctrl+Z also works. |

While painting or erasing, **right-click** a result cell to pick up its color as the brush color.

Cells you edited by hand stay when you change settings and convert again. They are cleared when the result size changes.

### 7. Controls

| Action | Mouse | Touch |
|---|---|---|
| Zoom | Wheel | Pinch |
| Pan | Drag with the Select tool, or drag with the middle button | Two-finger drag |
| Fit to screen | **화면 맞춤** button, or double-click | **화면 맞춤** button |
| Pick a color | Right-click while painting | Long press |
| Remove a color | Right-click with the Select tool | On swatches only |
| Hide the grid | Uncheck **격자** (Grid) | Same |

### 8. Export

Under **내보내기** (Export), choose a scale and press **PNG 만들기** (Make PNG). ×1 is the result pixel for pixel, and ×8 makes each pixel 8×8. In the preview, press **PNG 저장** (Save PNG) to choose where to save and under what name. The suggested name has the form `source-name_WxH_xScale.png`. **클립보드로 복사** (Copy to clipboard) lets you paste straight into another program.

For a game sprite, export at ×1 and set the texture filter to Point in the engine.

### 9. Troubleshooting

| Symptom | Fix |
|---|---|
| The result is a few big blocks, or far too many tiny pixels | Narrow **찾을 칸 크기 범위** around the real dot size. |
| Grid lines are off the color blocks | Adjust **색 경계로 볼 차이**. Raise it for noisy images, lower it for low contrast. |
| Mostly right, but a few cells are half width | Lower **최대 이동**, or turn off **칸 경계를 실제 색 경계에 맞춤**. |
| Automatic detection keeps failing | Switch **칸 크기 기준** to 수동 or 결과 크기 고정. A photo has no grid, so a fixed output size suits it. |
| Background remains, or part of the picture disappears | Adjust **허용 오차**. |
| Colors look muddy near cell edges | Raise **칸 테두리 무시**. |
| A small accent color disappeared | Pin it with **+**. |
| A shadow turned into a different hue | Turn on **명암 계열 우선**, or raise **색상 무게**. |
| Stray dots remain | Turn on **외톨이 픽셀 지우기** in stage 5. |

## Layout

```
pixel-changer/
├── SKILL.md                     Skill description and instructions
├── assets/pixel-changer.html    The tool
└── references/pipeline.md       How each stage works
```

[`pixel-changer/references/pipeline.md`](pixel-changer/references/pipeline.md) explains how each stage is computed.

## Not included

There are no preset candidates with scores, no automatic parameter search, no 1 px outline fitting, no sprite-sheet frame splitting, no dithering, and no batch conversion. Images longer than 1600 px on one side are scaled down when loaded.

## License

[MIT](LICENSE)
