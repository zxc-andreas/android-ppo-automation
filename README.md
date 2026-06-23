# Android PPO Automation

Codex skill for automating Android PPO marketing asset variants in Figma.

This skill starts from Apple Version assets and generates Android Version outputs, including a 512x512 icon copy, phone screenshot frames, pad screenshot frames, copied backgrounds, scaled titles, selected Android mockups, mapped in-mockup screen content, and Android status bars.

## What It Does

- Copies the Apple Version `Icon` into Android Version as a new `512x512` asset.
- Generates `360x640` phone frames from `430x932` source frames.
- Generates `840x1024` and `900x1280` pad frames from `1024x1366` source frames.
- Copies matching backgrounds into each generated frame.
- Copies titles at 70% scale while preserving source placement.
- Copies Android phone and pad mockups from the `Android phone and pad` section.
- Maps readable app screen content from Apple phone and iPad mockups into Android phone and pad mockups.
- Replaces copied Apple/iOS status bars with Android status bars from the `Android phone and pad` section.
- Keeps generated pad mockup shells, screen content, and Android status bars together as movable mockup suites.
- Normalizes generated title geometry/typography and centers titles in `360`, `840`, and `900` outputs.
- Normalizes generated mockup geometry and left/right/bottom distances to whole-pixel values.
- Asks before handling irregular mockups, no-mockup text pages, image-only pages, or color choices when multiple mockup colors exist.

## Expected Figma Structure

The Figma file should contain an `Android ppo` page with these sections:

- `Apple Version`
- `Android Version`
- `Android phone and pad`

Device templates are expected in `Android phone and pad`, such as:

- `Phone Black`
- `Phone White`
- `Android pad Black`
- `Android Status Bars Black`
- `Android Status Bars White`

## Installation

Clone or copy this repository into your Codex skills directory:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/zxc-andreas/android-ppo-automation.git ~/.codex/skills/android-ppo-automation
```

Restart Codex after installing so the skill can be discovered.

## Usage

Ask Codex to use the skill on a Figma file, for example:

```text
Use $android-ppo-automation to generate Android Version assets from Apple Version in this Figma file.
```

For the workflow used in this repository, the default prompt is:

```text
Use $android-ppo-automation to copy Apple Version assets into Android Version.
```

## Notes

- The original Apple Version frames and icon must remain unchanged.
- Generated nodes should be new copies under Android Version.
- Generated title and mockup positions should be whole-pixel values after the workflow finishes.
- The skill depends on Figma write operations, so Codex must follow `figma-use` before each `use_figma` call.
- The detailed workflow is in [`references/workflow.md`](references/workflow.md).
