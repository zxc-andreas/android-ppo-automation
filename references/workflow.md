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
- `phone-screen-content-from-6.7`
- `android-status-bar-360`
- `foreground-from-430`
- `pad-empty-840`
- `pad-empty-900`
- `pad-background-from-1024`
- `pad-title-from-1024`
- `pad-black-mockup`
- `android-status-bar-pad`
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
After all phone and pad title-copy steps are complete, run section 16 to remove decimal geometry/typography and center generated titles.

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
After copying or resizing phone mockups, run section 17 to normalize mockup root geometry and frame-edge distances.

## 6. Map Phone Mockup Screen Content

After regular `Phone Black` mockups exist in the `360_*` targets, copy readable screen-page content from each source `6.7_*` mockup into the matching target mockup.

Source detection:

- Find a nested frame named `2-SCREEN` inside the source `6.7_*` mockup.
- Copy the visible child page frame inside `2-SCREEN` such as `today`, `explore - Glp recipes`, `Recipes Detail`, or `chat`.
- If no `2-SCREEN` exists, skip that page and report it. Do not invent content.
- Known skips in the original workflow: `6.7_01` and `6.7_02`.

Target detection:

- In each `360_*`, find the `Phone Black` mockup and its nested `Paste Here` frame.
- Place generated content inside `Paste Here`, below the camera/island overlay when present.
- Set `Paste Here.clipsContent = true`.

Position and scale:

- Create a wrapper named `screen/6.7_XX -> 360_XX`.
- Use shared plugin data tag `phone-screen-content-from-6.7`.
- Scale the copied content proportionally to fit the `Paste Here` screen area.
- A working approach is `scale = max(PasteHere.width / sourceScreen.width, PasteHere.height / sourceScreen.height)`, with `wrapper.x` centered and `wrapper.y = 0`.
- Keep any generated Android status bar above this content layer.

## 7. Replace Phone Status Bars

Replace Apple/iOS status bars inside 360 phone mockups with Android status bars from `Android phone and pad`.

Template:

- Prefer `Android Status Bars Black` for black phone mockups.
- Use `Android Status Bars White` only if the user selects a white/light status-bar treatment.
- If both black and white status bar templates exist and the user has not specified one, ask which to use.

Target:

- For each `360_*`, find the nested `Paste Here` frame.
- Clone the selected Android status-bar template into `Paste Here`.
- Name it `Android Status Bar / 360_XX`.
- Tag with `android-status-bar-360`.
- Scale to `PasteHere.width / template.width`, then set `x = 0`, `y = 0`.
- Insert it above generated screen content and below only hardware overlays such as camera/island if needed.

Cleanup:

- Remove only prior generated nodes tagged `android-status-bar-360` before rerunning.
- Hide visible Apple/iOS nodes named `Status Bar` inside `Paste Here` descendants by setting `visible = false`.
- Do not delete source status bars.

Known behavior:

- `360_01` may have no `Paste Here` because its mockup is irregular; skip and report.
- `360_02` may have a `Paste Here` but no copied screen content; still place the Android status bar if the target exists.

## 8. Copy Phone Second-Page Foreground

For `360_02`, also copy non-mockup foreground content from `6.7_02`, such as:

- Drug name floating group
- Rating group

Do not edit individual text font sizes if fonts are unavailable. Copy the groups and uniformly rescale them smaller. A working scale used previously was `65%`.

Tag as `foreground-from-430`.

## 9. Generate Pad Frames

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

## 10. Copy Pad Backgrounds

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

## 11. Copy Pad Titles

Copy title layers from each `ipad_*` into both target sets.

Typical title layers:

- Most pages: `Group 2085660799`
- `ipad_01`: copy both `Group 2085660799` and `All-In-One`

Position and scale:

- Use separate target x/y ratios.
- Use `titleScale = min(xRatio, yRatio)` unless the user asks for a fixed percentage.
- Tag as `pad-title-from-1024`.

