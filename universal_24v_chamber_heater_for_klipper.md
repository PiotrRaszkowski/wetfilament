# Universal 24V Chamber Heater Control with Raspberry Pi Pico 2 & Klipper

## Project Overview

This project adds active chamber heating to an **Ender 3 V3 SE** (or any Klipper-based printer)
using a **24V PTC heater** controlled by a **Raspberry Pi Pico 2** as a secondary MCU.

### Inspiration & Source Projects

This design is a **hybrid of two Printables projects**, adapted for safe 24V DC operation:

1. **[NexGen3D Chamber Heater](https://www.printables.com/model/863675-nexgen3d-chamber-heater)** by NexGen-3D-Printing
   — Provides the **physical heater assembly**: ABS housing, 24V PTC element, 7530 blower fan,
   and the mounting bracket. Printed in ABS-CF (minimum ABS). No control system included —
   the author suggests connecting it directly to a spare PSU or using a relay.

2. **[115V/220V AC Chamber Heater for Klipper](https://www.printables.com/model/1211645-how-to-add-115v220v-ac-chamber-heater-to-creality)** by Carmelo Padula
   — Provides the **control system design**: Klipper integration via secondary MCU (BTT SKR Pico),
   NTC temperature sensing, KSD9700 thermal cutoff, fuse protection, PID regulation, and
   OrcaSlicer configuration. Designed for AC heaters (115V/220V) with SSR switching.

**Key differences in our 24V adaptation:**

| Aspect | NexGen3D Project | AC Klipper Project | This 24V Project |
|--------|-----------------|-------------------|-----------------|
| Heater | 24V PTC 200W | 115V/220V AC (Qibi Plus) | 24V PTC 200W (from NexGen3D) |
| Switching | None (always on) | 40A SSR (AC) | MOSFET modules (DC) |
| Control MCU | None | BTT SKR Pico (RP2040) | Raspberry Pi Pico 2 (RP2350) |
| Fan interlock | Wired in parallel | 24V relay | Klipper `heater_fan` (software) |
| Safety cutoff | Not included | KSD9700 75°C | KSD9700 **120°C** (see [rationale](#thermal-cutoff-ksd9700-120c)) |
| Mains voltage | No | Yes (inside enclosure) | No (24V DC only) |
| Klipper integration | No | Yes (PID, macros) | Yes (PID, macros, OrcaSlicer) |

---

## Bill of Materials

| # | Component | Specification | Qty | Notes |
|---|---|---|---|---|
| 1 | NexGen3D Chamber Heater | 24V PTC 200W + 24V 7530 blower fan | 1 | [Printables](https://www.printables.com/model/863675-nexgen3d-chamber-heater) |
| 2 | Raspberry Pi Pico 2 | RP2350 | 1 | Secondary Klipper MCU |
| 3 | MOSFET module (heater) | AOD4184A, 36V/15A/400W, 3.3V logic | 1 | PWM capable, logic-level |
| 4 | MOSFET module (fan) | AOD4184A, 36V/15A/400W, 3.3V logic | 1 | Same module as above |
| 5 | Power supply (heater) | 24V, 250W minimum (350W recommended) | 1 | e.g. Meanwell LRS-350-24 |
| 6 | NTC thermistor | 100K, NTC 3950 | 1 | Chamber temperature sensor |
| 7 | Pull-up resistor | 4.7kΩ | 1 | Voltage divider for NTC |
| 8 | Thermal cutoff switch | KSD9700 **120°C NC** (normally closed) | 1 | Hardware safety — see [rationale below](#thermal-cutoff-ksd9700-120c) |
| 9 | Inline fuse holder + fuse | Glass fuse 5×20mm, 15A | 1 | 200W/24V = 8.3A + inrush margin |
| 10 | Wire | 1.5 mm² (≥18AWG), silicone/PTFE insulated | ~2m | Heat resistant insulation |
| 11 | Dupont connectors/pins | For Pico 2 GPIO connections | misc | — |

### Component Notes

**MOSFET module (AOD4184A):**
- Must be **logic-level** — works with 3.3V signal from Pico 2
- IRF520 modules will NOT work (require ~10V gate voltage)
- Search AliExpress for: "MOSFET PWM 36V 400W 15A module" or "D4184 module"

**Fuse sizing:**
- Heater draws 200W / 24V = 8.3A continuous
- PTC heater has inrush current spike at cold start (~10–12A)
- 15A fuse provides margin for inrush without nuisance tripping

**Wire gauge:**
- 1.5 mm² rated for ~15–16A, heater draws 8.3A — nearly 2× safety margin
- Original NexGen3D BOM specifies 18AWG (0.82 mm²), so 1.5 mm² exceeds requirements
- Use silicone or PTFE insulated wire near the heater element

**Power supply:**
- Dedicated PSU for heater only — do not share with the printer's own PSU
- The printer (heated bed + hotend + steppers) already draws ~200–250W
- Adding a 200W heater with startup spikes would risk voltage drops and board resets
- **GND of both PSUs must be connected together** (common ground reference for MOSFETs)

---

## Hardware Modification: Separate Fan & Heater Wiring

⚠️ **Critical step.** The NexGen3D heater ships with the fan and PTC heater wired in parallel
on 2 shared wires. You must separate them into 4 independent wires.

**Why this matters:**
- Without separation, turning off the heater also kills the fan
- Hot PTC element without airflow will warp the ABS housing (author's own warning)
- Klipper's `heater_fan` feature needs independent fan control for automatic cooldown

**How to do it:**
1. Open the NexGen3D heater housing (remove the 8× M3 cover screws)
2. Disconnect the parallel wiring between fan and PTC
3. Run separate wire pairs:
   - 2 wires → 24V 7530 blower fan (~3–5W)
   - 2 wires → 24V PTC heater element (200W)
4. Route all 4 wires out through the cable exit
5. Reassemble the housing

---

## Thermal Cutoff: KSD9700 120°C

### Why 120°C instead of 75°C or 85°C

The KSD9700 is a **hardware emergency cutoff**, not a temperature regulator. It should only
trigger when something goes wrong (e.g. fan failure), never during normal operation.

The original AC project uses KSD9700 75°C because their heater (Qibi Plus) has its own enclosed
housing and the KSD is mounted **outside** the heater body. In the NexGen3D design, we mount
the KSD **directly on the aluminum PTC element** for fastest possible response — and this
changes the temperature requirements significantly.

**Temperature analysis at different mounting points:**

| Location | Normal operation (fan OK) | Fan failure |
|----------|--------------------------|-------------|
| Chamber air | 55–60°C | Slowly rises |
| Heater outlet air | 70–80°C | Drops (no flow) |
| PTC aluminum body | **70–90°C** | **150°C+** (rapid) |

With KSD mounted directly on the PTC metal body:
- **75°C** — will false-trigger during normal operation at higher chamber temps → cycling on/off
- **85°C** — still risks false triggers when targeting 60°C chamber
- **95–110°C** — workable but narrow margins
- **120°C** — **optimal**: well above normal operation (70–90°C), but responds instantly to
  fan failure (PTC reaches 150°C+ within seconds). 140°C would be too close to PTC max.

### Mounting Position

The KSD9700 120°C NC is mounted **under one of the M3 screws that secure the PTC element
to the ABS housing**. This provides:
- Direct metal-to-metal thermal contact with the PTC body
- Mechanical attachment without additional adhesive
- Fastest possible response to PTC overheating

**Mounting steps:**
1. Remove one of the 4× M3 screws holding the PTC to the housing
2. Place the KSD9700 metal tab under the screw head
3. Tighten the screw — ensures firm thermal contact
4. Optionally add a thin thermal pad between KSD and PTC for consistent contact

> **Note:** A thin layer of thermal paste between the KSD body and the PTC surface
> further improves thermal coupling and response time.

### Wiring

The KSD is wired **in series** with the PTC heater power line:

```
+24V → FUSE → MOSFET VOUT+ → KSD → PTC (+) → PTC (−) → MOSFET VOUT− → GND
```

---

## Wiring Diagram

### System Overview

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║   MAINS 230V ──→ [5V PSU] ──→ Raspberry Pi ──USB──→ Pico 2          ║
║   MAINS 230V ──→ [24V PSU] ──→ Heater circuit (via MOSFETs)         ║
║                       │                                               ║
║                      GND ──────────────────→ Pico 2 GND pin          ║
║                                              (one wire!)              ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Detailed Wiring

```
                        24V PSU (dedicated, 250W+)
                       ┌────────┴────────┐
                      +24V              GND ─────────────────┐
                       │                 │                    │
           ┌───────────┤                 │                    │
           │           │                 │                    │
       [FUSE 15A]      │                 │                    │
           │           │                 │                    │
           │           │                 │                    │
           │           │                 │                    │
         VIN+        VIN+               │                    │
     ┌─────────┐  ┌─────────┐           │                    │
     │ MOSFET  │  │ MOSFET  │           │                    │
     │   M1    │  │   M2    │           │                    │
     │(heater) │  │ (fan)   │           │                    │
     └─────────┘  └─────────┘           │                    │
      SIG  GND     SIG  GND            │                    │
       │    │       │    │              │                    │
       │    └───────┘    └──────────────┘                    │
       │    │       │                                         │
      GP15 GND    GP14                                       │
       └────┴───────┘                                         │
            │                                                 │
        [PICO 2]                                             │
            │                                                 │
           GND ──────────────────────────────────────────────┘
          GP26 ─── ← ── [4.7kΩ] ── ← ── 3.3V
           │
          NTC (in chamber, NOT on heater)


                     MOSFET M1 VOUT               MOSFET M2 VOUT
                    ┌────┴────┐                   ┌────┴────┐
                  VOUT+     VOUT-               VOUT+     VOUT-
                    │         │                    │         │
                 [KSD9700]    │                  FAN +     FAN -
                  120°C NC    │                    │         │
                    │         │                 [7530 FAN]   │
                  PTC +     PTC -                  │         │
                    │         │                    └────┬────┘
                 [PTC 200W]   │                         │
                    │         │                        GND
                    └────┬────┘
                         │
                        GND
```

### Power Flow Summary

```
+24V → FUSE (at PSU) → MOSFET M1 (VIN+→VOUT+) → KSD (on heater) → PTC → MOSFET M1 (VOUT-) → GND
       ↑                                          ↑                                    ↑
       short circuit                               overheat                             Klipper
       protection                                  protection                           control
```

### GPIO Pin Assignments

| Pico 2 Pin | Function | Connected To |
|------------|----------|-------------|
| GP15 | Heater PWM | MOSFET M1 SIG |
| GP14 | Fan control | MOSFET M2 SIG |
| GP26 (ADC0) | Temperature | NTC voltage divider |
| GND | Common ground | MOSFET GND + 24V PSU GND |
| 3V3(OUT) | Pullup reference | 4.7kΩ resistor → GP26 |

> **Critical:** Connect 24V PSU GND to Pico 2 GND. Without common ground reference,
> MOSFET gate signals won't work reliably. This is the only electrical connection
> between the 24V circuit and the Pico 2 (which is powered via USB from the Pi).

---

## Flashing Klipper on Raspberry Pi Pico 2 (RP2350)

The Pico 2 uses the **RP2350** chip, which requires different `menuconfig` settings than the
original RP2040-based Pico.

### Build Firmware

```bash
cd ~/klipper
make menuconfig
```

**Select these settings:**

```
[*] Enable extra low-level configuration options
    Micro-controller Architecture: Raspberry Pi RP2040/RP235x
    Processor model: rp2350
    Bootloader offset: No bootloader
    Communication Interface: USBSERIAL
```

> **Important:** Select `rp2350` as the processor model, NOT `rp2040`.
> The Pico 2 defaults to RP2350 ARM mode which Klipper supports.

Build the firmware:

```bash
make clean
make -j4
```

This generates `~/klipper/out/klipper.uf2`.

### Flash the Pico 2

1. **Disconnect** the Pico 2 from the Raspberry Pi USB
2. **Hold the BOOTSEL button** on the Pico 2
3. **While holding BOOTSEL**, connect the Pico 2 via USB to the Raspberry Pi
4. **Release BOOTSEL** — the Pico 2 appears as a USB mass storage device

```bash
# Mount and copy firmware
sudo mount /dev/sda1 /mnt
sudo cp ~/klipper/out/klipper.uf2 /mnt
sudo umount /mnt
```

The Pico 2 will automatically reboot with Klipper firmware.

### Subsequent Flashes (over USB, no BOOTSEL needed)

Once Klipper is running on the Pico 2, you can flash updates without pressing BOOTSEL:

```bash
sudo service klipper stop
make flash FLASH_DEVICE=/dev/serial/by-id/usb-Klipper_rp2350_XXXXXXXXXX-if00
sudo service klipper start
```

### Verify Connection

```bash
ls /dev/serial/by-id/
```

You should see something like:
```
usb-Klipper_rp2350_XXXXXXXXXX-if00
```

> **RP2350-E9 Errata Note:** The RP2350 has a known hardware errata (E9). For basic
> GPIO/ADC usage as in this project, it does not cause issues. The errata primarily
> affects certain memory access patterns. Klipper handles this correctly.

---

## Klipper Configuration

Add the following sections to your `printer.cfg`:

### Secondary MCU

```ini
[mcu pico_chamber]
serial: /dev/serial/by-id/usb-Klipper_rp2350_XXXXXXXXXX-if00
# Replace XXXXXXXXXX with your actual serial ID
```

### Chamber Heater (PID-controlled)

```ini
[heater_generic chamber_heater]
heater_pin: pico_chamber:gpio15
max_power: 1.0
sensor_type: NTC 100K beta 3950
sensor_pin: pico_chamber:gpio26
control: pid
# PID values will be calibrated — these are starting defaults
pid_Kp: 40.0
pid_Ki: 0.5
pid_Kd: 200.0
min_temp: 0
max_temp: 80
```

> **Important:** The `[heater_generic chamber_heater]` section must be in the main
> `printer.cfg` file (not in an included file), otherwise `SAVE_CONFIG` cannot
> write the PID calibration results.

### Chamber Fan (auto cooldown)

```ini
[heater_fan chamber_fan]
pin: pico_chamber:gpio14
heater: chamber_heater
heater_temp: 35.0
fan_speed: 1.0
```

This ensures the fan:
- Turns ON automatically whenever `chamber_heater` is active
- Stays ON after heating until the chamber drops below 35°C
- Protects the PTC element and ABS housing from heat damage

---

## G-Code Macros

### OrcaSlicer-Compatible Macros (M141 / M191)

OrcaSlicer uses **M141** (set chamber temperature) and **M191** (set and wait for chamber
temperature) to control active chamber heaters. These macros must be defined in Klipper
for OrcaSlicer integration to work.

```ini
# M141 — Set chamber heater temperature (no wait)
[gcode_macro M141]
gcode:
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET={params.S|default(0)}

# M191 — Set chamber heater temperature and wait until reached
[gcode_macro M191]
gcode:
    {% set s = params.S|float %}
    {% if s == 0 %}
        # Target is 0, turn off heater
        SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=0
        M117 Chamber heating off
    {% else %}
        SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET={s}
        # Optionally use heated bed to assist chamber heating:
        # M140 S100
        TEMPERATURE_WAIT SENSOR="heater_generic chamber_heater" MINIMUM={s-1} MAXIMUM={s+1}
        M117 Chamber at {s}C
    {% endif %}
```

### Convenience Macros

```ini
# Heat chamber to target temperature (non-blocking)
[gcode_macro HEAT_CHAMBER]
gcode:
    {% set temp = params.TEMP|default(0)|float %}
    M141 S{temp}

# Wait for chamber to reach target temperature (blocking)
[gcode_macro WAIT_FOR_CHAMBER]
gcode:
    {% set temp = params.TEMP|default(0)|float %}
    {% if temp > 0 %}
        M117 Waiting for chamber {temp}C...
        TEMPERATURE_WAIT SENSOR="heater_generic chamber_heater" MINIMUM={temp - 2}
        M117 Chamber ready
    {% endif %}

# Turn off chamber heater (fan auto-runs until cooldown)
[gcode_macro COOL_CHAMBER]
gcode:
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=0
    M117 Chamber cooling...
```

---

## OrcaSlicer Integration

OrcaSlicer has built-in support for active chamber heaters via the **M141/M191** commands.

### Enable Chamber Temperature Control

1. **Printer Settings → Basic Information → Accessory:**
   - Enable **"Support control chamber temperature"** checkbox

2. **Material Settings → Filament Tab → Temperature:**
   - Set **"Chamber temperature"** for each filament profile (e.g. 55°C for ABS)
   - Enable **"Activate temperature control"** checkbox

### How It Works

When both settings are enabled, OrcaSlicer automatically inserts:
- **`M191 S{chamber_temperature}`** at the **beginning** of G-code (before Machine Start G-code)
  — this heats the chamber and waits until the target is reached
- **`M141 S0`** at the **end** of the print — this turns off the chamber heater

### G-Code Variables Available in Machine G-Code

If you prefer to handle chamber heating in your START_PRINT macro instead of relying on
the automatic M191 insertion, you can use these variables:

```gcode
; Temperature of the first filament
M191 S{chamber_temperature[0]}

; Highest chamber temperature across all filaments (multi-material)
M191 S{overall_chamber_temperature}
```

### Example Machine Start G-Code (optional, if not using automatic insertion)

```gcode
; If using manual chamber control in START_PRINT:
HEAT_CHAMBER TEMP={chamber_temperature[0]}
WAIT_FOR_CHAMBER TEMP={chamber_temperature[0]}
```

### Example Machine End G-Code

```gcode
COOL_CHAMBER
```

### Recommended Chamber Temperatures

| Filament | Chamber Temp | Notes |
|---|---|---|
| PLA | 0 (off) | Does not need/want chamber heat |
| PETG | 0 (off) | Does not need chamber heat |
| ABS | 55–60°C | Eliminates warping, improves layer adhesion |
| ASA | 55–60°C | Similar to ABS |
| Nylon (PA) | 60–70°C | Higher temps for best results |
| PC | 65–75°C | Near max safe limit for printer components |

> ⚠️ Keep chamber temperature **below the filament's glass transition temperature**
> to prevent deformation. For most applications, 50–60°C is effective.
> Do not exceed 75–80°C — high temperatures can damage belts, bearings, and wiring.

---

## PID Calibration

After installation, calibrate PID for your specific enclosure:

```gcode
PID_CALIBRATE HEATER=chamber_heater TARGET=55
```

Wait for completion (several minutes), then save:

```gcode
SAVE_CONFIG
```

Klipper will automatically write the calibrated `pid_Kp`, `pid_Ki`, `pid_Kd` values.

---

## Safety Layers

The system has 5 independent safety layers:

| Layer | Type | Component | Function |
|---|---|---|---|
| 1 | Software | Klipper PID control | Regulates temperature to target |
| 2 | Software | Klipper `max_temp: 80` | Emergency shutdown if sensor reads >80°C |
| 3 | Hardware | KSD9700 **120°C NC** | Physically disconnects heater power on PTC overheat |
| 4 | Hardware | 15A fuse | Protects against short circuit |
| 5 | Software | `heater_fan` | Fan always runs when heater is active + auto cooldown |

### Why 120°C KSD is Safe Despite Higher Threshold

The Klipper `max_temp: 80` (Layer 2) monitors the **chamber air temperature** via the NTC
sensor. If the chamber reaches 80°C, Klipper shuts everything down — this is the primary
software safety.

The KSD9700 120°C (Layer 3) monitors the **PTC metal body temperature**. During normal
operation with the fan running, PTC body stays at 70–90°C. The 120°C threshold only triggers
if the fan fails and the PTC body starts climbing toward its 150–200°C operating limit.
This means the KSD acts as a **last-resort hardware failsafe for fan failure**, independent
of all software.

### Comparison with AC Project

| AC Project Component | This 24V Project | Why |
|---|---|---|
| 40A SSR | MOSFET module (AOD4184A) | DC switching, simpler |
| 24V relay (fan interlock) | Klipper `heater_fan` | Software control with auto cooldown |
| KSD9700 75°C NC | KSD9700 **120°C NC** | Mounted on PTC body, not outside housing |
| NTC 100K 3950 | NTC 100K 3950 | Same — temperature sensing |
| AC fuse (4A @ 115V) | DC fuse (15A @ 24V) | Same power, lower voltage = higher current |
| BTT SKR Pico (RP2040) | Raspberry Pi Pico 2 (RP2350) | Full Klipper MCU |
| 115V/220V heater | 24V PTC heater | No mains voltage in enclosure |

---

## Installation Checklist

### Preparation
- [ ] Open NexGen3D heater and separate fan/PTC wiring into 4 independent wires
- [ ] Use 1.5 mm² silicone/PTFE wire for all power connections
- [ ] Mount KSD9700 120°C NC thermal cutoff under M3 screw on PTC aluminum body
- [ ] Mount NTC 100K thermistor in the center of the printer enclosure (not on heater)

### Electrical Assembly
- [ ] Wire heater line: +24V → fuse 15A → MOSFET M1 (VIN+→VOUT+) → KSD9700 → PTC (+) → PTC (−) → MOSFET M1 (VOUT−) → GND
- [ ] Wire fan line: +24V → MOSFET M2 (VIN+→VOUT+) → fan (+) → fan (−) → MOSFET M2 (VOUT−) → GND
- [ ] Connect MOSFET M1 SIG → Pico GP15, VCC → 3.3V, GND → GND
- [ ] Connect MOSFET M2 SIG → Pico GP14, VCC → 3.3V, GND → GND
- [ ] Wire NTC voltage divider: 3.3V → 4.7kΩ → GP26 → NTC → GND
- [ ] Connect 24V PSU GND to Pico 2 GND pin (critical — common ground reference)
- [ ] Connect Pico 2 to Raspberry Pi via USB

### Software Setup
- [ ] Flash Klipper firmware on Pico 2 (`make menuconfig` → RP2350, USBSERIAL)
- [ ] Identify serial port: `ls /dev/serial/by-id/`
- [ ] Add all configuration sections to `printer.cfg`
- [ ] Define M141 and M191 macros for OrcaSlicer compatibility
- [ ] Enable "Support control chamber temperature" in OrcaSlicer printer settings
- [ ] Set chamber temperatures in OrcaSlicer filament profiles
- [ ] Restart Klipper: `FIRMWARE_RESTART`

### Testing & Calibration
- [ ] Verify Pico 2 MCU connection: check Klipper logs for `pico_chamber` MCU
- [ ] Verify temperature reading: should show ambient temp (~20–25°C)
- [ ] Test fan: `SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=40`
- [ ] Verify fan turns on automatically when heater starts
- [ ] Run PID calibration: `PID_CALIBRATE HEATER=chamber_heater TARGET=55`
- [ ] Save results: `SAVE_CONFIG`
- [ ] Verify cooldown: set target to 0, confirm fan stays on until chamber <35°C

---

## References

- NexGen3D Chamber Heater (physical assembly): https://www.printables.com/model/863675-nexgen3d-chamber-heater
- AC Chamber Heater for Klipper (control system inspiration): https://www.printables.com/model/1211645-how-to-add-115v220v-ac-chamber-heater-to-creality
- OrcaSlicer Chamber Temperature Wiki: https://github.com/OrcaSlicer/OrcaSlicer/wiki/Chamber-temperature
- OrcaSlicer Material Temperatures Wiki: https://github.com/OrcaSlicer/OrcaSlicer/wiki/material_temperatures
- Klipper heater_generic docs: https://www.klipper3d.org/Config_Reference.html#heater_generic
- Klipper heater_fan docs: https://www.klipper3d.org/Config_Reference.html#heater_fan
- Klipper secondary MCU: https://www.klipper3d.org/RPi_microcontroller.html
- Klipper RP2350 support: https://klipper.discourse.group/t/support-for-rp2350-micro-controllers/19656
