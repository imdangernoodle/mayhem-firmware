# Betty UI Stability and Performance Audit

Date: 2026-05-17
Branch: `betty-ui-pass`

## Source Quality

Highest-confidence evidence is the local firmware source and a successful local build. External references were used only for constraints that are not obvious from the local diff:

- Mayhem upstream repository: https://github.com/portapack-mayhem/mayhem-firmware
- Mayhem custom splash guidance: https://github.com/portapack-mayhem/mayhem-firmware/wiki/Create-a-custom-splash-screen
- FatFs application note, effective file access: https://elm-chan.org/fsw/ff/doc/appnote.html
- FatFs `f_read`: https://elm-chan.org/fsw/ff/doc/read.html
- FatFs `f_lseek`: https://elm-chan.org/fsw/ff/doc/lseek.html
- HackRF One documentation: https://hackrf.readthedocs.io/en/latest/hackrf_one.html

The Mayhem wiki is project-maintained but still a wiki, so it is good for SD-card/splash conventions rather than low-level correctness. FatFs documentation is primary for filesystem behavior. HackRF documentation is primary for device capability and safety, but it does not validate UI performance by itself.

## Decisions Implemented

1. Keep large character art on the SD card rather than baking it into firmware.
   - Reason: local build headroom is tight, while Mayhem already documents SD-card bitmap personalization. The splash guidance uses SD-resident BMP assets, including exact width requirements for PortaPack splash images.
   - Post-check: firmware still builds and the SD-backed menu background remains optional; missing/corrupt art falls back to normal fills.

2. Cache unavailable menu background state.
   - Reason: repeated failed SD opens during paint are wasted work. FatFs documents file reads and seeks as real filesystem operations, and its performance note favors fewer/larger transfers over tiny repeated work.
   - Post-check: `MenuBackgroundState::Unavailable` short-circuits both full background and region restore paths until the grid reloads.

3. Skip redundant per-tile region restores immediately after a full background paint.
   - Reason: the painter forces children dirty after a parent repaint. Without a hook, a full background draw was followed by child-level SD reads for every unselected tile.
   - Post-check: `BtnGridView::paint()` marks the background fresh only after successful full draw; `Painter::paint_widget()` calls `on_children_painted()` after child traversal so the skip is limited to that pass.

4. Reject undersized BMP coverage before drawing.
   - Reason: partial background success can leave stale pixels on taller layouts. Mayhem's splash guidance permits short splash heights in one context, but a menu background must cover the requested menu rect exactly.
   - Post-check: full draw now requires exact `width == parent_rect().width()` and `height == parent_rect().height()`. Region restores reject partially covered visible target rectangles.

5. Preserve blacklist compatibility after clearer labels.
   - Reason: the redesign changes user-visible `text` labels, while many existing settings likely refer to old app names. The subtitle usually retains the legacy app name.
   - Post-check: blacklist matching checks both `text` and `subtitle`.

6. Do not implement deeper bitmap caching yet.
   - Reason: caching decoded strips or a persistent BMP reader could reduce SD traffic further, but it would add lifetime, RAM, and SD-removal edge cases. The current patch is lower-risk and build-verified.
   - Needed proof before revisiting: hardware timing while navigating menus, plus memory/stack measurements under H4M.

## Verification

- `git diff --check`
- `cmake --build build-betty-gcc92 --target firmware ppfw -- -k 0`
- `tar -tf build-betty-gcc92/firmware/portapack-mayhem_OCI.ppfw.tar | rg '^APPS/.*\.ppm[ap]$' | wc -l`
- `hackrf_info`

Results:

- Build completed successfully.
- Firmware package contains 87 external app binaries.
- `hackrf_info` was installed and working, but reported `No HackRF boards found.`
- No SD card firmware mount was visible under `/Volumes`.

## Peer Audit

Curie found no P0/P1 blockers after the undersized BMP fix. Remaining followups:

- Decide whether `sdcard/SPLASH/betty-witches-preview.png` belongs in git. It is currently left untracked as a preview artifact.
- Consider moving `MenuBackgroundState` from file-global to per-grid if future UI layouts ever show more than one `BtnGridView` at once.
- Measure real hardware navigation latency before taking on more invasive SD-read batching or persistent bitmap-reader work.
