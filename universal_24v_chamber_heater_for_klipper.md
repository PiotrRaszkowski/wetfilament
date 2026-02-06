# 24V Chamber Heater Control with Raspberry Pi Pico 2 & Klipper

## Project Overview

Adding a **NexGen3D Chamber Heater** (24V PTC + fan) to an **Ender 3 V3 SE** running Klipper,
controlled by a **Raspberry Pi Pico 2** as a secondary MCU.
Mixing two projects:
- https://www.printables.com/model/1424592-nexgen3d-chamber-heater-v2
- https://www.printables.com/model/1211645-how-to-add-115v220v-ac-chamber-heater-to-creality

Based on the AC chamber heater project by Carmelo Padula, adapted for 24V DC operation.

The intention was to add it for my **Ender 3 V3 SE** but it can work for any printer running Klipper.

### Key Design Decisions

- **24V DC** instead of 115V/220V AC — no mains voltage inside the printer enclosure
- **Dedicated 24V PSU** for the heater — does not load the printer's power supply
- **MOSFET modules** instead of SSR — simpler and cheaper for DC
- **Pico 2** as Klipper MCU — full PID control, temperature monitoring, GCode integration
- **Separate fan/heater wiring** — enables safe cooldown after heating
- **Pico 2 powered via USB** from Raspberry Pi (5V) — no additional power conversion needed

---

## Bill of Materials

| # | Component | Specification | Qty | Notes |
|---|---|---|---|---|
| 1 | NexGen3D Chamber Heater | 24V PTC 200W + 24V 7530 blower fan | 1 | https://www.printables.com/model/1424592-nexgen3d-chamber-heater-v2 or https://www.printables.com/model/1424592-nexgen3d-chamber-heater-v2 or you can use anything you want with a 24V fan+heater |
| 2 | Raspberry Pi Pico 2 | RP2350 | 1 | |
| 3 | MOSFET module (heater) | AOD4184A, 36V/15A/400W, 3.3V logic | 1 | PWM capable, logic-level |
| 4 | MOSFET module (fan) | AOD4184A, 36V/15A/400W, 3.3V logic | 1 | Same module as above |
| 5 | Power supply (heater) | 24V, 250W minimum (350W recommended) | 1 | e.g. Meanwell LRS-350-24 |
| 6 | NTC thermistor | 100K, NTC 3950 | 1 | Chamber temperature sensor |
| 7 | Pull-up resistor | 4.7kΩ | 1 | Voltage divider for NTC |
| 8 | Thermal cutoff switch | KSD9700 75°C NC (normally closed) | 1 | Hardware safety |
| 9 | Inline fuse holder + fuse | Glass fuse 5x20mm, 15A | 1 | 200W/24V = 8.3A + inrush margin |
| 10 | Wire | 1.5 mm² (≥18AWG), silicone/PTFE insulated | ~2m | Heat resistant insulation |
| 11 | Dupont/JST-HR connectors/pins | For Pico 2 GPIO connections | misc | — |

### Component Notes

**MOSFET module (AOD4184A):**
- Must be **logic-level** — works with 3.3V signal from Pico 2
- IRF520 modules will NOT work (require ~10V gate voltage)
- Search for: "MOSFET PWM 36V 400W 15A module" or "D4184 module"

**Fuse sizing:**
- Heater draws 200W / 24V = 8.3A continuous
- PTC heater has inrush current spike at cold start (~10-12A)
- 15A fuse provides margin for inrush without nuisance tripping
- For comparison: the AC project uses 4A fuse because 400W / 115V = 3.5A (higher voltage = lower current)

**Wire gauge:**
- 1.5 mm² rated for ~15-16A, heater draws 8.3A — nearly 2x safety margin
- Original NexGen3D BOM specifies 18AWG (0.82 mm²), so 1.5 mm² exceeds requirements
- Use silicone or PTFE insulated wire near the heater element

**Power supply:**
- Dedicated PSU for heater only — do not share with printer PSU
- Printer (heated bed + hotend + steppers) draws ~200-250W from its own 350W PSU
- Adding 200W heater with startup spikes would risk voltage drops and board resets
- GND of both PSUs must be connected together

---

## Hardware Modification: Separate Fan & Heater Wiring

⚠️ **Critical step.** The NexGen3D heater ships with the fan and PTC heater wired in parallel
on 2 shared wires. You must separate them into 4 independent wires.

