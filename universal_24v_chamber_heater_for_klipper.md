# 24V Chamber Heater Control with Raspberry Pi Pico 2 & Klipper

## Project Overview

Adding a **NexGen3D Chamber Heater** (24V PTC + fan) to an **Ender 3 V3 SE** running Klipper,
controlled by a **Raspberry Pi Pico 2** as a secondary MCU.

### Key Design Decisions

- **24V DC** instead of 115V/220V AC — no mains voltage inside the printer enclosure
- **Dedicated 24V PSU** for the heater — does not load the printer's power supply
- **MOSFET modules** instead of SSR — simpler and cheaper for DC
- **Pico 2** as Klipper MCU — full PID control, temperature monitoring, GCode integration
- **Separate fan/heater wiring** — enables safe cooldown after heating

---

## Bill of Materials

| # | Component | Specification | Qty | Notes |
|---|-----------|---------------|-----|-------|
| 1 | NexGen3D Chamber Heater | 24V PTC 200W + 24V 7530 blower fan | 1 | Modify to separate fan/PTC wiring |
| 2 | Raspberry Pi Pico 2 | RP2350 | 1 | Secondary Klipper MCU |
| 3 | MOSFET module | JZ-MOS / AOD4184A, 36V/15A, 3.3V logic | 2 | One for heater, one for fan |
| 4 | Power supply | 24V, 250W minimum | 1 | Dedicated for heater only |
| 5 | NTC thermistor | 100K, Generic 3950 | 1-3 | Chamber temperature sensing |
| 6 | Pull-up resistor | 4.7kΩ | 1-3 | One per thermistor |
| 7 | Thermal cutoff switch | KSD9700 **85°C NC** | 1 | Hardware safety (NOT 75°C!) |
| 8 | Inline fuse holder + fuse | Glass fuse 5x20mm, **15A** | 1 | 200W/24V = 8.3A + inrush margin |
| 9 | Wire | 1.5 mm², silicone insulated | ~2m | Heat resistant |
| 10 | Thermal shield mat | Self-adhesive, aluminum + fiber | 50x100cm | For enclosure insulation |
| 11 | Aluminum plate | 3mm, ~150x100mm | 1 | Heater mounting bracket |

### Important Notes on Components

**KSD9700 — Use 85°C, NOT 75°C!**
- PTC metal housing reaches 70-90°C during normal operation with fan running
- 75°C would cause nuisance trips during normal heating
- 85°C gives margin for normal operation but still triggers on fan failure (PTC goes >150°C)

**Fuse — 15A, NOT 10A!**
- Heater draws 8.3A continuous (200W / 24V)
- PTC has inrush current spike at cold start (~10-12A)
- 15A prevents nuisance blowing while still protecting against shorts

**MOSFET Module — Must be Logic-Level!**
- JZ-MOS / D4184 / AOD4184A work with 3.3V from Pico ✓
- IRF520 does NOT work — requires 10V gate voltage ✗

---

## Hardware Modification: Separate Fan & Heater Wiring

⚠️ **Critical step!** The NexGen3D heater ships with fan and PTC wired in parallel (2 shared wires).

**Why separate them:**
- Without separation, turning off heater also kills the fan
- Hot PTC without airflow will warp the ABS housing
- Klipper's `heater_fan` needs independent fan control for automatic cooldown

**How to do it:**
1. Open the NexGen3D heater housing (remove cover screws)
2. Disconnect the parallel wiring between fan and PTC
3. Run separate wire pairs:
   - 2 wires → 24V 7530 blower fan (~3-5W)
   - 2 wires → 24V PTC heater element (200W)
4. Install KSD9700 85°C on PTC metal housing (under mounting screw or with thermal paste)
5. Install fuse inline (can be inside heater housing)
6. Route all wires out through cable exit

---

## Wiring Diagrams

### System Overview

