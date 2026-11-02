# MGPU Bridge frame generation flicker fix

[中文](README.md)

A patch for [MGPU Bridge](https://github.com/maohgad-web/Neural-coprocessor), based on [davidm71's fork](https://github.com/davidm71/Neural-coprocessor) at commit `c40734d` (source `DG3`). Submitted to that fork as [PR #1](https://github.com/davidm71/Neural-coprocessor/pull/1), and to MGPU Bridge itself as a port in [PR #76](https://github.com/maohgad-web/Neural-coprocessor/pull/76) (built, not run; see [issue #75](https://github.com/maohgad-web/Neural-coprocessor/issues/75)). It fixes:

- **Flickering dark horizontal bands** over the picture with frame generation on while the game is focused, and horizontal flicker over the add-on's panel.
- A panel pointer that **sometimes does not move**, and that **stays on screen** after the panel closes.

## Symptoms

- Only while the game is focused. Move focus to another monitor and the bands stop at once, even with nothing covering the picture.
- Present with the camera still, so motion is not the cause.
- Not visible in screenshots.
- Stronger at higher frame generation multipliers.

## Cause

`present_frame` writes each back buffer in stages: clear to dark grey, copy the picture over it, draw the HUD, then ReShade draws the panel inside `Present`.

The swap chain had **2 buffers**. With the game focused it runs as `Hardware: Independent Flip` with tearing allowed, with flips as little as about 0.3 ms apart. The screen was then caught reading the buffer being written, and a band shows whichever stage had been reached:

| Stage when read | What you see |
|---|---|
| Just cleared | Dark band |
| Copied, panel not yet drawn | Bands of game image over the panel |

Screenshots miss it because capture reads the buffer after the copy.

### Evidence

1. **Dye:** with the clear colour changed to magenta, the bands turned magenta.
2. **Removal:** without the clear, the dark bands were gone at 2X and 4X, but the panel still flickered, so the clear is one visible stage of a wider problem.
3. **More buffers,** with the probe clearing every frame to magenta:

| Buffers | 2X | 4X |
|---|---|---|
| 2 | Persistent magenta bands, panel flickers | Not recorded separately |
| 3 | One magenta frame on opening the panel | A few bands |
| 4 | Clean | A few bands |
| **6** | **Clean, panel steady** | **Clean** |

## What changed

- **Clear only when needed:** the clear runs only when the picture will not cover the back buffer, so frames no longer pass through a dark grey stage. This removes the dark bands.
- **`PresentBuffers`:** 2-6, default 6, adjustable in the panel, applied on restart. This stops the panel flicker.
- **`PresentRaceProbe=1`:** keeps the magenta clear as a diagnostic that needs no special build.
- **Scaler:** descriptor sets are per buffer, so they do not collide at higher buffer counts.
- **Pointer left on screen:** nothing reset `MouseDrawCursor` when the panel closed. It is now reset.
- **Pointer that does not move:** the mouse hook won for the whole opening once it had seen one event, and this game keeps the hook nearly silent. Whichever source moved last is now used. In a log of eight panel openings, this explains all three stuck ones.

### Latency

Present-to-display time did not grow with more buffers:

| Buffers | Focused frames | p50 | p99 |
|---|---|---|---|
| 2 | about 9,400 | 0.50 ms | 2.66 ms |
| 4 | about 600 | 0.37 ms | 2.45 ms |
| 6 | about 30,800 | 0.39 ms | 2.42 ms |

All measured with `PresentLatency1=1`. This covers present-to-display only, not full input-to-photon latency.

## Install

You need a working MGPU Bridge install (davidm71's DG3).

Download: [MGPU_Bridge_BB1.zip](release/MGPU_Bridge_BB1.zip) (SHA-256: `347633912b3c691442c3bc57b51a1197d8b95cd9af3f843b9020b0fbefd0f542`)

1. Exit the game completely.
2. Back up `nvngx.dll_mgpu_bridge.addon64` in the game folder, next to `dxgi.dll`.
3. Replace it with the file of the same name from the release.
4. **Do not overwrite your `mgpu.ini`.** With no new keys, 6 buffers is the default.
5. In-game, `ReShade.log` should show `source BB1` and `present chain created ... buffers=6`.

To roll back, restore your backup.

## Test scope

Tested on one setup only. **Other GPUs, games and settings are unverified.**

- 2 × RTX 5080 (one rendering, one running DLSS), driver 621.01
- Stellar Blade, 4K 120 Hz, `DcompOverlay=1`, `Scaler=0`, frame generation 2X and 4X

## Still open

- **Why the screen reads a buffer still being written.** The explanation above fits the evidence best, but the exact mechanism is not established. The buffers needed rise with the multiplier, which fits flips queued ahead of the screen. That is inference.
- **Why the focused chain does not wait for vsync.** The code calls `Present(1, 0)`, yet PresentMon reports `SyncInterval=0` while focused.
- **Latency with low-latency present off.** With `PresentLatency1=0` and 6 buffers the picture was confirmed clean, but latency was not captured. The original 2-buffer build in that setting already shows 12-24 ms p50 on the unfocused path.
- **The scaler path.** Reviewed, not run.

Blocky image quality at 4X is out of scope. The tester attributes it to DLSS frame generation itself, which was not verified.

## Build from source

```bash
git clone https://github.com/davidm71/Neural-coprocessor
cd Neural-coprocessor
git checkout c40734d
git am --keep-cr patches/bb1-on-davidm71-c40734d.patch
```

Dependencies are described in that repository's `.github/workflows/build.yml` (ReShade and NVIDIA DLSS headers into `ext/`). Then build with VS 2022 and CMake. The release was built with MSVC 14.44, Ninja, Release, and did not go through that repository's CI.

That CI currently fails its own guard, because `src/gputime.cpp:217` uses `GetClockCalibration`. This is unrelated to the patch.

## License and credits

MGPU Bridge is by Marcelo Guibout under the MIT license, with the fork maintained by davidm71. This patch uses the same license; see [LICENSE](LICENSE).

Diagnosis and patch were done with the help of [Claude Code](https://claude.com/claude-code).

If this fix helped, a star is welcome, and so is an issue with your results on other GPUs and games.
