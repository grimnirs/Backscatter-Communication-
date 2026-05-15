# WCNES Project — Group 12 (Backscatter Communication)

## Course
Wireless Communication and Networked Embedded Systems, Project (1DT195), Uppsala University.

## Project Goal
**Primary dimension: Communication Range.**
Switch from the default 2-FSK modulation to **OOK (On-Off Keying)** to improve communication range. The hypothesis is that OOK's simpler demodulation requires lower SNR, extending range at the cost of throughput.

---

## Progress Log (as of 2026-05-05)

### Completed
- **Single-antenna 2-FSK baseline** verified working (only AE3/GPIO 27 connected; `TWOANTENNAS=false`, `PIN_TX1=27`).
- **OOK implementation in `backscatter.c`/`backscatter.h`**: added `ook` branch to PIO program generator. Symbol 1 toggles at d1 (tone at fc + CLK/d1); symbol 0 holds pin low (no backscatter). Config math: `fcenter = CLK/d1`, `fdeviation = 0`, `minRxBw = baud`.
- **`OOK_MODE` toggle in main.c** for A/B testing 2-FSK vs OOK.
- **Frequency band switched to 868 MHz** (sub-1 GHz CC1352 PCB antenna + OOK presets available in SmartRF). `CARRIER_FEQ` left at 2.45 GHz — irrelevant since on-Pico CC2500 is unused with CC1352 setup.
- **Sync word identified**: `0x930B51DE` (32-bit) from `packet_hdr_1352` in `packet_generation.c`.
- **Preamble identified**: 4 bytes (`0xAA 0xAA 0xAA 0xAA`) preceding the sync word. Full header = 4 preamble + 4 sync + 1 length + 1 seq = 10 B (`HEADER_LEN` in `packet_generation.h`).
- **Baud reduced to 5000** (TA recommendation). Pico serial confirms: baudrate 5000, Center offset 6944444 Hz, deviation 0, RX BW 5000.
- **Physical link validated**: at 874.944 MHz RX, huge RSSI rise when tag active (carrier + CLK/d1 = 868 + 6.944 MHz).
- **OOK modulation confirmed via spectrum analyser**: observed a single tone at 874.944 MHz (not the two sidebands expected of 2-FSK), confirming the PIO is producing OOK rather than 2-FSK.
- **Working OOK link at baseline (6.944 MHz offset, d1=18)**:
  - **PER = 2.91 %** (3 lost out of ~103 sent), **BER = 2.86 %**, mean RSSI ≈ −100.5 dBm.
  - Receiver: SmartRF Packet RX, 874.944 MHz, OOK 39 kHz BW preset, fixed length = 16 B (1 length byte + 1 seq byte + 14 payload bytes after sync), CRC disabled.
- **TX_DURATION restored to 250 ms** in `main.c` (was briefly 500). Empirical inter-packet period at receiver: **268 ms** (the on-air time is shorter than the naive 38.4 ms theoretical estimate; firmware loop period dominates).
- **Statistics notebook (`stats/statistics.ipynb`) extended**:
  - New gap-based PER analysis cell. Estimates lost packets from inter-arrival timestamp gaps using `round(gap / nominal_period) - 1`, with the nominal period derived empirically as the mean of non-spike gaps (gaps below 1.5× theoretical). This compensates for the receiver's "stop after 100 received packets" behaviour, which otherwise hides packet loss entirely.
  - New "Frequency offset sweep" section that loads multiple capture CSVs and produces a 3-panel comparison (PER / BER / mean RSSI vs offset, log x-axis).
- **Frequency-offset sweep experiment (10 data points, 100 received packets each)**:
  - Sweep was run at carrier 868 MHz with even PIO dividers (`d1` must be even — odd values trigger a runtime warning and produce an invalid tone). Achieved offsets: 0.500, 1.008, 2.016, 2.976, 5.208, 6.944, 8.929, 12.500, 15.625, 20.833 MHz.
  - **Sweet spot confirmed at 6.944 MHz** (existing baseline): lowest PER (2.91 %) and lowest BER (2.86 %) of the sweep.
  - **Wide reliable region 1–9 MHz**: PER ≤ 5 % across this range. The system is robust to offset choice within ±a few MHz of baseline.
  - **Cliff below 1 MHz** (carrier self-interference): at 0.5 MHz, PER = 91 %, BER = 43 % — link essentially dead.
  - **Cliff above ~12 MHz** (signal collapse): at 12.5 MHz PER = 46 %, at 20.8 MHz PER = 58 %. Likely combination of receiver bandwidth limits, antenna response roll-off, and divider-quantisation error.
  - **Reproducible 2 MHz anomaly**: at 2.016 MHz, PER = 22 % and BER = 41 % on rerun (worse than 1 MHz and 3 MHz neighbours). Suspected external interferer near 870 MHz or a receiver image / spur. Worth a no-tag RSSI scan to distinguish.
  - **PER and BER decouple in the mid-range**: e.g. at 8.929 MHz PER is fine (5 %) but BER is poor (30 %). Receiver locks on preamble + sync but the slicer struggles on the payload — a SmartRF-side optimisation lever, not a tag-side one.
  - **Best mean RSSI is at 3 MHz, not 7 MHz** — best signal strength does not equal best demodulation. Argues that BER is filter-aligned, not path-loss-limited.