```
╔════════════════════════════════════════════════════════════════════════════╗
║                                                                            ║
║   MAINS 230V ──→ [5V PSU] ──→ Raspberry Pi ──USB──→ Pico 2                ║
║                                                                            ║
║   MAINS 230V ──→ [24V PSU 250W] ──→ Heater circuit (via MOSFETs)          ║
║                        │                                                   ║
║                       GND ─────────────────────────→ Pico 2 GND pin       ║
║                                                      (one wire!)           ║
║                                                                            ║
╚════════════════════════════════════════════════════════════════════════════╝
```

### MOSFET Module Pinout (JZ-MOS)

```
                    ┌─────────────────────────────────┐
                    │                                 │
     Screw          │    ┌───┐ ┌───┐                 │
     Terminals ───→ │    │ Q1│ │ Q2│   JZ-MOS       │
                    │    └───┘ └───┘                 │
                    │                                 │
                    │   VIN-  VIN+  VOUT- VOUT+      │
                    │    ●     ●     ●     ●         │
                    └────┼─────┼─────┼─────┼─────────┘
                         │     │     │     │
                         │     │     │     └── To load (+)
                         │     │     └──────── To load (-)  
                         │     └────────────── From PSU (+24V)
                         └──────────────────── From PSU (GND)
                    
                    
     Signal         ┌─────────────────────────────────┐
     Pins ────→     │  ○ ○ ○                          │
                    │ GND    TRIG/PWM                 │
                    │  │       │                      │
                    │  │       └── Signal from Pico GPIO
                    │  └────────── Ground (to Pico GND)
                    └─────────────────────────────────┘
```

### Complete Wiring Diagram

```
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃                         24V PSU (250W, dedicated)                         ┃
┃                        ┌─────────┴─────────┐                              ┃
┃                       +24V                GND                             ┃
┃                        │                   │                              ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                         │                   │
          ┌──────────────┴──────┐            │
          │                     │            │
          ▼                     ▼            │
   ┌─────────────┐       ┌─────────────┐     │
   │ MOSFET #1   │       │ MOSFET #2   │     │
   │ (HEATER)    │       │ (FAN)       │     │
   │             │       │             │     │
   │ VIN+  ◄──24V│       │ VIN+  ◄──24V│     │
   │ VIN-  ◄─────────────│ VIN-  ◄─────┼─────┤
   │             │       │             │     │
   │ VOUT+ ──────────┐   │ VOUT+ ─────────┐  │
   │ VOUT- ──────────┼─┐ │ VOUT- ─────────┼──┤
   │             │   │ │ │             │  │  │
   │ TRIG ◄──GP15│   │ │ │ TRIG ◄──GP14│  │  │
   │ GND  ◄──────────┼─┼─│ GND  ◄──────┼──┼──┤
   └─────────────┘   │ │ └─────────────┘  │  │
                     │ │                  │  │
                     │ │                  │  │
   ══════════════════╪═╪══════════════════╪══╪═══  Cable to heater housing
                     │ │                  │  │
   ┌─────────────────┼─┼──────────────────┼──┼───────────────────────────┐
   │  HEATER HOUSING │ │                  │  │                           │
   │                 │ │                  │  │                           │
   │                 │ │      ┌───────────┘  │                           │
   │                 │ │      │              │                           │
   │                 │ │   FAN (+)       FAN (-)                         │
   │                 │ │      │              │                           │
   │                 │ │      ▼              ▼                           │
   │                 │ │   ┌─────────────────────┐                       │
   │                 │ │   │    7530 BLOWER     │                       │
   │                 │ │   │       FAN          │                       │
   │                 │ │   └─────────────────────┘                       │
   │                 │ │                                                 │
   │                 │ │                                                 │
   │                 ▼ ▼                                                 │
   │   ┌──────┐   ┌─────────┐   ┌───────────────────┐                   │
   │   │ FUSE │──►│ KSD9700 │──►│    PTC HEATER     │                   │
   │   │ 15A  │   │  85°C   │   │      200W         │                   │
   │   └──────┘   │   NC    │   └───────────────────┘                   │
   │              └─────────┘            │                               │
   │                  ▲                  │                               │
   │                  │                  ▼                               │
   │            Mounted on          Back to MOSFET #1                    │
   │            PTC metal           VOUT- (via cable)                    │
   │                                                                     │
   └─────────────────────────────────────────────────────────────────────┘


   ┌─────────────────────────────────────────────────────────────────────┐
   │                      RASPBERRY PI PICO 2                            │
   │                                                                     │
   │    USB ◄────────────────────────────────────► Raspberry Pi          │
   │                                                (power + data)       │
   │                                                                     │
   │    GP15 ─────────────────────────────────────► MOSFET #1 TRIG      │
   │    GP14 ─────────────────────────────────────► MOSFET #2 TRIG      │
   │    GND  ─────────────────────────────────────► MOSFETs GND         │
   │                                                + 24V PSU GND        │
   │                                                                     │
   │    3.3V ────┬────────────────────────────────► NTC pull-up         │
   │             │                                                       │
   │    GP26 ◄───┴─── [4.7kΩ] ───┬─── [NTC 100K] ──► GND               │
   │         (ADC0)              │                                       │
   │                        Voltage                                      │
   │                        divider                                      │
   │                        junction                                     │
   └─────────────────────────────────────────────────────────────────────┘
```