After all phone and pad title-copy steps are complete, run section 16 to remove decimal geometry/typography and center generated titles.

## 12. Copy Pad Mockups

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
- The generated Android status bar for this target must later be parented inside this `pad-black-mockup` frame, not left as a sibling under the target `840_*` or `900_*` frame.

Known final working sizes in this workflow:

- `840_*`: pad mockup width `643`
- `900_*`: pad mockup width `715`

After copying or resizing pad mockups, run section 17 to normalize mockup root geometry and frame-edge distances.

## 13. Do Not Map Pad Mockup Screen Content

Do not copy source iPad UI pages into generated `840_*` or `900_*` pad mockups.

Rules:

- Do not create new `screen/ipad_XX -> 840_XX` or `screen/ipad_XX -> 900_XX` wrappers.
- Do not tag new nodes as `pad-screen-content-from-1024`.
- Do not copy `Frame 2147228900 > image` content from `ipad_*` sources into pad mockup `Paste Here` areas.
- Do not delete or modify existing copied pad UI pages if the Figma file already has them from a previous run. Leave already-copied content unchanged unless the user explicitly asks to remove, regenerate, or update it.
- Continue with pad Android status-bar replacement in section 14.

## 14. Replace Pad Status Bars

Replace Apple/iOS status bars for `840_*` and `900_*` pad mockups with Android status bars from `Android phone and pad`.

Template:

- Prefer `Android Status Bars Black` for `Android pad Black`.
- Use `Android Status Bars White` only if the user selects a white/light status-bar treatment.
- If both black and white status bar templates exist and the user has not specified one, ask which to use.

Target and placement:

- For each `840_*` and `900_*`, find the pad mockup `Paste Here`.
- Because pad `Paste Here` can be rotated by `90` degrees, clone the status bar as an unrotated overlay inside the matching `pad-black-mockup` frame.
- Name it `Android Status Bar / 840_XX` or `Android Status Bar / 900_XX`.
- Tag with `android-status-bar-pad`.
- Scale by the visual screen width: `scale = PasteHere.absoluteBoundingBox.width / template.width`.
- Set local position with absolute bounds:
  - `clone.x = PasteHere.absoluteBoundingBox.x - targetFrame.absoluteBoundingBox.x`
  - `clone.y = PasteHere.absoluteBoundingBox.y - targetFrame.absoluteBoundingBox.y`
- Keep the Android status bar above the pad shell/screen area.
- Do not leave the status bar outside the mockup frame. Moving the `pad-black-mockup` frame must move the shell and Android status bar together.

Cleanup:

- Remove only prior generated nodes tagged `android-status-bar-pad` before rerunning.
- Hide visible copied Apple/iOS nodes named `Status Bar` inside generated content wrappers.
- Do not delete source Apple status bars.

Known working sizes:

- `840_*`: Android status bar about `577.98 x 49.32`
- `900_*`: Android status bar about `642.70 x 54.84`

## 15. Copy Pad Second-Page Foreground

For `840_02` and `900_02`, copy non-mockup foreground content from `ipad_02`:

- Drug name floating group
- Rating group

Use group scaling instead of editing fonts.

Known working scales:

- `840_02`: `68%`
- `900_02`: `78%`

Tag as `pad-foreground-from-1024`.

## 16. Normalize Generated Titles

Normalize generated title nodes after title copying and after any later operation that changes typography, size, or position.

Targets:

- Phone titles tagged `title-from-430` in `360_*` frames.
- Pad titles tagged `pad-title-from-1024` in `840_*` and `900_*` frames.

Integer rules:

- Round title node `x`, `y`, `width`, and `height` to whole pixels.
- Remove decimal typography values from every text segment:
  - `fontSize`
  - `lineHeight.value`
  - `letterSpacing.value`
- Use `getStyledTextSegments(["fontSize", "fontName", "lineHeight", "letterSpacing"])` on text descendants. Do not trust only whole-node `fontSize`, because mixed-font text can keep decimal values in hidden ranges.
- When changing text geometry, use `resizeWithoutConstraints()` or `resize()` as appropriate, then set integer `x/y`.

