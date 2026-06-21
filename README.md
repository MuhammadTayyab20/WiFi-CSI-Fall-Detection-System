# WiFi CSI Fall Detection System

A WiFi sensing system using Channel State Information (CSI) to detect human movement and falls, built on two ESP32-S3 boards. Final Year Project work by Tayyab (Computer Engineering, Karachi). This is a separate project from an earlier completed LoRa-based rescue system FYP.

## Goal

Use the way a human body disturbs WiFi signals between two ESP32-S3 boards to detect movement and falls, ultimately surfaced through a live browser dashboard.

## Hardware

- 2x ESP32-S3-DevKitC-1 N16R8 (16MB flash, 8MB PSRAM)
- **TX board** (`active_ap` firmware) — COM12, MAC `AC:A7:04:13:42:91`
- **RX board** (`active_sta` firmware) — COM11, MAC `AC:A7:04:EF:97:88`
- Baud rate: 115200
- WiFi channel 6, 2.4GHz

## Firmware

Based on [ESP32-CSI-Tool](https://github.com/StevenMHernandez/ESP32-CSI-Tool) by Steven Hernandez, ported from ESP-IDF v4.x to **v6.0.1**. Key fixes applied for the version jump:

- Removed `esp_spi_flash.h` include (header removed in v6)
- Enabled driver-level WiFi CSI via menuconfig (`Component config → Wi-Fi → WiFi CSI`)
- Lowered FreeRTOS task priority from 100 → 5 in `active_sta/main/main.cc` (v6 caps priority below 25)
- Added `esp_mac.h` include in `active_ap/main/main.cc` (MACSTR/MAC2STR macros moved in v6)

Both boards are flashed and stream live CSI data over serial at ~10-11 packets/second.

## Data format

Each row from the RX board looks like:

```
CSI_DATA,STA,<TX_MAC>,<rssi>,...,<len>,[<imag> <real> <imag> <real> ...]
```

**Important:** the RX board tags each row with the **TX board's MAC** (the device it's hearing from), not its own. The bracketed array length varies per packet (commonly 256/384, sometimes others) depending on bandwidth/mode — any parser must handle variable-length arrays, not assume a fixed subcarrier count.

## Pipeline (Python, in `WiFi project/` folder)

| Script | Purpose |
|---|---|
| `capture.py` | Reads COM11, filters rows by TX MAC, saves to timestamped CSV (20s default) |
| `port_test.py` / `peek.py` | Diagnostics — confirm port opens and inspect raw row format |
| `analyze.py` | Computes a single average-amplitude "movement score" per file |
| `plot.py` / `plot_fall.py` / `plot_fall2.py` | Plot raw per-packet amplitude over time |
| `activity.py` / `activity2.py` | Rolling variance of raw amplitude (packet-to-packet change, smoothed) |
| `activity3.py` / `activity4.py` | Same, but with **per-packet normalization** first (divide each packet by its own mean to cancel gain noise) before measuring variance |

### Recorded reference clips

- `still.csv`, `still2.csv` — baseline, no movement (still2 recorded with phone/laptop in airplane mode for a cleaner baseline)
- `move.csv`, `move2.csv`, `move3.csv` — walking/movement near the boards (move3: deliberately crossing the direct line between the two boards repeatedly)
- `fall.csv` — cushion/bag dropped between the boards
- `fall2.csv` — large sudden body movement (crouch/sit) between still periods

## Findings so far

1. **Raw per-packet amplitude is too noisy to use directly.** Still and move clips overlap almost completely when measured this way — the ESP32's own gain/AGC noise is comparable in size to a human body's effect on the link.

2. **Per-packet normalization (dividing each packet by its own mean) is the most effective technique found.** It cancels gain swings and reveals shape changes across subcarriers. Applied in `activity3.py`/`activity4.py`.

3. **One clear, repeatable result:** in `fall2.csv` and in the beam-crossing test (`activity3.py`), a real movement event through the **direct line of sight between the two boards** produces a visible, distinct rise in normalized rolling activity — rising well above the still baseline and returning to baseline afterward. This confirms the underlying physics and pipeline work correctly.

4. **Sustained movement that doesn't cross the direct beam does not reliably separate from stillness** with the current board placement (boards close together, same desk). Median activity ratios between "still" and "move" clips have ranged from ~0.99x to ~1.13x — essentially no separation — except during the specific moments a person crosses directly between the two boards.

5. **This is a hardware/placement limitation, not an algorithm one.** No further signal-processing change is expected to extract more signal from the existing recordings; further improvement requires physically separating the boards (several meters apart, clear line of sight, chest/hip height) so a person's presence has a larger relative effect on the link. Through-wall placement was considered and rejected — it weakens the signal further rather than helping.

## Current status

- ✅ Hardware flashed and streaming
- ✅ Capture-to-file pipeline working and reproducible
- ✅ Signal processing pipeline (normalization + rolling variance) implemented correctly
- ✅ Beam-crossing detection demonstrated and repeatable
- ⏳ General room-movement / fall detection not yet reliable with current board placement
- ⏳ Live HTML dashboard — next planned step

## Next steps

1. Re-test with boards physically separated (several meters, clear line of sight, raised height) to see if separation improves
2. Build a live dashboard: Python script reads COM11 in real time, applies normalization + rolling variance, and pushes activity level to a browser page that updates every second, flagging when activity crosses a threshold
3. Document the line-of-sight / beam-crossing limitation honestly in the final report, alongside the working demo

## Notes on working style

- Hardware handled entirely by the project owner; this README/pipeline reflects the software and signal-processing side
- Boards can be safely unplugged between sessions — firmware is permanently flashed, no data loss
- Use a normal PowerShell for all Python work; the ESP-IDF PowerShell shortcut is reserved strictly for flashing/monitoring via `idf.py`