### Heater Circuit Flow (Current Path)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   +24V ──► FUSE 15A ──► KSD9700 NC ──► PTC (+) ──► PTC (-) ──┐          │
│                                                               │          │
│                         ┌─────────────────────────────────────┘          │
│                         │                                                │
│                         ▼                                                │
│                    MOSFET VOUT+                                          │
│                         │                                                │
│                    ┌────┴────┐                                           │
│                    │ MOSFET  │◄──── TRIG (GPIO15 from Pico)             │
│                    │ D4184   │                                           │
│                    └────┬────┘                                           │
│                         │                                                │
│                    MOSFET VOUT-                                          │
│                         │                                                │
│                         ▼                                                │
│                        GND ◄─────────────────────────────────── 24V PSU │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

When TRIG = HIGH (3.3V from Pico):  MOSFET conducts → Current flows → PTC heats
When TRIG = LOW  (0V from Pico):    MOSFET blocks   → No current   → PTC off
```

### NTC Thermistor Voltage Divider

```
        3.3V (from Pico)
           │
           │
        ┌──┴──┐
        │     │
        │4.7kΩ│  Fixed resistor
        │     │
        └──┬──┘
           │
           ├─────────────────► GP26 (ADC0) — to Pico
           │
        ┌──┴──┐
        │     │
        │ NTC │  100K @ 25°C, Beta 3950
        │100K │  (resistance drops as temp rises)
        │     │
        └──┬──┘
           │
           │
          GND


Voltage at GP26 = 3.3V × ( NTC / (4.7kΩ + NTC) )

At 25°C:  NTC = 100kΩ  →  V = 3.3 × (100/104.7) = 3.15V
At 50°C:  NTC = ~33kΩ  →  V = 3.3 × (33/37.7)   = 2.89V  
At 80°C:  NTC = ~12kΩ  →  V = 3.3 × (12/16.7)   = 2.37V
```

### Multiple Thermistors (Optional)

```
        3.3V (from Pico)
           │
     ┌─────┼─────┬─────────────┐
     │     │     │             │
  ┌──┴──┐ ┌┴──┐ ┌┴──┐          │
  │4.7kΩ│ │4.7│ │4.7│          │
  └──┬──┘ └─┬─┘ └─┬─┘          │
     │      │     │            │
     ├──►GP26     ├──►GP27     ├──►GP28
     │      │     │            │
  ┌──┴──┐ ┌─┴─┐ ┌─┴─┐          │
  │NTC 1│ │NTC│ │NTC│          │
  │     │ │ 2 │ │ 3 │          │
  └──┬──┘ └─┬─┘ └─┬─┘          │
     │      │     │            │
     └──────┴─────┴────────────┴──► GND

