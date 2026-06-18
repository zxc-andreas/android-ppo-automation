# Android PPO Figma Variant Workflow

## Inputs

Start from a Figma design file containing:

- `Android ppo` page
- `Apple Version` section with source frames
- `Android Version` section as the output target
- `Android phone and pad` section containing device templates:
  - `Phone Black`
  - `Android pad Black`

Find nodes by name first:

```js
const page = figma.root.children.find(p => p.name === "Android ppo");
await figma.setCurrentPageAsync(page);
const appleSection = page.children.find(n => n.name === "Apple Version");
const androidSection = page.children.find(n => n.name === "Android Version");
const deviceSection = page.children.find(n => n.name === "Android phone and pad");
```

If the user provides a node URL, use that exact page/node as the starting context and verify names.

## Idempotency Tags

Use shared plugin data on generated nodes:

- `icon-512-copy`
- `empty-360-from-430`
- `background-from-430`
- `title-from-430`
- `phone-black-mockup`
- `foreground-from-430`
- `pad-empty-840`
- `pad-empty-900`
- `pad-background-from-1024`
- `pad-title-from-1024`
- `pad-black-mockup`
- `pad-foreground-from-1024`

Before rerunning a step, remove only generated nodes with the matching tag. Do not remove user-created nodes or original source nodes.

## 1. Copy Icon To Android Version

Goal: create a new `512x512` icon copy in `Android Version` while leaving the original `Apple Version/Icon` unchanged.

1. Find `Icon` under `Apple Version`.
2. Ensure the original remains parented to `Apple Version`, at its original local position and size when known.
3. Remove prior generated `icon-512-copy` nodes in `Android Version`.
4. Clone the original icon.
5. Append the clone to `Android Version`.
6. Set local position to match the source icon local position.
7. Rescale to `512 / original.width`.
8. Tag with `icon-512-copy`.

Important: section child coordinates are local to the section. Do not add section absolute coordinates.

## 2. Generate Phone Frames

Source frames: `Apple Version` direct children with rounded size `430x932`, usually `6.7_01..6.7_08`.

Target frames:

- Names: `360_01..360_08`
- Size: `360x640`
- Parent: `Android Version`
- Position: same local `x/y` as the corresponding `430x932` source unless the user asks otherwise
- Empty at creation
- White fill, `clipsContent = true`
- Tag: `empty-360-from-430`

Mapping: `6.7_01 -> 360_01`, `6.7_02 -> 360_02`, etc.

## 3. Copy Phone Backgrounds

For each source `6.7_*`, copy background-like direct children into the matching `360_*`.

Use these heuristics:

- Include visible, non-text layers that are large decorative/background elements.
- Include image fills and large masks/mockup/background shapes when they are background context.
- Exclude layers with visible text.
- Exclude small buttons/icons unless they are clearly part of the background.
- Set target frame fills/strokes from source frame fills/strokes.

Position and scale:

- `xRatio = 360 / 430`
- `yRatio = 640 / 932`
- For background objects, set `clone.x = source.x * xRatio`, `clone.y = source.y * yRatio`.
- Use a uniform scale based on width ratio unless visual requirements demand otherwise.
- Tag copied layers as `background-from-430`.

## 4. Copy Phone Titles

Copy only title layers from each `6.7_*` into the matching `360_*`.

Typical title layers:

- Most pages: `Group 2085660799`
- `6.7_01`: copy both `Group 2085660799` and `All-In-One`

Position and scale:

- `xRatio = 360 / 430`
- `yRatio = 640 / 932`
- `clone.x = source.x * xRatio`
- `clone.y = source.y * yRatio`
- Rescale the title object to `70%`.
- Tag as `title-from-430`.

Avoid editing font sizes directly when local fonts may be unavailable. Group scaling is safer.

## 5. Copy Phone Mockups

Template: use the phone mockup color chosen by the user from `Android phone and pad`, such as `Phone Black` or `Phone White`.

Before copying phone mockups:

- Inspect available phone templates in `Android phone and pad`.
- If both black and white variants exist and the user has not already specified one in the current request, ask which color to use.
- Suggested question: "I found both Phone Black and Phone White. Which color should I use for the phone mockups?"
- If the user specifies a color in the request, use that template without asking again.

Rules:

- If a source page uses an irregular, angled, distorted, perspective, or unusually cropped mockup, do not decide silently. Pause before copying that page's mockup and ask the user whether to copy it or skip it.
- If the user says to skip, leave that target without a regular phone mockup and mention the deliberate skip in the final summary.
- If the user says to copy, map it using absolute bounds and the same center-mapping rules as regular mockups.
- If a source page has no detectable phone mockup and appears to be pure text, image-only, or a non-device composition, do not silently skip or invent a mockup. Ask the user how to handle it before continuing.
- Suggested question: "This page does not appear to have a normal phone mockup. Should I keep it as a no-mockup page and copy its main text/image content, skip extra content for this page, or place the standard Phone Black anyway?"
- For other pages, find the corresponding phone mockup in the source (`iphone16pro`, `Black`, or nested phone frame).
- Use absolute bounds to compute source mockup bounds relative to the source frame when the mockup is nested.
- Map source mockup center to target frame using separate x/y ratios.
- Clone the selected phone template, insert below title layers, and align its center to the mapped source center.
- Use 10% scale steps and keep the mockup width an integer.
- If the mockup looks too large, shrink one 10% step, preserving center.
- Tag as `phone-black-mockup`.