Centering rules:

- Center the combined visual bounds of all generated title nodes inside each target frame.
- Target horizontal centers:
  - `360_*`: `180`
  - `840_*`: `420`
  - `900_*`: `450`
- Keep all final title `x/y/width/height` values as integers.
- If combined title bounds cannot center without producing `.5`, widen the rightmost title text box by `1px`, then recenter.
- Preserve the original relative vertical placement unless the user explicitly asks for a different layout.

Validation:

- Confirm every generated target frame has centered titles with no decimal `x/y/width/height`.
- Confirm no text segment in generated titles has decimal `fontSize`, `lineHeight.value`, or `letterSpacing.value`.

## 17. Normalize Generated Mockups

Normalize generated mockups after copying, resizing, or changing mockups/status bars.

Targets:

- Phone mockups tagged `phone-black-mockup` in `360_*` frames.
- Pad mockups tagged `pad-black-mockup` in `840_*` and `900_*` frames.
- `360_01` may intentionally have no regular generated mockup when an irregular source mockup was skipped.

Integer rules:

- Only edit generated Android Version mockups. Do not edit original Apple Version mockups or source frames.
- Round each mockup root node `x`, `y`, `width`, and `height` to whole pixels.
- Use `resizeWithoutConstraints()` or `resize()` for width/height changes, then set integer `x/y`.
- For `360_*` phone mockups, preserving the mapped visual center while rounding is acceptable.
- For `840_*` and `900_*` pad mockups, integer-fix the `pad-black-mockup` frame as the movable unit. The generated Android status bar should already be inside that frame, so it moves with the shell.
- After rounding, verify frame-relative distances:
  - `left = mockup.x`
  - `right = frame.width - (mockup.x + mockup.width)`
  - `bottom = frame.height - (mockup.y + mockup.height)`
- `left`, `right`, and `bottom` must all be integers for `360_*`, `840_*`, and `900_*`.
- Negative `bottom` is allowed when the mockup deliberately extends below the frame, but it must still be a whole number.
- Preserve visual alignment with the source-page mapping. For pad targets, if a mockup needs to be moved after creation, move the `pad-black-mockup` frame so the shell and status bar remain locked together.

Validation:

- Count generated phone and pad mockups by tag.
- Report skipped pages explicitly, especially irregular/no-mockup pages that the user chose to skip.
- Confirm every generated mockup root has integer `x/y/width/height`.
- Confirm every generated mockup has integer left, right, and bottom distances relative to its parent frame.
- Confirm every generated pad Android status bar is parented inside the matching `pad-black-mockup` frame.
- Confirm no new `pad-screen-content-from-1024` nodes or `screen/ipad_XX -> 840_XX` / `screen/ipad_XX -> 900_XX` wrappers were created during the current run.

## 18. Validation Checklist

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
  - Readable `6.7_*` screen content is copied into matching 360 mockup `Paste Here` frames where available
  - Android status bars are present in 360 mockups with `Paste Here`, and copied Apple `Status Bar` nodes are hidden
- Pad targets:
  - 8 frames `840_01..840_08`
  - 8 frames `900_01..900_08`
  - Each has backgrounds, titles, and `Android pad Black`
  - `840_02` and `900_02` have foreground content
  - No new source iPad UI pages were copied into `840_*` or `900_*` pad mockups
  - Android pad status bars are present where pad mockups exist
- Titles in `360_*`, `840_*`, and `900_*` are horizontally centered and have no decimal geometry or typography values.
- Mockup root `x/y/width/height` values are integers.
- Mockup left, right, and bottom distances to parent frames are integers.
- Generated layers are below titles when they might overlap.
- Generated phone screen content wrappers are clipped and do not cover Android status bars.
- Any no-mockup pure text/image pages were handled according to the user's explicit choice.

Return a concise user summary with counts and any deliberate skips.
