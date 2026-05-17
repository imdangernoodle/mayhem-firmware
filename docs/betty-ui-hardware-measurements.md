# Betty UI Hardware Measurements

Date: 2026-05-17
Device: HackRF One / PortaPack, 240x320 display
Firmware branch: `betty-ui-pass`
Firmware commit tested: `fe19bd8`

## Setup

- Flashed `build-betty-gcc92/firmware/portapack-mayhem-firmware.bin` with `hackrf_spiflash`.
- Reset out of HackRF mode with `hackrf_spiflash -R`.
- Confirmed PortaPack USB shell at `/dev/cu.usbmodemTransceiver1`.
- Confirmed device-side SD access:
  - `filesize /BETTY/menu_bg.bmp` returned `218934`.
  - `ls /BETTY` listed `menu_bg.bmp`, `menu_peek.bmp`, and Bitty assets.

## Method

The USB shell `button` command calls `control::debug::inject_switch()` and then waits for two frame syncs before replying. That makes host-side command time a practical end-to-end UI repaint/navigation measurement, not a pure CPU timer.

Measurements were taken with Python/pyserial against `/dev/cu.usbmodemTransceiver1`, using repeated `button 8` and `button 7` encoder events while the SD-backed Betty menu background was available.

## Results

SD-backed menu navigation:

| Command | Samples | Min | Median | Mean | P95 | Max |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `button 8` | 30 | 50.5 ms | 53.7 ms | 55.3 ms | 77.8 ms | 105.2 ms |
| `button 7` | 30 | 50.3 ms | 55.1 ms | 55.9 ms | 78.3 ms | 106.3 ms |

Earlier fallback-path navigation, when the device could not see the SD card:

| Command | Samples | Min | Median | Mean | Max |
| --- | ---: | ---: | ---: | ---: | ---: |
| `button 8` | 20 | 50.2 ms | 53.3 ms | 53.4 ms | 55.9 ms |
| `button 7` | 20 | 50.9 ms | 52.8 ms | 52.9 ms | 55.2 ms |

Memory and stack high-water readings from `sysinfo`:

| State | M0 Heap | M0 Stack | Notes |
| --- | ---: | ---: | --- |
| Home, SD unavailable | 31008 | 387 | before SD-backed art was visible |
| Home, SD-backed art visible | 28760 | 315 | after `/BETTY/menu_bg.bmp` was readable |
| After repeated SD-backed navigation | 15272 | 281 | lowest observed M0 stack reading |

`M0 stack` is reported in 32-bit words by `get_free_stack_space()`, so `281` means about `1124` bytes free.

## Interpretation

- The optimized SD-backed menu path is close to the fallback path in typical navigation. Median cost is only about 1-3 ms higher.
- Occasional SD-backed outliers around 105 ms exist. They are visible in the host-side frame-sync timing and are the only remaining performance reason to consider deeper SD read batching or caching.
- The lowest observed M0 stack headroom was about 1.1 KB. That is acceptable for the current patch, but it supports the earlier decision not to add a larger stack-heavy bitmap cache without a separate design.
- The shell `fread` command is useful for confirming file access, but not for raw SD throughput, because it emits data as text over USB serial.

## Decision

No further firmware optimization is justified before the next design pass. The current hardened renderer is stable enough to keep, and the remaining performance work should wait for a specific symptom:

- visible lag during normal hand navigation,
- repeated SD-backed navigation max times above roughly 150 ms,
- or stack headroom dropping below 128 words.
