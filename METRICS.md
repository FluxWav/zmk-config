# Wav50hz — Data Metrics Reference

Raw numbers, timings, counts, and specs for use in documentation, infographics,
and diagrams. Organized by category.

---

## Repository & Code Change

| Metric | Value |
|--------|-------|
| Total commits in repo history | 57 |
| Files in repository (excl. .git) | 29 |
| Total firmware source lines (all .overlay / .conf / .dtsi / .dts / .keymap / Kconfig) | 846 |
| **Files changed in PS/2 migration** | **8** |
| Lines added (migration) | 136 |
| Lines removed (migration) | 126 |
| Net line change | +10 |
| Largest single-file change | `wav50hz_right.overlay` (+113 / -31) |

### Per-file line counts (post-migration)

| File | Lines |
|------|-------|
| `wav50hz.keymap` | 234 |
| `wav50hz_right.overlay` | 136 |
| `wav50hz.dtsi` | 65 |
| `wav50hz-layouts.dtsi` | 62 |
| `cosmos_lemon_wireless.overlay` | 72 |
| `cosmos_lemon_wireless.dtsi` | 97 |
| `cosmos_lemon_wireless-pinctrl.dtsi` | 40 |
| `wav50hz_right.conf` | 23 |
| `wav50hz_left.conf` | 13 |
| `wav50hz.conf` | 6 |
| `Kconfig.defconfig` | 24 |
| `Kconfig.shield` | 6 |
| `west.yml` | 19 |

---

## Keyboard Layout

| Metric | Value |
|--------|-------|
| Total physical keys | 50 |
| Layout halves | 2 (left + right) |
| Keymap layers | 8 |
| Layer names | Default, Nav, Mouse, Media, Num, Sym, Fun, Adj |
| Combos defined | 2 |
| Keymap source lines | 234 |
| RGB LEDs (WS2812 chain) | 49 |
| Keymap CONFIG options (right side) | 12 |
| Keymap CONFIG options (left side) | 8 |
| Shared CONFIG options | 3 |

---

## Hardware Specs

### MCU — Cosmos Lemon Wireless (nRF52840)

| Spec | Value |
|------|-------|
| SoC | Nordic nRF52840 |
| CPU | ARM Cortex-M4 @ 64 MHz |
| Wireless | Bluetooth 5.0 |
| GPIO ports | 2 (P0: 32 pins, P1: 16 pins) |
| Flash | 1 MB |
| RAM | 256 KB |
| Operating voltage | 3.3 V |

### RGB Underglow

| Spec | Value |
|------|-------|
| LED chip | WS2812 (GRB) |
| Chain length | 49 LEDs |
| SPI bus | SPI3 |
| SPI data pin | P0.26 |
| SPI max frequency | 4,000,000 Hz (4 MHz) |
| SPI one-frame byte | 0xE0 |
| SPI zero-frame byte | 0x80 |
| Reset delay | 50 µs |
| Brightness start | 50 / 255 |
| Effect on startup | Effect 3 (off at boot) |

### VIK Connector Pin Map

| VIK index | nRF52840 pin | Old role (Procyon) | New role (TrackPoint) |
|-----------|-------------|-------------------|----------------------|
| 0 — SDA | P0.15 | I2C SDA | PS/2 RST |
| 1 — SCL | P0.13 | I2C SCL | Unused |
| 2 — RGB Data | P0.26 | WS2812 data | WS2812 data (unchanged) |
| 3 — AD_1 | P0.29 | MaxTouch CHG (interrupt) | PS/2 CLK |
| 4 — MOSI | P0.12 | SPI MOSI | SPI MOSI (unchanged) |
| 5 — AD_2 | P0.31 | Unused | PS/2 DAT (+ UART RX) |
| 6 — CS | P0.08 | SPI CS | SPI CS (unchanged) |
| 7 — MISO | P0.06 | SPI MISO | SPI MISO (unchanged) |
| 8 — SCLK | P1.09 | SPI SCLK | SPI SCLK (unchanged) |

Pins repurposed: **3** (P0.15, P0.29, P0.31)
Pins left unchanged: **6**
PCB modifications required: **0**

---

## PS/2 Protocol & UART Driver

| Metric | Value |
|--------|-------|
| PS/2 protocol origin year | 1987 |
| PS/2 clock frequency (typical TrackPoint) | ~14,925 Hz |
| PS/2 bit cycle length (typical) | ~67 µs |
| UART baud rate used | 14,400 |
| UART bit cycle length | 69.44 µs |
| Timing error vs. true PS/2 | ~3.65 % |
| PS/2 allowed clock range | 10,000 – 16,700 Hz |
| Max interrupt service latency allowed | 30 – 50 µs |
| UART pinctrl states | 3 (`default`, `sleep`, `off`) |
| GPIOTE interrupt priority (new) | 0 (highest on Cortex-M4) |
| All other peripheral priority (new) | 3 |
| Default peripheral priority (before migration) | 1 |
| nRF52840 peripherals with adjusted priority | 44 |
| Unexposed dummy UART TX pin | P0.27 |
| Unexposed dummy UART RX pin | P0.28 |
| Fallback baud rate options | 9,600 / 19,200 |

### PS/2 Self-Test Response Codes

| Code | Meaning |
|------|---------|
| `0xAA` | Self-test passed — device initialized successfully |
| `0xFC` | Self-test failed — check wiring, RST signal, logic level |

### Alternate Baud Rate Comparison