**Why this matters:**
- Without separation, turning off the heater also kills the fan
- Hot PTC element without airflow will warp the ABS housing (author's own warning)
- Klipper's `heater_fan` feature needs independent fan control for automatic cooldown

**How to do it:**
1. Open the NexGen3D heater housing (remove the 8x M3 cover screws)
2. Disconnect the parallel wiring between fan and PTC
3. Run separate wire pairs:
   - 2 wires → 24V 7530 blower fan (~3-5W)
   - 2 wires → 24V PTC heater element (200W)
4. Route all 4 wires out through the cable exit
5. Reassemble the housing

---

## Wiring Diagram

### System Overview

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║   MAINS 230V ──→ [5V PSU] ──→ Raspberry Pi ──USB──→ Pico 2            ║
║   MAINS 230V ──→ [24V PSU] ──→ Heater circuit (via MOSFETs)           ║
║                       │                                               ║
║                      GND ──────────────────→ Pico 2 GND pin           ║
║                                              (one wire!)              ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Detailed Wiring

```
                        24V PSU (dedicated, 250W+)
                       ┌────────┴────────┐
                      +24V              GND ─────────────────┐
                       │                 │                   │
           ┌───────────┤                 │                   │
           │           │                 │                   │
       [FUSE 15A]      │                 │                   │
           │           │                 │                   │
       [KSD9700]       │                 │                   │
        75°C NC        │                 │                   │
           │           │                 │                   │
           │           │                 │                   │
         PTC +       FAN +               │                   │
           │           │                 │                   │
      [ HEATER ]  [ FAN 7530 ]           │                   │
       PTC 200W     ~3-5W                │                   │
           │           │                 │                   │
         PTC -       FAN -               │                   │
           │           │                 │                   │
        ┌──┘           └──┐              │                   │
        │                 │              │                   │
    LOAD+ LOAD-      LOAD+ LOAD-         │                   │
    ┌─────────┐      ┌─────────┐         │                   │
    │ MOSFET  │      │ MOSFET  │         │                   │
    │ MODULE  │      │ MODULE  │         │                   │
    │   M1    │      │   M2    │         │                   │
    └────┬────┘      └────┬────┘         │                   │
     SIG│VCC│GND     SIG│VCC│GND         │                   │
      │  │   │        │  │   │           │                   │
      │  │   │        │  │   │           │                   │
    ┌─┴──┴───┴────────┴──┴───┴───────────┴───────────────────┤
    │                                                        │
    │                 RASPBERRY PI PICO 2                    │
    │                                                        │
    │  GP15 ──→ SIG M1 (heater MOSFET)                       │
    │  GP14 ──→ SIG M2 (fan MOSFET)                          │
    │  3.3V ──→ VCC M1 + VCC M2 + pull-up resistor           │
    │                                                        │
    │  GP26 (ADC0) ←── ┬── [NTC 100K 3950] ──→ GND           │
    │                   │                                    │
    │  3.3V ──→ [4.7kΩ resistor] ──→ ┘                       │
    │                                                        │
    │  GND ──→ GND M1 + GND M2 + 24V PSU GND                 │
    │                                                        │
    │  USB ←──────────────────────────────→ Raspberry Pi     │
    │                                       (power + data)   │
    └────────────────────────────────────────────────────────┘
```

### Heater Line Detail (series chain)

```
+24V → [FUSE 15A] → [KSD9700 75°C NC] → PTC heater (+) → PTC heater (-) → MOSFET M1 LOAD → GND
```

### NTC Temperature Sensor Circuit

```
3.3V (Pico) ──→ [4.7kΩ] ──┬── [NTC 100K 3950] ──→ GND (Pico)
                            │
                         GP26 (ADC0)
```

### Common Ground (Critical!)

```
24V PSU GND ────┬──── MOSFET M1 GND
                ├──── MOSFET M2 GND
                └──── Pico 2 GND pin

Pico 2 is powered via USB from Raspberry Pi (5V).
The single GND wire from 24V PSU to Pico provides
the common reference for MOSFET gate signals.
```

---

## Pico 2 GPIO Pin Assignment

| GPIO | Function | Direction | Connected To |
|---|---|---|---|
| GP15 | Heater PWM control | Output | MOSFET M1 SIG pin |
| GP14 | Fan control | Output | MOSFET M2 SIG pin |
| GP26 (ADC0) | Chamber temperature | Input (analog) | NTC voltage divider |
| 3.3V | Logic power | Power | MOSFET VCC pins + NTC pull-up |
| GND | Common ground | Power | MOSFET GND pins + 24V PSU GND + NTC |
| USB | Communication + power | Data/Power | Raspberry Pi USB port |

---

## Firmware: Flashing Klipper on Pico 2

### Build Firmware

On the Klipper host (Raspberry Pi):

```bash
cd ~/klipper
make menuconfig
```

Configuration:
```
Micro-controller Architecture: Raspberry Pi RP2350
Communication interface: USB (UART over USB)
```

> **Note:** If your Klipper version doesn't support RP2350 yet,
> select `Raspberry Pi RP2040` — the RP2040 firmware runs on Pico 2
> in compatibility mode.

```bash
make
```

### Flash to Pico 2

1. Hold the **BOOTSEL** button on Pico 2
2. Connect USB cable to Raspberry Pi
3. Release BOOTSEL — Pico appears as a USB mass storage device
4. Copy the firmware:
   ```bash
   cp out/klipper.uf2 /media/pi/RPI-RP2/
   ```
5. Pico reboots automatically with Klipper firmware

### Identify Serial Port

```bash
ls /dev/serial/by-id/
```

Expected output:
```
usb-Klipper_rp2350_XXXXXXXXXX-if00
```

Save this path for the Klipper configuration.

---

## Klipper Configuration

Add the following to `printer.cfg`:

### Secondary MCU

```ini
# ==============================================================
# CHAMBER HEATER — Pico 2 as secondary MCU
# ==============================================================

[mcu pico_chamber]
serial: /dev/serial/by-id/usb-Klipper_rp2350_XXXXXXXXXX-if00
# Replace with actual serial ID from: ls /dev/serial/by-id/
```

### Chamber Heater (PID controlled)

```ini
[heater_generic chamber_heater]
heater_pin: pico_chamber:gpio15
sensor_type: NTC 100K beta 3950
sensor_pin: pico_chamber:gpio26
min_temp: 0
max_temp: 80
control: pid
# Temporary values — run PID_CALIBRATE after installation:
pid_Kp: 30
pid_Ki: 1.0
pid_Kd: 300
```

### Heater Fan (automatic on/off with cooldown)

```ini
[heater_fan chamber_heater_fan]
pin: pico_chamber:gpio14
heater: heater_generic chamber_heater
heater_temp: 35.0
fan_speed: 1.0
```

> **How `heater_fan` works in Klipper:**
> - Fan turns ON automatically when heater is active
> - Fan stays ON until chamber temperature drops below `heater_temp` (35°C)
> - This ensures PTC element cooldown after heating stops
> - No additional relay needed (unlike the AC project)

### GCode Macros

```ini
[gcode_macro HEAT_CHAMBER]
description: Heat chamber to target temperature
gcode:
    {% set TEMP = params.TEMP|default(55)|int %}
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET={TEMP}
    M118 Heating chamber to {TEMP}°C...

[gcode_macro COOL_CHAMBER]
description: Turn off chamber heater (fan cools down automatically)
gcode:
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=0
    M118 Chamber heater off. Fan cooling down...

[gcode_macro WAIT_FOR_CHAMBER]
description: Wait until chamber reaches target temperature
gcode:
    {% set TEMP = params.TEMP|default(55)|int %}
    TEMPERATURE_WAIT SENSOR="heater_generic chamber_heater" MINIMUM={TEMP}
    M118 Chamber reached {TEMP}°C!
```

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

> **Important:** The `[heater_generic chamber_heater]` section must be in the main
> `printer.cfg` file (not in an included file), otherwise `SAVE_CONFIG` cannot
> write the PID results.

---

## Slicer Integration (OrcaSlicer)

### Machine Start G-code

Add before your START_PRINT macro:

```gcode
; Chamber heating
HEAT_CHAMBER TEMP={chamber_temperature}
WAIT_FOR_CHAMBER TEMP={chamber_temperature}
```

### Machine End G-code

Add after your END_PRINT macro:

```gcode
COOL_CHAMBER
```

### Recommended Chamber Temperatures

| Filament | Chamber Temp | Notes |
|---|---|---|
| PLA | 0 (off) | Does not need/want chamber heat |
| PETG | 0 (off) | Does not need chamber heat |
| ABS | 55-60°C | Eliminates warping, improves adhesion |
| ASA | 55-60°C | Similar to ABS |
| Nylon (PA) | 60-70°C | Higher temps for best results |
| PC | 65-75°C | Near max safe limit for printer components |

> ⚠️ Do not exceed 75-80°C — the KSD9700 thermal cutoff will trigger at 75°C,
> and high temperatures can damage printer components (belts, bearings, wiring).

---

## Safety Layers

The system has 5 independent safety layers:

| Layer | Type | Component | Function |
|---|---|---|---|
| 1 | Software | Klipper PID control | Regulates temperature to target |
| 2 | Software | Klipper `max_temp: 80` | Emergency shutdown if sensor reads >80°C |
| 3 | Hardware | KSD9700 75°C NC | Physically disconnects heater power above 75°C |
| 4 | Hardware | 15A fuse | Protects against short circuit |
| 5 | Software | `heater_fan` | Fan always runs when heater is active + cooldown |

### Comparison with AC Project

| AC Project Component | This 24V Project | Why |
|---|---|---|
| 40A SSR | MOSFET module (AOD4184A) | DC switching, simpler |
| 24V relay (fan interlock) | Klipper `heater_fan` | Software control with auto cooldown |
| KSD9700 75°C NC | KSD9700 75°C NC | Same — hardware thermal cutoff |
| NTC 100K 3950 | NTC 100K 3950 | Same — temperature sensing |
| AC fuse (4A @ 115V) | DC fuse (15A @ 24V) | Same power, lower voltage = higher current |
| BTT SKR Pico | Raspberry Pi Pico 2 | Full Klipper MCU |
| 115V/220V heater | 24V PTC heater | No mains voltage in enclosure |

---

## Installation Checklist

### Preparation
- [ ] Open NexGen3D heater and separate fan/PTC wiring into 4 independent wires
- [ ] Use 1.5 mm² silicone/PTFE wire for all power connections
- [ ] Mount KSD9700 thermal cutoff on heater housing (M3 screw or thermal adhesive)
- [ ] Mount NTC 100K thermistor in the center of the printer enclosure (not directly on heater)

### Electrical Assembly
- [ ] Wire heater line: +24V → fuse 15A → KSD9700 NC → PTC (+) → PTC (-) → MOSFET M1 LOAD
- [ ] Wire fan line: +24V → fan (+) → fan (-) → MOSFET M2 LOAD
- [ ] Connect MOSFET M1 SIG → Pico GP15, VCC → 3.3V, GND → GND
- [ ] Connect MOSFET M2 SIG → Pico GP14, VCC → 3.3V, GND → GND
- [ ] Wire NTC voltage divider: 3.3V → 4.7kΩ → GP26 → NTC → GND
- [ ] Connect 24V PSU GND to Pico 2 GND pin (critical — common ground reference)
- [ ] Connect Pico 2 to Raspberry Pi via USB

### Software Setup
- [ ] Flash Klipper firmware on Pico 2 (RP2350 or RP2040 compatibility mode)
- [ ] Identify serial port: `ls /dev/serial/by-id/`
- [ ] Add configuration sections to `printer.cfg`
- [ ] Restart Klipper: `FIRMWARE_RESTART`

### Testing & Calibration
- [ ] Verify Pico 2 MCU connection: check Klipper logs for `pico_chamber` MCU
- [ ] Verify temperature reading: `TEMPERATURE_WAIT` should show ambient temp
- [ ] Test fan: `SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=40`
- [ ] Verify fan turns on automatically when heater starts
- [ ] Run PID calibration: `PID_CALIBRATE HEATER=chamber_heater TARGET=55`
- [ ] Save results: `SAVE_CONFIG`
- [ ] Verify cooldown: set target to 0, confirm fan stays on until chamber <35°C

---

## Tips & Tricks

### Enclosure Insulation

Line the inside of your enclosure panels with **self-adhesive thermal shield mat** (aluminum + fiber,
rated for high temperatures, available as automotive heat shields). A 50x100cm sheet is enough
to cover the back and side panels. This reflects radiant heat back into the chamber, reduces heat
loss, speeds up heating, and lowers power consumption.

### Back Panel Material

If your back panel is foam PVC (Forex/Palight), replace it — foam PVC deforms at ~60°C.
Good alternatives:
- **HDF / hardboard (płyta pilśniowa)** — dense, rigid, handles 60-70°C well, cheap
- **Plywood 3-4mm** — similar properties to HDF

Line the inside with thermal mat regardless of material.

### Heater Mounting

- Cut an opening in the back panel and mount a **3mm aluminum plate (~150x100mm)** as a heater
  mounting bracket. Screw the NexGen3D bracket to the aluminum plate.
- Aluminum handles any temperature and provides a rigid, fireproof mount point.
- Available at any hardware store (Castorama, Leroy Merlin) for a few PLN.

### Heater Placement

- **Mount at the top of the back panel** — hot air rises naturally and circulates through the chamber.
- **Aim the outlet forward/downward** — toward the print zone, not directly at a wall.
- **Do not mount at the bottom** — hot air rises anyway, and you'd unnecessarily heat the bed from below.

### NTC Sensor Placement

- Mount the NTC thermistor on the **opposite side of the chamber** from the heater.
- This measures actual chamber temperature, not the hot air stream from the heater outlet.
- Avoid placing it near the heated bed or near enclosure openings (drafts cause false readings).

---

## References

- NexGen3D Chamber Heater: https://www.printables.com/model/863675-nexgen3d-chamber-heater
- AC Chamber Heater Guide (inspiration): https://www.printables.com/model/1211645-how-to-add-115v220v-ac-chamber-heater-to-creality
- Klipper heater_generic docs: https://www.klipper3d.org/Config_Reference.html#heater_generic
- Klipper heater_fan docs: https://www.klipper3d.org/Config_Reference.html#heater_fan
- Klipper secondary MCU: https://www.klipper3d.org/RPi_microcontroller.html