- **Notebook environment fixed**: project root `.venv/` (Python 3.14.4) is the active kernel; `matplotlib`, `numpy`, `pandas`, `nbconvert` installed via pip. The older `stats/.venv/` (Python 3.9) is unused and can be deleted.

### 2026-05-05: 2-FSK regression attempted (for OOK-vs-2-FSK comparison)

**Goal:** capture a 2-FSK baseline at the same 5 kBaud / 6.944-MHz-area offset as the working OOK setup, then compare PER/BER/RSSI to argue OOK's range advantage.

**Theoretical analysis of achievable 2-FSK at this center frequency:**
- PIO divider must be even integer → frequency tones are restricted to `125 / (2k)` MHz.
- Around 6.94 MHz center, the only feasible even-divider pair is `d0=20, d1=18` → f0 = 6.250 MHz, f1 = 6.944 MHz, **center 6.597 MHz, deviation 347 kHz**.
- Minimum achievable FSK deviation at this center is 347 kHz (`125 / (d(d+2))` at d=18). **Cannot do narrow 2-FSK** (e.g. 5–25 kHz deviation typical for low-rate FSK) at 6.94 MHz center on this hardware without redesigning the PIO program for fractional dividers.
- Modulation index `h = 2δ/R = 694/5 ≈ 139` (huge; typical FSK uses h = 0.5–2). Means 2-FSK on this tag is forced into wide-bandwidth operation (~700 kHz RX BW vs OOK's 39 kHz). This is a **hardware-imposed bandwidth penalty on 2-FSK** — useful framing for the analytical-optimisation angle in the report.

**Configuration tried** (`main.c`):
- `OOK_MODE = false`
- `CLOCK_DIV0 = 20`, `CLOCK_DIV1 = 18`, `DESIRED_BAUD = 5000`
- Pico serial confirms computed parameters: baudrate 5000, Center offset 6597222 Hz, deviation 347222 Hz, RX BW 699444 Hz. No warnings.
- SmartRF RX: 874.597 MHz, 2-FSK, 5 kBaud, 347 kHz deviation, 783.6 kHz BW, sync `0x930B51DE`, fixed length 16 B, no CRC.

**Observed:** zero packets received. Even continuous-RX RSSI does not rise when the tag is plugged in.

**Diagnostics performed:**
1. **Reverted `OOK_MODE = true`** with the same `CLOCK_DIV1 = 18` — OOK link still works fine, ~3 % PER, full RSSI rise. Confirms carrier, antenna, RX hardware, and packet-format settings are all OK.
2. **Tried 2-FSK at 100 kBaud** (the framework's original validated baud) — still no signal received, no RSSI rise. Rules out a 5-kBaud-specific bug.
3. **Spectrum analyser scan** (ZNL6, span 873–876 MHz) with the tag in 2-FSK mode: **only one tone visible at ~874.9 MHz** (= carrier + f1 = 868 + 6.944). The expected second tone at **874.250 MHz (= carrier + f0)** is missing.
4. Reviewed `git diff` of `project_pico_libs/backscatter.c` against the original framework. The FSK branch (the `else { ... }` block in `generatePIOprogram`) appears byte-for-byte identical to the original code — i.e. our OOK additions did not modify the FSK code path. Despite this, only the f1 tone is being produced.

**Conclusion / blocker:** the FSK symbol-0 path of the PIO program is producing no f0 tone at d0=20. We don't yet know why — the C source for that branch reads as identical to the framework's original. Possible suspects (not yet verified):
- Build artefact / stale object — full clean rebuild of `carrier-receiver-baseband/build/` may be needed.
- Subtle interaction between the OOK additions and the FSK code path that isn't visible by inspection.
- Original framework FSK code never actually worked on this hardware in this configuration (we have no clean baseline of it being verified working end-to-end on our setup; the "Single-antenna 2-FSK baseline verified working" note in earlier log entries refers only to the on-Pico CC2500 receiver path getting RSSI hits, not full packet decode).

**Next steps for unblocking 2-FSK:**
- Stash our `backscatter.c`/`backscatter.h` modifications and test the original framework code path on its own (with `OOK_MODE` argument removed from `main.c` calls) — establishes whether FSK ever worked on our hardware setup.
- If the original framework FSK code also produces only one tone, the issue is hardware/RX-config rather than our code — and the 2-FSK comparison may not be feasible for this report.
- Alternative for the OOK-vs-other comparison: compare against a different OOK configuration (e.g. with vs without specific filtering/encoding tweaks), or focus the analytical angle on the bandwidth-penalty framing instead of running 2-FSK directly.

### Current Status
- **OOK baseline (6.944 MHz, 5 kBaud) works**: PER 2.91 %, BER 2.86 %.
- **First experimental optimisation complete**: frequency-offset sweep over 0.5–20 MHz, sweet-spot confirmed.
- **Second optimisation (2-FSK comparison) blocked**: tag emits only the f1 tone in 2-FSK mode despite code reviewing as correct.

### Known Quirks
- **2-FSK PIO produces only one tone (f1, missing f0)** — root cause unknown. Confirmed via ZNL6 spectrum analyser scan 873–876 MHz, span shows a single spike at ~874.9 MHz instead of the expected pair at 874.250 + 874.944 MHz.
- **PIO clock dividers `d0` and `d1` must be even** — odd values print "the clock divider d{0,1} has to be an even integer" on Pico boot and the state machine produces an invalid waveform. Affects achievable offsets (e.g. target 5 MHz → d1=24 → 5.208 MHz, not 5.000).
- **Framework's `compute_ber` prints a misleading "total packets transmitted by the tag is X"** based on `(seq[last] + 1) mod 256`. The seq counter is a global byte that wraps, so this number does not reflect actual transmissions. Use the gap-based "sent" estimate from the new analysis cell instead.
- **SmartRF TX silent-fail**: pressing Start sometimes doesn't begin TX. Restart Device Control Manager. (See memory `smartrf_tx_quirk.md`.)
- **BUFFER OVERFLOW on Pico serial**: caused by the local on-Pico CC2500 RX path trying to listen at 2.45 GHz. Not relevant at 868 MHz; can be disabled if it reappears (`setupReceiver`/`RX_start_listen` in main.c).

### Possible Next Steps
- **Resolve the 2-FSK blocker** (see "Next steps for unblocking 2-FSK" above), or pivot to optimisation #2 candidates that don't depend on it:
  - **Sweep SmartRF RX bandwidth** at fixed 6.944 MHz offset (39 / 78 / 200 kHz). The PER/BER decoupling in the mid-range from the offset sweep suggests RX bandwidth is the limiting factor for BER away from baseline.
  - **Investigate the 2 MHz anomaly** with a no-tag RSSI scan at 869 / 870 / 871 MHz to distinguish external interferer from receiver image/spur. Good fit for the analytical-optimisation angle (Grade 5).
- Document findings in report with SNR/BER theory reference (Varshney et al., LoRea SenSys 2017). The 2-FSK bandwidth-penalty framing (modulation index h ≈ 139, forced 700 kHz RX BW vs OOK's 39 kHz) is a strong analytical-optimisation thread regardless of whether 2-FSK packet decode is achieved.

---

## Grading Targets
- **Grade 4 (group):** Needs 2 experimental optimisations across status reports. Relate report to literature. Ask a question to another group at presentations.
- **Grade 5 (individual):** Analytical optimisation (apply theory/literature to observations). Motivate directions for further improvement in report.

---

## System Architecture

### Overview
This is a **backscatter communication** system. A carrier generator transmits a continuous wave. A backscatter tag reflects that signal and modulates it by switching antenna impedance states. A receiver decodes the modulated reflection.

### Hardware (3 devices)
1. **Carrier (TX):** TI CC1352P7 LaunchPad running SmartRF Studio 7 on a Windows laptop.
   - Continuous TX, Unmodulated, at the carrier frequency.
   - Has built-in sub-1 GHz PCB antenna (for 868 MHz). 2.4 GHz requires external SMA antenna.
2. **Backscatter Tag:** Raspberry Pi Pico (or Maker Pico) + custom backscatter frontend board with 4× frontends.
   - Connected to Mac via USB. Code is flashed as .uf2 file (drag to RPI-RP2 drive while holding BOOTSEL).
   - PIO (Programmable I/O) generates baseband signal to switch antenna impedance.
   - Impedance switching between open circuit (Γ=+1) and short circuit (Γ=−1) creates frequency shift.
3. **Receiver (RX):** TI CC1352P7 LaunchPad running SmartRF Studio 7 on a second Windows laptop.
   - Packet RX mode, tuned to the backscattered frequency (carrier + offset).

### Additional Equipment
- **Rohde & Schwarz ZNL6** Vector Network Analyser (5 kHz – 6 GHz). Useful for measuring antenna impedance and reflection coefficients of the backscatter frontend.

---

## Current Working 2-FSK Baseline

### Codebase
The main project is in `carrier-receiver-baseband/`. Key files:
- `main.c` — Entry point. Configures carrier, receiver, backscatter PIO, and runs TX/RX loop.
- `baseband/backscatter.h` / `backscatter.c` — PIO program generation for impedance switching. Computes modulation parameters.
- `baseband/packet_generation.h` / `packet_generation.c` — Packet framing: preamble, sync word, header, payload.
- `carrier-CC2500/` — SPI driver for CC2500 clicker board (used as carrier in simple setup, NOT used with CC1352).
- `receiver-CC2500/` — SPI driver for CC2500 clicker board (used as receiver in simple setup, NOT used with CC1352).
- `carrier-receiver-CC1352/` — SmartRF configuration guide for CC1352P7.

### Key Parameters (from Pico serial output)
```
Computed baseband settings:
- baudrate: 100000
- Center offset: 6597222 Hz
- deviation: 347222 Hz
- RX Bandwidth: 794444 Hz
```

### main.c Key Defines
```c
#define TX_DURATION            250   // ms between packets
#define RECEIVER              1352   // CC1352 receiver (not CC2500)
#define PIN_TX1                  6   // backscatter frontend pin 1
#define PIN_TX2                 27   // backscatter frontend pin 2
#define CLOCK_DIV0              20   // PIO clock divider for freq 0 (larger = lower freq)
#define CLOCK_DIV1              18   // PIO clock divider for freq 1 (smaller = higher freq)
#define DESIRED_BAUD        100000   // 100 kbaud
#define TWOANTENNAS          true   // use two antenna frontends
#define CARRIER_FEQ     2450000000   // 2.45 GHz (for CC2500 carrier, ignored when using CC1352)
```

### How 2-FSK Works in This System
- The PIO switches antenna impedance at two different frequencies: Δf0 and Δf1.
- Δf0 and Δf1 are derived from CLOCK_DIV0 and CLOCK_DIV1 respectively.
- The carrier at fc gets shifted to fc+Δf0 (for bit 0) and fc+Δf1 (for bit 1).
- The receiver tunes to the center frequency fR = fc + ½(Δf0 + Δf1).
- FSK deviation δ = ½|Δf1 − Δf0|.

### SmartRF Settings That Worked for 2-FSK
**Carrier CC1352:** Continuous TX, Unmodulated, 2450 MHz, 0 dBm.
**Receiver CC1352:** Packet RX, 2456.597 MHz, 100 kBaud, 347 kHz deviation, ~800 kHz RX BW.
Note: Packets were received (transmissions detected only when tag was active) but 100% packet error — likely sync word mismatch in SmartRF config. This is expected and not a blocker.

---

## OOK Implementation Plan

### What Needs to Change
OOK (On-Off Keying) encodes data by switching between **reflecting** and **not reflecting** (or reflecting vs absorbing) the carrier signal. Unlike 2-FSK which uses two different frequency shifts, OOK uses amplitude: signal present = 1, signal absent = 0.

### Approach Options (choose one)

#### Option A: Modify PIO to do OOK
Instead of switching between two impedance states at different frequencies, switch between:
- **Reflect** (short or open circuit, Γ = ±1) for bit "1"
- **Absorb/Match** (matched impedance, Γ = 0) for bit "0"
The challenge: the current hardware switches between open and short circuit. True absorption (Γ=0) requires matched impedance which may not be achievable with the current frontend.

#### Option B: Simple ON/OFF switching
- For bit "1": switch impedance at a single frequency Δf (creates backscatter signal at fc+Δf)
- For bit "0": stop switching (no backscatter signal)
This is effectively OOK from the receiver's perspective — signal present or absent.
The receiver on CC1352 would use an OOK preset in SmartRF (e.g., "4.8 kbps, OOK, 39 kHz RX Bandwidth" at the appropriate frequency).

#### Option C: Frequency shift to single tone + amplitude detection
Use a single constant Δf shift. The receiver detects presence/absence of that tone.

### Receiver Side (SmartRF)
SmartRF has OOK presets available. The receiver would need to be configured for:
- OOK demodulation
- Appropriate center frequency (fc + Δf)
- Matching baud rate
- Appropriate RX bandwidth

### Key Tradeoffs to Discuss in Report
- OOK requires lower SNR than FSK for same BER → potentially longer range
- OOK is more susceptible to amplitude noise/fading
- OOK has lower spectral efficiency
- Self-interference affects OOK differently than FSK

---

## Build & Flash Process

### Building
The project uses Pico SDK with CMake. The VS Code Pico extension handles the build.
1. Open `carrier-receiver-baseband/` folder in VS Code with Pico extension.
2. Click Compile/Build.
3. Output: `build/carrier_receiver_baseband.uf2` (and .elf)

### Flashing the Pico
1. Hold BOOTSEL button on Pico while plugging in USB.
2. Pico appears as USB mass storage drive (RPI-RP2).
3. Drag `carrier_receiver_baseband.uf2` onto the drive.
4. Pico reboots and runs automatically.

### Serial Monitor
```bash
# Find the port
ls /dev/tty.usb*
# Connect (115200 baud)
screen /dev/tty.usbmodem101 115200
```
The Pico prints computed radio settings at boot, then starts transmitting packets.

### Test Sequence (important order!)
1. Start carrier CC1352 (Continuous TX, Unmodulated)
2. Start receiver CC1352 (Packet RX with correct settings)
3. Plug in / reboot Pico (tag) LAST

---

## Repository Structure
```
wcnes-project2026/
├── baseband/                    # PIO backscatter + packet generation (CORE — modify this for OOK)
│   ├── backscatter.h
│   ├── backscatter.c
│   ├── packet_generation.h
│   └── packet_generation.c
├── carrier-receiver-baseband/   # Main project to build and flash (CORE)
│   ├── main.c
│   ├── CMakeLists.txt
│   ├── flash.sh
│   └── build/
├── carrier-CC2500/              # CC2500 carrier driver (not used with CC1352)
├── receiver-CC2500/             # CC2500 receiver driver (not used with CC1352)
├── carrier-receiver-CC1352/     # SmartRF config guide for CC1352
├── carrier-nrf52840/            # nRF52840 carrier config (alternative carrier)
├── carrier-Firefly/             # Firefly carrier config (alternative)
├── carrier-characteristics/     # Carrier characterisation tools
├── hardware/                    # Hardware documentation
├── project_pico_libs/           # Pico library dependencies
├── stats/                       # Statistics/analysis scripts
├── README.md
└── requirements.txt
```

---

## Important Theory (for analytical optimisation / Grade 5)

### Received Signal Strength
Pr = Pt × (λ²Gt / (4πD1)²) × (Gb²α|Γ|²/4) × (λ²Gr / (4πD2)²) ∝ 1/(D1²·D2²)

Where D1 = carrier-to-tag distance, D2 = tag-to-receiver distance. Worst case is D1 = D2.

### SNR
SNR = Pr / (N0 × B) where B = |Δf1 − Δf0| + 1/Ts (baudrate + frequency deviation)

### Reflection Coefficient
Γ = (ZL − Z0) / (ZL + Z0)
- Open circuit (ZL=∞): Γ = +1 (full reflection)
- Short circuit (ZL=0): Γ = −1 (full reflection, phase shifted)
- Matched (ZL=Z0): Γ = 0 (full absorption)

### Key Reference
Varshney et al., "LoRea: A Backscatter Architecture that Achieves a Long Communication Range," ACM SenSys 2017.

---

## Files to Focus On for OOK Implementation
1. `baseband/backscatter.c` — PIO program generation. This creates the impedance switching pattern. **Primary file to modify.**
2. `baseband/backscatter.h` — Config struct and function declarations.
3. `carrier-receiver-baseband/main.c` — Parameter defines and main loop. May need new defines for OOK mode.
4. `baseband/packet_generation.c` — Sync word and packet framing. May need changes for OOK-compatible framing.
