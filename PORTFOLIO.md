# Wav50hz — Custom Split Keyboard with PS/2 TrackPoint Integration

A fully wireless, split ergonomic keyboard with an integrated IBM/Lenovo PS/2 TrackPoint
pointing stick, built on the ZMK firmware ecosystem.

---

## Project Overview

The Wav50hz is a custom 50-key split keyboard running on the Cosmos Lemon Wireless
(nRF52840) MCU board. The right half hosts an embedded pointing device connected
through a VIK (Versatile Input Konnector) ZIF connector — a standardized interface
that allows hot-swapping between different pointing device modules without any PCB
rework.

This portfolio entry documents the migration from a Microchip MaxTouch capacitive
trackpad (Procyon 42×50mm) to a ThinkPad-sourced PS/2 TrackPoint — a pointing stick
salvaged from a laptop keyboard and adapted to work wirelessly via Bluetooth.

---

## Hardware Stack

| Component | Part |
|-----------|------|
| MCU board | Cosmos Lemon Wireless (nRF52840, Bluetooth 5) |
| Keyboard firmware | ZMK (petejohanson `feat/pointers-move-scroll-ptp` fork) |
| Pointing device | IBM/Lenovo ThinkPad PS/2 TrackPoint (blue PCB + red cap) |
| Adapter | Custom PS/2 breakout PCB with 3V3/5V logic selection |
| Connector | VIK ZIF (shared with Procyon trackpad — direct hot-swap) |
| TrackPoint driver | [`infused-kim/kb_zmk_ps2_mouse_trackpoint_driver`](https://github.com/infused-kim/kb_zmk_ps2_mouse_trackpoint_driver) |

The adapter PCB exposes four signals: **CLK**, **DAT**, **RST**, **GND** — configured
for 3.3V logic to match the nRF52840's GPIO voltage.

---

## Engineering Challenges

### 1. PS/2 over Bluetooth: an interrupt timing problem

PS/2 is a synchronous protocol from 1987. The device clocks data at ~14.9 kHz,
requiring the host to sample each bit within **30–50 µs**. On a desktop, this is
trivial. On a wireless keyboard running Bluetooth, the BT radio constantly fires
interrupts that can steal CPU time for hundreds of microseconds — longer than the
entire PS/2 bit window.

**Solution:** The infused-kim driver solves this with two techniques:
1. **UART hardware reception** — instead of GPIO bit-banging for incoming data,
   the nRF52840's UART peripheral captures the PS/2 data stream at 14,400 baud
   (one UART bit ≈ one PS/2 clock cycle), offloading the timing-critical work to
   hardware.
2. **Interrupt priority inversion** — GPIOTE (GPIO interrupt controller) is
   promoted to priority 0 (highest on ARM Cortex-M4), and all other peripherals
   including the BT radio are demoted to priority 3. This guarantees PS/2 events
   are never blocked.

### 2. The three-state UART pinctrl problem

UART uses separate TX and RX lines. PS/2 uses a single bidirectional DATA line.
The driver reconciles this with three pinctrl states:

| State | UART TX | UART RX | Purpose |
|-------|---------|---------|---------|
| `default` | P0.27 (unexposed dummy) | P0.31 (DAT) | Receive PS/2 data via UART |
| `sleep` | P0.27 | P0.31 | Low-power mode |
| `off` | P0.27 | P0.28 (unexposed dummy) | Release DAT pin for GPIO bit-bang TX |

When the host needs to send a command to the TrackPoint (e.g., set sensitivity),
the driver switches to `off` state, giving GPIO full control of CLK (P0.29) and
DAT (P0.31) for bit-banged transmission. Then it switches back to `default` for
reception. P0.27 and P0.28 are internal nRF52840 pads not routed to any connector
on the Cosmos Lemon Wireless.

### 3. VIK connector pin reuse

The VIK connector exposes a fixed set of signals. The Procyon trackpad used:
- **I2C SDA/SCL** (P0.15, P0.13) for data
- **AD_1** (P0.29) as the MaxTouch CHG interrupt pin

The PS/2 TrackPoint repurposes those same physical connector pins:

| VIK pin | Old role (Procyon) | New role (TrackPoint) |
|---------|-------------------|----------------------|
| AD_1 (P0.29) | MaxTouch CHG interrupt | PS/2 CLK |
| AD_2 (P0.31) | Unused | PS/2 DAT (also UART RX) |
| SDA (P0.15) | I2C data | PS/2 RST |
| SCL (P0.13) | I2C clock | Unused |

No PCB modification required — the swap is entirely in firmware.

---

## Firmware Changes

Eight files were modified across the ZMK config repository:

### `config/west.yml`
Replaced `george-norton/maxtouch-zephyr-module` with
`infused-kim/kb_zmk_ps2_mouse_trackpoint_driver`. The PS/2 driver module provides
the `uart-ps2` and `zmk,input-mouse-ps2` device tree bindings and all associated
Kconfig symbols.

### `wav50hz_right.overlay` (primary change)
Complete rewrite of the pointing device section:
- 40-line interrupt priority override block covering all nRF52840 peripherals
- Three-state UART pinctrl for PS/2 protocol emulation
- `uart-ps2` driver node inside `&uart0` at 14,400 baud
- `zmk,input-mouse-ps2` device node with RST GPIO
- `zmk,input-listener-ps2` connecting the device to ZMK's input system

### `boards/cosmos_lemon_wireless.overlay`
Removed I2C pinctrl overrides (no longer needed). Retained SPI3 RGB underglow
and the VIK connector gpio-map (updated comments to reflect new PS/2 roles).

### `wav50hz_right.conf`
Swapped `CONFIG_I2C + CONFIG_INPUT_MICROCHIP_MAXTOUCH + CONFIG_ZMK_TRACKPAD_*`
for `CONFIG_UART=y + CONFIG_ZMK_INPUT_MOUSE_PS2_ENABLE=y`.

### `Kconfig.defconfig` / `Kconfig.shield`
Removed the four `ZMK_TRACKPAD_*` dimension symbols (logical/physical X/Y
resolution) that were specific to the MaxTouch module's calibration requirements.
The PS/2 TrackPoint driver does not require pre-declared dimensions — sensitivity
is adjusted at runtime via keymap bindings.

---

## Runtime Tuning (no reflash required)

A unique capability of the infused-kim driver: TrackPoint parameters can be
adjusted live and persist to flash after 60 seconds of inactivity.

| Parameter | Default | Keymap binding |
|-----------|---------|----------------|
| Sensitivity | 128 | `U_MSS_TP_S_I` / `U_MSS_TP_S_D` |
| Negative inertia | 6 | `U_MSS_TP_NI_I` / `U_MSS_TP_NI_D` |
| Upper plateau speed | 97 | `U_MSS_TP_V6_I` / `U_MSS_TP_V6_D` |

Axis inversion and X/Y swap are set in the device tree if needed.

---

## Test & Validation Milestones

| # | Milestone | Tool / Method | Signal |
|---|-----------|--------------|--------|
| 0 | Pre-build check | Module README, board schematic | P0.27/28 confirmed unexposed |
| 1 | Clean build | `west build -b cosmos_lemon_wireless -- -DSHIELD=wav50hz_right` | No errors |
| 2 | PS/2 init | USB serial log (`CONFIG_ZMK_USB_LOGGING=y`, `CONFIG_PS2_LOG_LEVEL_DBG=y`) | `0xaa` in log = self-test pass |
| 3 | Cursor movement | Host OS mouse movement | TrackPoint stick moves cursor |
| 4 | Button click | Red TrackPoint cap | Left-click registered |
| 5 | Sensitivity tuning | Runtime keymap bindings | Comfortable feel without reflash |
| 6 | Split integration | Flash both halves | Left keys + right TrackPoint work together |
| 7 | RGB validation | Visual inspection | 49-LED WS2812 underglow intact |
| 8 | Final clean build | Remove debug logging, rebuild both halves | Production firmware |

**Fallback path:** If 14,400 baud doesn't match this specific TrackPoint's clock
frequency, try 9,600 or 19,200 baud, then fall back to `gpio-ps2` driver.

---

## What This Demonstrates

- **Embedded systems**: nRF52840 peripheral configuration (UART, GPIO, pinctrl,
  interrupt priority) via Zephyr Device Tree
- **Protocol adaptation**: Emulating PS/2 (a 1987 protocol) using UART hardware
  on a modern BLE SoC
- **Firmware architecture**: ZMK module integration, Kconfig dependency management,
  device tree binding authorship
- **Hardware/software co-design**: Leveraging the VIK connector standard to swap
  pointing devices in firmware alone, with zero PCB changes
- **Debugging methodology**: Serial console logging, PS/2 self-test codes (`0xaa`/
  `0xfc`), interrupt priority analysis

---

## Branch / Status

Working branch: `claude/admiring-herschel`
Target: merge to `master` after hardware validation (Milestones 1–8 above)

The Procyon trackpad config is preserved in `master` history and can be restored
at any time by reverting this branch.