| Baud rate | Cycle length | Delta from 67 µs | Delta % |
|-----------|-------------|-------------------|---------|
| 9,600 | 104.17 µs | +37.17 µs | +55.5 % |
| **14,400** | **69.44 µs** | **+2.44 µs** | **+3.6 %** |
| 19,200 | 52.08 µs | −14.92 µs | −22.3 % |

---

## Procyon Trackpad (removed — reference)

| Spec | Value |
|------|-------|
| Sensor | Microchip MaxTouch (mXT336UD) |
| Physical size | 42 mm × 50 mm |
| I2C address | 0x4A |
| Logical resolution X | 1050 (default) / 2100 (2× in conf) |
| Logical resolution Y | 1250 (default) / 2500 (2× in conf) |
| Physical resolution X | 420 (0.1 mm units = 42 mm) |
| Physical resolution Y | 500 (0.1 mm units = 50 mm) |
| CHG pin | P0.29 (VIK AD_1) |
| I2C bus | I2C0 (P0.15 SDA / P0.13 SCL) |
| Active acquisition time | 10 ms |
| Idle acquisition time | 32 ms |
| Touch threshold | 12 |
| Touch hysteresis | 5 |
| Gain | 8 |
| Orientation fixes required | 2 (invert-x + invert-y) |
| Orientation commits to land correct | 4 |

---

## PS/2 TrackPoint (new)

| Spec | Value |
|------|-------|
| Origin | IBM/Lenovo ThinkPad keyboard |
| Protocol | PS/2 |
| Supply voltage | 3.3 V (set via breakout PCB jumper) |
| PS/2 CLK pin | P0.29 (VIK AD_1) |
| PS/2 DAT pin | P0.31 (VIK AD_2) |
| PS/2 RST pin | P0.15 (VIK SDA) |
| Continuous current draw | ~3.0 – 3.85 mA |
| Default sensitivity | 128 (range: 0 – 255) |
| Default negative inertia | 6 (range: 0 – 15) |
| Default upper plateau speed | 97 (range: 0 – 255) |
| Default sampling rate | 100 Hz |
| Supported sampling rates | 10, 20, 40, 60, 80, 100, 200 Hz |
| Runtime tuning (no reflash) | Yes — persists to flash after 60 s |
| Tunable parameters | 5 (sensitivity, neg. inertia, speed, axes, rate) |

---

## ZMK Dependencies (before vs. after)

### Before (Procyon trackpad)

| Dependency | Remote | Revision |
|------------|--------|----------|
| zmk | petejohanson | feat/pointers-move-scroll-ptp |
| maxtouch-zephyr-module | george-norton | main |
| zmk-fingerpunch-vik | rianadon | main |
| vik-core | sadekbaroudi | (via deps.yml) |

### After (PS/2 TrackPoint)

| Dependency | Remote | Revision |
|------------|--------|----------|
| zmk | petejohanson | feat/pointers-move-scroll-ptp |
| kb_zmk_ps2_mouse_trackpoint_driver | infused-kim | main |
| zmk-fingerpunch-vik | rianadon | main |
| vik-core | sadekbaroudi | (via deps.yml) |

Dependencies changed: **1** (swap, not add)
Net dependency count: **unchanged (4)**

---

## Kconfig Symbols

### Removed (Procyon-specific)

```
CONFIG_I2C=y
CONFIG_INPUT_MICROCHIP_MAXTOUCH=y
CONFIG_ZMK_TRACKPAD=y
CONFIG_ZMK_TRACKPAD_LOGICAL_X=2100
CONFIG_ZMK_TRACKPAD_LOGICAL_Y=2500
CONFIG_ZMK_TRACKPAD_PHYSICAL_X=420
CONFIG_ZMK_TRACKPAD_PHYSICAL_Y=500
```

Symbols removed: **7**

### Added (PS/2-specific)

```
CONFIG_UART=y
CONFIG_ZMK_INPUT_MOUSE_PS2_ENABLE=y
```

Symbols added: **2**

Net Kconfig diff: **−5 symbols**

---

## Validation Test Checklist Counts

| Category | Tests |
|----------|-------|
| Pre-build (schematic + module compat) | 3 |
| Build validation | 4 |
| PS/2 initialization | 4 |
| Cursor + button | 3 |
| Sensitivity tuning | 5 |
| Split integration | 3 |
| RGB validation | 2 |
| Final clean build | 4 |
| **Total milestones** | **8** |
| **Total test checkpoints** | **28** |

---

## Timeline

| Date | Event |
|------|-------|
| 2025-11-14 | First commit in repository |
| 2026-02-03 | Most recent pre-migration commit (`c591ff4` — Rename board to Wav50hz) |
| 2026-03-27 | PS/2 TrackPoint migration implemented (this branch) |
| TBD | Hardware validation + merge to master |

---

## Quick-Reference Numbers for Infographics

```
50   keys
8    layers
49   RGB LEDs
29   files in repo
846  firmware source lines

8    files changed in migration
136  lines added
126  lines removed

44   nRF52840 peripherals with adjusted interrupt priority
3    UART pinctrl states
3    VIK pins repurposed
0    PCB modifications

14,400   UART baud rate (Hz)
67       PS/2 bit cycle (µs)
69.44    UART bit cycle (µs)
3.65     timing error (%)
30–50    max interrupt latency allowed (µs)

128      default TrackPoint sensitivity (0–255)
100      default sampling rate (Hz)
3.85     continuous current draw (mA)
```
