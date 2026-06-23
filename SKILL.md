---
name: android-ppo-automation
description: Automate Android PPO marketing asset variants in Figma from Apple Version frames. Use when the user asks to copy Apple Version assets into Android Version, generate 512 icons, generate 360x640 phone frames, generate 840x1024 or 900x1280 pad frames, copy backgrounds/titles/mockups, normalize generated title integer geometry/typography, center 360/840/900 titles, normalize generated mockup integer geometry and frame-edge distances, map source mockup screen content into Android mockups, replace Apple/iOS status bars with Android status bars, or repeat the Android ppo variant workflow.
---

# Android PPO Automation

Use this skill for the Figma workflow that starts with copying the `Apple Version` icon into `Android Version` as a new `512x512` copy, then generates Android phone and pad variant frames from the Apple source frames, normalizes generated titles and mockups, maps readable source mockup screen content into Android mockups, and replaces Apple status bars with Android status bars.

This skill depends on Figma write operations, so always load and follow `figma-use` before every `use_figma` call.

## Source Structure

The expected Figma structure is:

- Page/node: `Android ppo`
- Source section: `Apple Version`
- Destination section: `Android Version`
- Device template section: `Android phone and pad`

Do not assume node IDs are stable across files. Prefer finding sections and templates by name unless the user explicitly works in the same file and IDs have already been verified.

## Required Workflow

Read `references/workflow.md` before acting. It contains the ordered procedure, layer-selection heuristics, naming conventions, scaling rules, and validation checks.

At a high level:

1. Restore or preserve the original `Apple Version` assets.
2. Copy `Icon` into `Android Version` as a separate `512x512` icon.
3. Generate phone frames from `430x932` sources: `360_01..360_08`.
4. Copy phone backgrounds, titles, selected mockups, second-page foreground content, readable mockup screen pages, and Android status bars.
5. Generate pad frames from `1024x1366` sources: `840_01..840_08` and `900_01..900_08`.
6. Copy pad backgrounds, titles, `Android pad Black` mockups, second-page foreground content, readable mockup screen pages, and Android status bars.
7. Normalize generated title coordinates, dimensions, and typography so Figma does not show decimal values; center `360_*`, `840_*`, and `900_*` titles horizontally after typography changes.
8. Normalize generated mockup root geometry and frame-relative left, right, and bottom distances so Figma does not show decimal values.
9. Verify counts, dimensions, child counts, text presence, status-bar replacement, screen-content wrappers, integer title values, centered generated titles, and integer mockup geometry/distances.

## Figma Rules

- Never move the original `Apple Version` source frames or icon.
- Make new copies in `Android Version`.
- Make operations idempotent by tagging generated nodes with `setSharedPluginData("codex", "source", "...")` and removing only matching generated nodes on rerun.
- Use local coordinates after appending into a section or frame. Do not add section absolute coordinates to child `x/y`.
- Use separate x/y ratios when mapping positions between different aspect ratios.
- Prefer group scaling over editing text properties when copied text uses missing local fonts.
- After copying titles, remove decimal values from generated title geometry and typography. Check every styled text segment with `getStyledTextSegments()`, not just whole-node `fontSize`, because mixed-font titles can hide decimal ranges.
- After title typography changes, center the title group inside each generated `360_*`, `840_*`, and `900_*` frame by the combined visual title bounds. Keep local `x/y` integers and avoid `.5` centers by widening the rightmost title text box by 1px when combined title bounds have the wrong parity.
- After copying or scaling mockups, round generated mockup root `x/y/width/height` to integers and verify frame-relative left, right, and bottom distances are integers. Negative bottom distances are allowed when the mockup deliberately extends beyond the frame, but they must still be whole numbers.
- Put mockups below title layers so titles remain visible.
- If a source page contains an irregular, angled, distorted, or unusually cropped mockup, stop before that mockup step and ask the user whether to copy it or skip it.
- If a source page has no mockup and appears to be a pure text page or image-only page, stop before inventing a device treatment and ask the user how to handle that page.
- If the device template section contains both black and white mockup variants, ask the user which color to use before copying mockups, unless the user has already specified the color in the current request.
- Use `Android Status Bars Black` or `Android Status Bars White` from `Android phone and pad` when replacing Apple status bars. If both are present and the user has not specified a color, ask which status-bar color to use.
- For rotated pad mockup `Paste Here` containers, align screen content and Android status bars using absolute bounds relative to the target frame instead of appending them directly into the rotated container.
- Always return `createdNodeIds` and `mutatedNodeIds` from write scripts.