NTC 1 (GP26): Used by heater_generic for PID control
NTC 2 (GP27): Chamber ambient temperature
NTC 3 (GP28): Top of chamber (optional)
```

---

## KSD9700 Mounting

Mount the thermal cutoff switch **directly on the PTC metal housing**:

```
   ┌─────────────────────────────────────────┐
   │           PTC HEATER ELEMENT            │
   │  ┌───────────────────────────────────┐  │
   │  │                                   │  │
   │  │   ████████████████████████████   │  │
   │  │   █  Aluminum PTC housing    █   │  │
   │  │   █                          █   │  │
   │  │   █    ┌─────────┐           █   │  │
   │  │   █    │ KSD9700 │◄── Mounted under existing   
   │  │   █    │  85°C   │    M3 screw, or with       
   │  │   █    └─────────┘    thermal paste + tape    
   │  │   █                          █   │  │
   │  │   ████████████████████████████   │  │
   │  │                                   │  │
   │  └───────────────────────────────────┘  │
   │                  24V                     │
   └─────────────────────────────────────────┘

KSD9700 in series with PTC power line:
- Normal operation (PTC at 70-90°C): Contact CLOSED → heater works
- Fan failure (PTC rises >85°C):     Contact OPENS  → heater stops
- After cooling (<85°C):             Contact CLOSES → auto-reset
```

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
Micro-controller Architecture: Raspberry Pi RP2040
Communication interface: USB
```

> Note: Select RP2040 even for Pico 2 — it works in compatibility mode.

```bash
make clean
make
```

### Flash to Pico 2

1. Hold **BOOTSEL** button on Pico 2
2. Connect USB cable to Raspberry Pi
3. Release BOOTSEL — Pico appears as USB mass storage
4. Copy firmware:
   ```bash
   sudo mount /dev/sda1 /mnt
   sudo cp out/klipper.uf2 /mnt/
   sudo umount /mnt
   ```
5. Pico reboots automatically

### Identify Serial Port

```bash
ls /dev/serial/by-id/
```

Expected output:
```
usb-Klipper_rp2350_XXXXXXXXXXXX-if00
```

---

## Klipper Configuration

Add to `printer.cfg`:

```ini
# ==============================================================
# CHAMBER HEATER — Pico 2 as secondary MCU
# ==============================================================

[mcu pico]
serial: /dev/serial/by-id/usb-Klipper_rp2350_XXXXXXXXXXXX-if00
# Replace XXXXXXXXXXXX with your actual serial ID

# --------------------------------------------------------------
# Chamber Heater (PID controlled)
# --------------------------------------------------------------
[heater_generic chamber_heater]
heater_pin: pico:gpio15
sensor_type: Generic 3950
sensor_pin: pico:gpio26
min_temp: 0
max_temp: 80
control: pid
# Temporary values — run PID_CALIBRATE after installation:
pid_Kp: 30
pid_Ki: 1.0
pid_Kd: 300

# --------------------------------------------------------------
# Heater verification (relaxed for slow chamber heating)
# --------------------------------------------------------------
[verify_heater chamber_heater]
max_error: 300
check_gain_time: 480
heating_gain: 1

# --------------------------------------------------------------
# Heater Fan (automatic on/off with cooldown)
# --------------------------------------------------------------
[heater_fan chamber_heater_fan]
pin: pico:gpio14
heater: chamber_heater
heater_temp: 35.0
fan_speed: 1.0

# --------------------------------------------------------------
# Additional temperature sensors (optional)
# --------------------------------------------------------------
# Uncomment if you have additional thermistors:

#[temperature_sensor chamber_ambient]
#sensor_type: Generic 3950
#sensor_pin: pico:gpio27
#min_temp: -10
#max_temp: 100

#[temperature_sensor chamber_top]
#sensor_type: Generic 3950
#sensor_pin: pico:gpio28
#min_temp: -10
#max_temp: 100
```

