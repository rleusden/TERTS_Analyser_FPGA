# TERTS Spectrum Analyzer — Project Documentation

> Inspired by the Elektuur Terts-Analyzer (March 1984 edition)
> A journey from an unaffordable teenage dream to a modern digital implementation.

---

## Table of Contents

- [Background](#background)
- [Version Overview](#version-overview)
- [v1.0 — Music Analyzer (ESP32-C3)](#v10--music-analyzer-esp32-c3)
- [v2.0 — Failed Detour (ESP32 classic + INMP441)](#v20--failed-detour-esp32-classic--inmp441)
- [v4.0 — TERTS Analyzer FPGA (Tang Nano 20K)](#v40--terts-analyzer-fpga-tang-nano-20k)
- [Technical Analysis](#technical-analysis)
- [Why Analog Still Wins on Framerate](#why-analog-still-wins-on-framerate)
- [Architecture Decisions](#architecture-decisions)
- [Hardware](#hardware)
- [Toolchain](#toolchain)
- [Progress Log](#progress-log)

---

## Background

In March 1984, the Dutch electronics magazine **Elektuur** published a three-part article on a real-time 1/3-octave spectrum analyzer — the "Terts-Analyzer". As a teenager at the time, building it was financially out of reach:

- 30 active bandpass filters with **1% tolerance components**
- 300 single-color LEDs
- Multiplexer ICs, comparators, a CD4060 clock oscillator
- Several circuitboards
- Estimated total cost: **300–400 Dutch guilders** (~€430 in today's purchasing power, applying the historical inflation factor)

42 years later, the same functionality can be built for **€45**, with RGB LEDs, higher display resolution, and digital stability that the original analog could never achieve.

---

## Version Overview

| Version | Name | Hardware | Status |
|---|---|---|---|
| v1.0 | Music Analyzer | ESP32-C3 + MAX4466 + 32×16 matrix | ✅ Working |
| v2.0 | — | ESP32 classic + INMP441 + 32×16 matrix | ❌ Abandoned |
| v3.0 | — | RP2040 + PCM1808 + 2× 16×16 matrix | ⏭ Skipped |
| v4.0 | TERTS Analyzer | Tang Nano 20K FPGA + MAX4466 + 1× 64x32 HUB75 matrix | 🔄 In development |

---

## v1.0 — Music Analyzer (ESP32-C3)

### Hardware
- **MCU:** MakerGo ESP32-C3 Super Mini (~€2)
- **Microphone:** MAX4466 analog electret amplifier (~€1.50)
- **Display:** 32×16 WS2812B LED matrix, column-oriented zigzag (Serpentine) wiring (~€24)
- **Total cost:** ~€30

### Audio Input Circuit
```
                        100kΩ ── 3.3V
                           │
MAX4466 out ──┤100nF├──────────────── GPIO 3 (ADC)
                           │
                        100kΩ ── GND
```
DC bias to 1.65V (midpoint of 3.3V ADC range). The coupling capacitor blocks DC from the source; `dcRemoval()` in the FFT library handles any remaining offset.

### Pin Assignment
| Signal | GPIO | Notes |
|---|---|---|
| Audio ADC | 3 | ADC1 CH3 — safe, no ADC2 bug |
| LED data | 10 | GPIO 8 conflicts with onboard LED |
| Button | 9 | Onboard BOOT button — works fine |
| L/R select | 26 | Pull LOW for mono left channel |

> ⚠️ **ESP32-C3 ADC2 (GPIO 5) is broken** — always use ADC1 (GPIO 0–4).
> ⚠️ **GPIO 8** drives the onboard blue LED — use GPIO 10 for WS2812B data.
> ⚠️ **Disconnect LED strip during flashing** to avoid boot issues.
> ⚠️ **Set "USB CDC on Boot" to Enabled** in Arduino IDE for Serial Monitor.

### FFT Configuration
```cpp
#define SAMPLES       512
#define SAMPLING_FREQ 40000    // Hz
#define NUM_BANDS     32
#define AMPLITUDE     4000     // Sensitivity — tune to taste
#define NOISE         1000     // Noise floor threshold
```

**Timing:**
- Sample time: `512 / 40000 = 12.8ms`
- FFT compute (ESP32-C3, no FPU, software float): ~0.3ms
- WS2812B update (512 LEDs): ~15ms, CPU-blocking
- **Effective framerate: ~50–76 FPS**

### Band Mapping
Logarithmic distribution across 32 bands, bins 2–255 of the FFT output:

```
Band 0:  bin 2        (~78 Hz)
Band 1:  bin 3        (~117 Hz)
...
Band 7:  bins 12–14   (~469–547 Hz)
Band 15: bins 77–93   (~3.0–3.6 kHz)
Band 31: bins 254–255 (~9.9–10.0 kHz)
```

### EQ Filter
Per-band correction factor to compensate for natural microphone roll-off at low and high frequencies:

```cpp
const byte eq[NUM_BANDS] = {
   60,  65,  70,  75,  80,  85,  90,  95,   // low bands: boost
  100, 100, 100, 100, 100, 100, 100, 100,   // mid bands: flat
  105, 110, 115, 120, 130, 140, 155, 170,   // upper mid: slight boost
  185, 195, 200, 200, 200, 200, 200, 200    // high bands: boost
};
```
Applied as: `barHeight = bandValues[band] * eq[band] / 100 / AMPLITUDE;`

### Libraries
| Library | Source |
|---|---|
| FastLED | Arduino Library Manager |
| FastLED_NeoMatrix | Arduino Library Manager |
| Adafruit_GFX | Arduino Library Manager |
| ArduinoFFT (v2.x) | Arduino Library Manager |
| EasyButton | Arduino Library Manager |

> **ArduinoFFT v2.x API change:** class is now `ArduinoFFT<double>`, methods are lowercase:
> `dcRemoval()`, `windowing()`, `compute()`, `complexToMagnitude()`

### Intro Animation
"MUSIC" is displayed statically on the top row, "Analyzer 2026" scrolls left across the bottom row in rainbow colors. FFT starts after the scroll completes.

Uses a custom `MatrixGFX` class (Adafruit_GFX subclass) with column-oriented zigzag (Serpentine) pixel mapping:
```cpp
// Even columns: top → bottom
// Odd columns:  bottom → top
index = (x % 2 == 0) ? (x * HEIGHT + y) : (x * HEIGHT + HEIGHT - 1 - y);
```

### WiFi / Bluetooth
Disabled at startup to eliminate ADC noise caused by RF activity:
```cpp
esp_wifi_stop();
esp_wifi_deinit();
esp_bt_controller_disable();
```

---

## v2.0 — Failed Detour (ESP32 classic + INMP441)

### Why it was attempted
- ESP32 classic has dual-core (240MHz) and hardware FPU
- INMP441 is a 24-bit I2S digital microphone — no DC bias circuit needed, lower noise
- I2S DMA would free the CPU from sampling duty

### Why it was abandoned
The INMP441 was **too sensitive for this use case**, with low-frequency noise (vibration, 50Hz hum) permanently lighting up the first 4 bands regardless of AMPLITUDE and NOISE settings. The MAX4466, with its adjustable gain trimmer, proved far more practical.

### I2S API note (ESP32 Arduino Core v3.x)
The legacy `driver/i2s.h` API conflicts with ESP-IDF v5.x. Use the new API:
```cpp
#include "driver/i2s_std.h"
// i2s_new_channel() + i2s_channel_init_std_mode() + i2s_channel_enable()
// i2s_channel_read() instead of i2s_read()
```

### Dual-core analysis
With I2S DMA, the sampling bottleneck shifts to the **DMA fill time**, not CPU speed:
```
Sample time (1024 @ 44100Hz) = 23.2ms  → 43 FPS
FFT compute (ESP32 FPU, float) = 0.2ms → negligible
LED update (FastLED) = 15ms            → overlaps with DMA on core 1
```
Dual-core ping-pong buffers help, but the DMA fill time dominates regardless.

---

## v3.0 — Skipped (RP2040 + PCM1808)

### Why it was considered
- RP2040 has true dual-core + PIO state machines
- PIO handles WS2812B timing in hardware — CPU not blocked during LED updates
- PCM1808 is a 24-bit stereo ADC with I2S output

### Architecture concept
```
MAX4466 mono ──┬── PCM1808 L input
               └── PCM1808 R input  (1kΩ isolation resistors)

Core 0: I2S DMA sampling → ping-pong buffer
         FFT bands 0–15  → PIO 0 → 16×16 matrix left
Core 1: FFT bands 16–29  → PIO 1 → 16×16 matrix right
```

### Why it was skipped
The Tang Nano 20K FPGA achieves the same goal (parallel filter processing, no Fourier bottleneck) at similar cost but with a fundamentally superior architecture — 30 parallel digital IIR bandpass filters running simultaneously, just like the original 1984 analog design.

---

## v4.0 — TERTS Analyzer FPGA (Tang Nano 20K)

### Motivation
The fundamental limitation of all FFT-based approaches is the **Fourier uncertainty principle**:

```
Frequency resolution = Sample rate / Number of samples
Sample time          = Number of samples / Sample rate
→ Sample time        = 1 / Frequency resolution
```

You cannot have both high frequency resolution and high frame rate simultaneously. This is why the 1984 analog design with 213 FPS cannot be matched by any FFT implementation:

| Config | Bin resolution | Frame rate |
|---|---|---|
| 512 @ 40kHz  | 78 Hz  | 76 FPS |
| 1024 @ 44.1kHz | 43 Hz | 43 FPS |
| 2048 @ 44.1kHz | 21 Hz | 22 FPS |
| 4096 @ 44.1kHz | 10 Hz | 11 FPS |

A **parallel digital filterbank** on an FPGA has no such limitation, just like the original analog design; all 30 filters run continuously and simultaneously.

### Hardware
- **FPGA:** Sipeed Tang Nano 20K (Gowin GW2AR-18, 20,736 LUT4, 48 DSP18×18 blocks)
- **Microphone:** MAX4466 (proven in v1.0)
- **Display:** 1x 64x32 HUB75 matrix
- **Total cost:** ~€70

### HUB75 advantage over WS2812B on FPGA
HUB75 works by sliding a row of pixels into a shift register and using a demultiplexer to select the correct row; all 64 pixels of a row are shifted in parallel. This fits perfectly with the FPGA architecture. The FPGA does exactly these kinds of parallel shift operations naturally. The HUB75 displays allow very high refresh rates, whereas the WS2812 refresh rates drop with every LED you add to the strip.

### Resources required (estimated)
| Resource | Available | Needed (30 IIR filters) |
|---|---|---|
| LUT4 | 20,736 | ~3,000 |
| DSP 18×18 | 48 | 30–60 |
| BSRAM | 828 Kbit | minimal |
| PLL | 2 | 1 |

### Architecture
```
MAX4466 ── ADC (external) ── FPGA
                              ├── 30× parallel IIR bandpass filters (FPGA fabric)
                              │    ├── Band 1:  25 Hz  (1/3 oct)
                              │    ├── Band 2:  31.5 Hz
                              │    │   ...
                              │    └── Band 30: 20 kHz
                              │
                              ├── Peak detection
                              ├── LED matrix left  (30x32)
                              └── LED matrix right (30x32)
```

### ISO 1/3 Octave Band Center Frequencies
| Band | Freq (Hz) | Band | Freq (Hz) | Band | Freq (Hz) |
|---|---|---|---|---|---|
| 1 | 25 | 11 | 250 | 21 | 2,500 |
| 2 | 31.5 | 12 | 315 | 22 | 3,150 |
| 3 | 40 | 13 | 400 | 23 | 4,000 |
| 4 | 50 | 14 | 500 | 24 | 5,000 |
| 5 | 63 | 15 | 630 | 25 | 6,300 |
| 6 | 80 | 16 | 800 | 26 | 8,000 |
| 7 | 100 | 17 | 1,000 | 27 | 10,000 |
| 8 | 125 | 18 | 1,250 | 28 | 12,500 |
| 9 | 160 | 19 | 1,600 | 29 | 16,000 |
| 10 | 200 | 20 | 2,000 | 30 | 20,000 |

Each band: `f_lower = f_center / 2^(1/6)`, `f_upper = f_center × 2^(1/6)`

### Expected Performance
- **Frame rate:** limited only by display update speed, not sampling
- **Frequency accuracy:** exact, no component drift
- **Dynamic range:** determined by ADC resolution
- **Latency:** near-zero (filters respond in real time)

---

## Technical Analysis

### Original Elektuur 1984 Update Rate

The CD4060 oscillator with R54=27kΩ, C4=330pF:
```
f_osc = 1 / (2.2 × 27kΩ × 330pF) = 51 kHz

CD4060 outputs used as 5-bit address for 1-of-30 multiplexer:
Q3 (÷8) = 6,377 Hz → multiplexer clock

Scan rate = 6,377 / 30 = 213 FPS
```

The analog design achieves 213 FPS because the 30 bandpass filters run **continuously in parallel** — the multiplexer only scans the LED display, not the audio processing.

### Comparison: Elektuur 1984 vs. v1.0 vs. v4.0

| Aspect | Elektuur 1984 | v1.0 ESP32-C3 | v4.0 Tang Nano 20K |
|---|---|---|---|
| Price | ~400 guilders | ~€16 | ~€35 |
| LEDs | 300 mono | 512 RGB | 2048 RGB |
| Height resolution | 10 pixels | 16 pixels | 32 pixels |
| Bands | 30 ISO terts | 32 logarithmic | 30 ISO terts |
| Frame rate | 213 FPS | ~76 FPS | >200 FPS (target) |
| Frequency accuracy | ±1% components | FFT limited | Exact |
| Drift over time | Yes (RC components) | No | No |
| Calibration | Manual, per band | One constant | One constant |
| Build time | Weeks | Hours | TBD |

### Why FFT Cannot Match Analog Frame Rate

The Fourier uncertainty principle is a mathematical law, not an engineering limitation. For ISO terts band 1 (25 Hz center frequency), the filter bandwidth is:
```
f_lower = 25 / 2^(1/6) = 22.3 Hz
f_upper = 25 × 2^(1/6) = 28.1 Hz
```
To resolve 22 Hz in an FFT requires at a minimum:
```
Samples needed = Sample rate / 22 Hz = 44100 / 22 = ~2000 samples
Sample time    = 2000 / 44100 = 45ms → max 22 FPS
```
An analog bandpass filter centered at 25 Hz responds in real time — no accumulation window required.

---

## Architecture Decisions

### Why MAX4466 over INMP441
- MAX4466 has an **adjustable gain trimmer** — critical for tuning sensitivity
- INMP441 is too sensitive for open-air use; low-frequency mechanical vibrations overwhelm the low bands
- MAX4466 requires a DC bias circuit (two 100kΩ resistors + 100nF cap) but this is trivial
- Proven working in v1.0

### Why 32×16 over 16×16 (v1.0)
- 32 bands fit naturally on 32 columns
- 16 rows gives 60% more height resolution than the original Elektuur 10-LED display

### Why 2× 16×16 (v4.0)
- 30 ISO terts bands fit on 32 columns (bands on x=1–30, x=0 and x=31 empty)
- Two square matrices look cleaner physically
- FPGA can drive both matrices independently via separate I/O

### Why Tang Nano 20K over RP2040 (v3.0 skipped)
- FPGA fabric implements truly parallel filters — no Fourier bottleneck
- BL616 RISC-V coprocessor handles LED matrix output
- Architecturally equivalent to the original analog design
- One board instead of two

### Why Verilog over VHDL (v4.0)
- Far more example projects available for Gowin/Sipeed boards
- Better community support on r/GowinFPGA and Sipeed forums
- VHDL possible but GW_JTAG primitive has known Verilog-only quirks

---

## Hardware

### v1.0 Bill of Materials

| Component | Value | Notes |
|---|---|---|
| ESP32-C3 Super Mini | — | MakerGo or equivalent |
| MAX4466 module | — | With gain trimmer |
| WS2812B matrix | 32×16 | Column-oriented zigzag (Serpentine)|
| Resistor | 2× 100kΩ | DC bias divider |
| Capacitor | 100nF | DC coupling |
| Pushbutton | — | Pattern select |

### v4.0 Bill of Materials (planned)

| Component | Value | Notes |
|---|---|---|
| Sipeed Tang Nano 20K | — | GW2AR-18 FPGA |
| MAX4466 module | — | Same as v1.0 |
| HUB75 matrix | 1× 64x32 | Left and Right Channel, 2x 30x32|
| External ADC | TBD | For FPGA audio input |
| Decoupling caps | 100nF | Per supply pin |

---

## Toolchain

### v1.0 (Arduino)
- Arduino IDE 2.x
- ESP32 Arduino Core **v3.3.8** (important: v3.x uses new I2S API)
- Board: "ESP32C3 Dev Module"
- USB CDC on Boot: **Enabled**

### v4.0 (FPGA)
- **Gowin EDA Pro** (free since September 2024)
  - Download from: gowin-semi.com
  - License server: `106.55.34.119:10559` (Sipeed hosted, no registration needed)
  - **Windows or Linux only — no Mac support**
- Alternative: **APIO** (open-source, Yosys + nextpnr) — less mature for GW2AR-18
- USB driver: WinUSB via Zadig for BL616 JTAG

### v5.0 — TERTS Analyzer HDMI
The fourth and final version of the TERTS Analyzer project adds HDMI video output to the v4.0 FPGA implementation, mirroring exactly what Elektuur did in May 1984, when they followed their LED matrix analyzer with a TV output version.
The HDMI display shows the output of all 30 ISO 1/3-octave bands in color on any modern HDMI monitor or TV at 1280×720 resolution. Each bar is subdivided into 16 sections, each representing a step of approximately 2 dB. The color of each bar changes per section so that the signal level of each band can be read at a glance. This makes the display suitable not only for personal use but also for demonstrations or situations where the analyzer needs to be read from a greater distance.
Where the original Elektuur TV output version was limited by the low resolution of a PAL composite video signal, v5.0 outputs full HD 1280×720 — allowing wider bars, finer color gradients, peak indicators, frequency labels in Hz, and visual elements that are physically impossible on an LED matrix.

No additional hardware required — the Tang Nano 20K already has a 27 MHz crystal and an onboard MS5351 clock generator specifically for HDMI output. v5.0 is a pure Verilog extension of v4.0.

---

## Progress Log

### June 2026
- [x] v1.0 hardware assembled and tested
- [x] FFT sketch working with 32 bands, 512 samples, ~76 FPS
- [x] Intro animation working ("TERTS" + scrolling "Analyser 2026")
- [x] EQ filter added per-band correction
- [x] MAX4466 sensitivity tuned via hardware trimmer
- [x] WiFi/BT disabled for lower ADC noise
- [x] v2.0 attempted and abandoned (INMP441 too sensitive)
- [x] v3.0 architecture designed but skipped in favor of FPGA approach
- [x] Tang Nano 20K ordered
- [ ] Tang Nano 20K arrived
- [ ] Gowin EDA Pro installed and first blink project working
- [ ] Single IIR bandpass filter implemented in Verilog
- [ ] 30-channel parallel filterbank implemented
- [ ] WS2812B output via BL616 RISC-V
- [ ] v4.0 full system integration
- [ ] Performance measurement vs. Elektuur 1984 target (213 FPS)
- [ ] v5.0 HDMI 1280×720 Output