Known final working phone width used in this workflow: `270`.

## 6. Copy Phone Second-Page Foreground

For `360_02`, also copy non-mockup foreground content from `6.7_02`, such as:

- Drug name floating group
- Rating group

Do not edit individual text font sizes if fonts are unavailable. Copy the groups and uniformly rescale them smaller. A working scale used previously was `65%`.

Tag as `foreground-from-430`.

## 7. Generate Pad Frames

Source frames: `Apple Version` direct children with rounded size `1024x1366`, usually `ipad_01..ipad_08`.

Generate two target sets in `Android Version`:

- `840_01..840_08`: `840x1024`
- `900_01..900_08`: `900x1280`

This doubles the original iPad source count.

Create empty frames first:

- White fill
- `clipsContent = true`
- Tags: `pad-empty-840` and `pad-empty-900`

Place them in a clear area. In the original workflow, `840_*` were placed around local y `3293` and `900_*` around local y `4359`, but adapt if those positions are occupied.

## 8. Copy Pad Backgrounds

For each `ipad_*`, copy background-like direct children into both matching pad targets.

Use the same background heuristics as phone:

- Include visible non-text background, image, mask, vector, gradient, and device/decorative layers.
- Exclude layers with visible text.
- Copy source frame fills/strokes to target.
- Tag as `pad-background-from-1024`.

Ratios:

- For `840_*`: `xRatio = 840 / 1024`, `yRatio = 1024 / 1366`
- For `900_*`: `xRatio = 900 / 1024`, `yRatio = 1280 / 1366`

Set `x/y` with separate ratios and use a uniform width-based scale for copied layers unless the user asks for a different fit.

## 9. Copy Pad Titles

Copy title layers from each `ipad_*` into both target sets.

Typical title layers:

- Most pages: `Group 2085660799`
- `ipad_01`: copy both `Group 2085660799` and `All-In-One`

Position and scale:

- Use separate target x/y ratios.
- Use `titleScale = min(xRatio, yRatio)` unless the user asks for a fixed percentage.
- Tag as `pad-title-from-1024`.

## 10. Copy Pad Mockups

Template: use the pad mockup color chosen by the user from `Android phone and pad`, such as `Android pad Black` or `Android pad White`.

Before copying pad mockups:

- Inspect available pad templates in `Android phone and pad`.
- If both black and white variants exist and the user has not already specified one in the current request, ask which color to use.
- Suggested question: "I found both Android pad Black and Android pad White. Which color should I use for the pad mockups?"
- If the user specifies a color in the request, use that template without asking again.

Rules:

- If a source page uses an irregular, angled, distorted, perspective, or unusually cropped pad mockup, do not decide silently. Pause before copying that page's mockup and ask the user whether to copy it or skip it.
- If a source page has no detectable pad mockup and appears to be pure text, image-only, or a non-device composition, do not silently skip or invent a mockup. Ask the user how to handle it before continuing.
- Suggested question: "This pad page does not appear to have a normal pad mockup. Should I keep it as a no-mockup page and copy its main text/image content, skip extra content for this page, or place Android pad Black anyway?"
- For each `ipad_*`, find the source device/content mockup:
  - Some pages use `iPad Pro 2018`.
  - Most content pages use `Frame 2147228900`.
- Use absolute bounds relative to source frame.
- Map source center to target frame with separate x/y ratios.
- Clone the selected pad template, insert below titles, and center on the mapped point.
- If it fits, keep original size.
- If it does not fit, shrink by 10% steps and keep width integer.
- Tag as `pad-black-mockup`.

Known final working sizes in this workflow:

- `840_*`: pad mockup width `643`
- `900_*`: pad mockup width `715`

## 11. Copy Pad Second-Page Foreground

For `840_02` and `900_02`, copy non-mockup foreground content from `ipad_02`:

- Drug name floating group
- Rating group

Use group scaling instead of editing fonts.

Known working scales:

- `840_02`: `68%`
- `900_02`: `78%`

Tag as `pad-foreground-from-1024`.

## 12. Validation Checklist

After each run, verify with a read-only `use_figma` call:

- The original `Apple Version` source nodes still exist and were not moved or resized.
- The 512 icon copy exists in `Android Version`.
- Mockup color choices were either specified by the user or explicitly confirmed before copying.
- Phone targets:
  - 8 frames `360_01..360_08`
  - Each is `360x640`
  - Backgrounds and titles exist
  - Any irregular mockup source was handled according to the user's explicit copy/skip choice
  - `360_02` has foreground content
- Pad targets:
  - 8 frames `840_01..840_08`
  - 8 frames `900_01..900_08`
  - Each has backgrounds, titles, and `Android pad Black`
  - `840_02` and `900_02` have foreground content
- Mockup widths are integers.
- Generated layers are below titles when they might overlap.
- Any no-mockup pure text/image pages were handled according to the user's explicit choice.

Return a concise user summary with counts and any deliberate skips.