### Configuration Notes

**`[verify_heater chamber_heater]`** — Critical for chamber heaters!
- Default Klipper settings expect hotend-like heating (fast)
- Chamber heaters are slow (heating entire air volume)
- Without these relaxed settings, Klipper will error: "not heating at expected rate"

**`heater: chamber_heater`** — Note: NO prefix!
- Correct: `heater: chamber_heater`
- Wrong: `heater: heater_generic chamber_heater`

**Temperature sensor** — No separate sensor needed!
- `[heater_generic chamber_heater]` automatically shows in Mainsail/Fluidd
- Only add `[temperature_sensor]` if you have additional thermistors in different locations

---

## GCode Macros

Add to `printer.cfg`:

```ini
# --------------------------------------------------------------
# Chamber Heater Macros
# --------------------------------------------------------------

[gcode_macro HEAT_CHAMBER]
description: Heat chamber to target temperature
gcode:
    {% set TEMP = params.TEMP|default(55)|int %}
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET={TEMP}
    M118 Heating chamber to {TEMP}°C...

[gcode_macro COOL_CHAMBER]
description: Turn off chamber heater (fan cools automatically)
gcode:
    SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=0
    M118 Chamber heater off. Fan cooling...

[gcode_macro WAIT_FOR_CHAMBER]
description: Wait until chamber reaches target temperature
gcode:
    {% set TEMP = params.TEMP|default(55)|int %}
    TEMPERATURE_WAIT SENSOR="heater_generic chamber_heater" MINIMUM={TEMP}
    M118 Chamber reached {TEMP}°C!
```

---

## PID Calibration

**After installation, calibrate PID for your specific setup:**

```gcode
PID_CALIBRATE HEATER=chamber_heater TARGET=50
```

Wait for completion (several minutes), then save:

```gcode
SAVE_CONFIG
```

Klipper writes calibrated `pid_Kp`, `pid_Ki`, `pid_Kd` values automatically.

> **Important:** `[heater_generic chamber_heater]` must be in main `printer.cfg`
> (not in an included file), otherwise `SAVE_CONFIG` cannot write PID results.

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
|----------|--------------|-------|
| PLA | 0 (off) | Does not need chamber heat |
| PETG | 0 (off) | Does not need chamber heat |
| ABS | 55-60°C | Eliminates warping |
| ASA | 55-60°C | Similar to ABS |
| Nylon | 60-70°C | Higher temps for best results |
| PC | 65-75°C | Near max safe limit |

---

## Safety Layers

| Layer | Type | Component | Function |
|-------|------|-----------|----------|
| 1 | Software | Klipper PID | Regulates temperature to target |
| 2 | Software | Klipper `max_temp: 80` | Emergency shutdown if >80°C |
| 3 | Hardware | **KSD9700 85°C NC** | Physically disconnects power |
| 4 | Hardware | **Fuse 15A** | Protects against short circuit |
| 5 | Software | `heater_fan` | Auto fan + cooldown |

---

## Tips & Tricks

### Enclosure Insulation

Line the inside of your enclosure panels with **self-adhesive thermal shield mat**
(aluminum + fiber, rated for high temperatures). A 50x100cm sheet covers back and
side panels. Benefits:
- Reflects radiant heat back into chamber
- Reduces heat loss
- Speeds up heating
- Lowers power consumption

### Back Panel Material

If your back panel is foam PVC (Forex/Palight), **replace it** — foam PVC deforms at ~60°C.

Good alternatives:
- **HDF / hardboard** — dense, rigid, handles 60-70°C well
- **Plywood 3-4mm** — similar properties

Line the inside with thermal mat regardless of material.

### Heater Mounting

- Cut an opening in back panel
- Mount **3mm aluminum plate (~150x100mm)** as heater bracket
- Screw NexGen3D bracket to aluminum plate
- Aluminum handles any temperature, provides rigid fireproof mount

### Heater Placement

