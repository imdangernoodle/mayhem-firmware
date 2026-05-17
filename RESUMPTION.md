# Betty Mayhem Firmware Resumption Note

Date: 2026-05-17
Branch: `betty-ui-pass`
Standalone project target: `/Users/admin/Projects/mayhem-ui-betty-nightly`

## Current State

- The active work started in `/Users/admin/Projects/splats/mayhem-ui-betty-nightly`.
- The branch is pushed to `fork/betty-ui-pass` on `https://github.com/imdangernoodle/mayhem-firmware.git`.
- Latest pushed code commit before this note: `0e470a1 Record Betty UI hardware measurements`.
- The hardened Betty firmware was flashed successfully to the connected HackRF/PortaPack.
- The SD card contains the hardened firmware copies:
  - `/FIRMWARE/betty-rf-grimoire-260517-hardened.bin`
  - `/FIRMWARE/betty-rf-grimoire-260517-hardened.ppfw.tar`
- Device-side SD access was confirmed through the PortaPack shell:
  - `/BETTY/menu_bg.bmp` returned size `218934`.
  - `/BETTY` listed the Betty and Bitty assets.

## Important Files

- `docs/betty-ui-audit.md` records the source-backed render/performance decisions.
- `docs/betty-ui-hardware-measurements.md` records real device timing and stack/heap readings.
- `docs/betty-ui-preview.html` is the current Safari design preview.
- `docs/preview-assets/` contains PNG preview copies converted from the firmware SD BMP assets.
- `firmware/application/ui/ui_btngrid.cpp` contains the SD-backed menu background renderer and hardened tile restore logic.
- `firmware/application/ui_navigation.cpp` contains the current purpose-first app labels and subtitles.

## Hardware Measurements

SD-backed menu navigation, measured through the firmware shell:

- `button 8`: median `53.7 ms`, max `105.2 ms`.
- `button 7`: median `55.1 ms`, max `106.3 ms`.
- Lowest observed M0 stack: `281` words, about `1124` bytes free.

Conclusion: no deeper performance work is justified before the next visual design pass unless hand navigation shows obvious lag, repeated max times exceed roughly `150 ms`, or M0 stack drops below `128` words.

## Design Direction

The next phase is design fidelity, not optimization:

- Keep Mayhem's compact navigation model.
- Keep large character art SD-backed, not baked into flash.
- Make the UI feel more like the Safari preview: witch-library atmosphere, warmer selected states, clearer tile hierarchy.
- Keep action-first labels with real app names as subtitles.
- Use Echidna as quiet menu atmosphere.
- Use Betty peeking sparingly.
- Use Bitty only when large enough to read visually, especially receive/transmit activity states.

## Known Caution

- `button 6` in the firmware shell toggles the debug/performance overlay. Do not use it as a generic Back key.
- If the overlay appears, a clean `reboot` shell command returns to the `Betty RF` home screen.
- `screenframe` can capture the current device framebuffer directly; no webcam is needed for most UI inspection.
- `docs/current-portapack-screen.png` and `.ppm` were scratch framebuffer captures and should not be committed unless intentionally needed.
- `.toolchains/` and `build*` directories are generated/heavy and should stay out of the standalone project copy unless rebuilding locally from that copy.

## Next Useful Step

Open `docs/betty-ui-preview.html` in Safari, compare it to the current device UI, then patch the firmware in a small design pass:

1. Main menu tile selected-state colors.
2. Category title bar treatment.
3. Receive menu labels and Bitty receive-state placement.
4. Flash, capture framebuffer, and compare.