- **Mount at TOP of back panel** — hot air rises naturally
- **Aim outlet forward/downward** — toward print zone
- **Do NOT mount at bottom** — wastes energy heating under the bed

### NTC Sensor Placement

- Mount thermistor on **opposite side** of chamber from heater
- This measures actual chamber temp, not hot air from heater outlet
- Avoid placing near heated bed or enclosure openings (drafts)

### 3D Printed Cases

Recommended enclosures for your components:

| Component | Model | Link |
|-----------|-------|------|
| Pico 2 | Raspberry Pi Pico Case (for Klipper) | [Printables #226610](https://www.printables.com/model/226610-raspberry-pi-pico-case) |
| MOSFET | Holder for XY-MOS D4184 | [Printables #355368](https://www.printables.com/model/355368-holder-for-xy-mos-d4184-power-mosfet-breakout-modu) |

Print 2× MOSFET holders (one for heater, one for fan).

---

## Installation Checklist

### Preparation
- [ ] Open NexGen3D heater, separate fan/PTC into 4 independent wires
- [ ] Mount KSD9700 **85°C NC** on PTC metal housing
- [ ] Install fuse 15A inline (can be inside heater housing)
- [ ] Mount NTC thermistor in center of chamber (away from heater)

### Electrical Assembly
- [ ] Connect MOSFET #1 (heater): VIN+ ← 24V, VIN- ← GND, VOUT+/VOUT- → to heater cable
- [ ] Connect MOSFET #2 (fan): VIN+ ← 24V, VIN- ← GND, VOUT+/VOUT- → to fan cable
- [ ] Connect MOSFET #1 TRIG → Pico GP15
- [ ] Connect MOSFET #2 TRIG → Pico GP14
- [ ] Connect MOSFETs GND → Pico GND
- [ ] Connect 24V PSU GND → Pico GND (critical!)
- [ ] Wire NTC voltage divider: 3.3V → 4.7kΩ → GP26 → NTC → GND
- [ ] Connect Pico 2 to Raspberry Pi via USB

### Software Setup
- [ ] Flash Klipper firmware on Pico 2
- [ ] Identify serial port: `ls /dev/serial/by-id/`
- [ ] Add configuration to `printer.cfg`
- [ ] `FIRMWARE_RESTART`

### Testing & Calibration
- [ ] Verify temperature reading shows ambient temp
- [ ] Test: `SET_HEATER_TEMPERATURE HEATER=chamber_heater TARGET=40`
- [ ] Verify fan turns on automatically
- [ ] Run: `PID_CALIBRATE HEATER=chamber_heater TARGET=50`
- [ ] Save: `SAVE_CONFIG`
- [ ] Test cooldown: set target to 0, verify fan stays on until chamber <35°C

---

## Troubleshooting

### "Unknown pin chip name 'pico_chamber'"
Your MCU has a different name. Check your `[mcu]` section — use that name in pin references.
Example: if `[mcu pico]` then use `pico:gpio15` not `pico_chamber:gpio15`

### "pin gpioXX used multiple times"
You're referencing the same pin in multiple sections. The heater's sensor is automatically
available — don't create a separate `[temperature_sensor]` with the same pin.

### "Heater not heating at expected rate"
Add `[verify_heater chamber_heater]` section with relaxed timing. Chamber heaters are slow.

### "Option 'heater' must be specified" / wrong heater reference
Use `heater: chamber_heater` (without `heater_generic` prefix).

---

## References

- NexGen3D Chamber Heater: https://www.printables.com/model/863675-nexgen3d-chamber-heater
- AC Chamber Heater Guide (inspiration): https://www.printables.com/model/1211645-how-to-add-115v220v-ac-chamber-heater-to-creality
- Klipper heater_generic: https://www.klipper3d.org/Config_Reference.html#heater_generic
- Klipper heater_fan: https://www.klipper3d.org/Config_Reference.html#heater_fan
- Klipper verify_heater: https://www.klipper3d.org/Config_Reference.html#verify_heater
