# Apollo 11 Spa Controller — ESPHome HMI Replacement

**Replaces the dead OEM controller/HMI on an Apollo 11 spa pack with an ESP32 running ESPHome, using a Home Assistant dashboard as the user interface.**

The Apollo 11 spa pack's original HMI/controller board is dead. The relay box and all power-stage hardware — including the heater, pumps, relays, and manifold-mounted temperature sensors — are still fully functional. This project plugs an ESP32 into the relay box's DB15 port via a terminal adapter, replacing the OEM interface entirely. A Home Assistant dashboard provides thermostat control, pump management, filter scheduling, and real-time temperature monitoring.

---

## Architecture

```
┌─────────────────────┐     DB15 Terminal      ┌──────────────────────────┐
│   ESP32 (uPesy      │     Adapter             │  Apollo 11 Relay Box     │
│   WROOM DevKit)     ├─────────────────────────┤                          │
│                     │  Logic signals (3.3V)   │  Internal relays for:    │
│  ESPHome firmware   │  OEM NTC sensor inputs  │  - Heater coil           │
│  WiFi → Home Asst.  │  3.3V power FROM box    │  - Hi-Limit 1 & 2       │
│                     │                         │  - Pump 1 (Low/High)     │
│  External sensor:   │  External sensor wire   │  - Pump 2               │
│  Spa body NTC ──────┤  (not through DB15)     │  - Circulator            │
│                     │                         │                          │
└─────────────────────┘                         │  OEM sensors (on heater):│
                                                │  - Inlet NTC (DB15/5)    │
                                                │  - Outlet NTC (DB15/6)   │
                                                └──────────────────────────┘
```

**The ESP32 does not drive relays directly.** It sends logic-level signals through the DB15 connector to the Apollo 11 relay box, which handles all power switching. The relay box also provides 3.3 V power to the ESP32 through the same DB15 connection.

The **inlet and outlet temperature sensors are OEM** — they are built into the Apollo 11 relay box and physically attached to the heater manifold. The **spa body temperature sensor is external**, added as part of this project to measure the actual spa water temperature for thermostat control.

---

## Hardware

| Component | Detail |
|---|---|
| Spa Pack | Apollo 11 (OEM relay box intact, OEM HMI dead) |
| MCU | ESP32 uPesy WROOM DevKit |
| Connection | DB15 terminal adapter → Apollo 11 relay box DB15 port |
| Power | 3.3 V supplied by the relay box via DB15 Pin 1 |
| Framework | ESPHome (ESP-IDF) |
| OEM Sensors | Inlet & outlet NTCs (built into relay box, on heater manifold) |
| External Sensor | Spa body NTC (added, measures actual spa water temp) |
| Integration | Home Assistant via encrypted native API |

### DB15 Pinout

| DB15 Pin | GPIO | ESPHome ID | Function |
|----------|------|------------|----------|
| Pin 1 | 3.3V | — | Power from relay box |
| Pin 2 | GPIO25 | `r_HiLim_1` | Heater hi-limit relay 1 |
| Pin 3 | GPIO26 | `r_HiLim_2` | Heater hi-limit relay 2 |
| Pin 4 | GPIO13 | `r_Heater` | Heater coil relay |
| Pin 5 | GPIO34 | `inlet_sensor` | Inlet water temp — OEM (ADC) |
| Pin 6 | GPIO35 | `outlet_sensor` | Outlet water temp — OEM (ADC) |
| Pin 7 | GPIO39 | `spa_sensor` | Spa body water temp — external (ADC) |
| Pin 10 | GPIO18 | `r_P1_low` | Pump 1 — low speed |
| Pin 11 | GPIO19 | `r_P1_high` | Pump 1 — high speed |
| Pin 13 | GPIO32 | `r_Circ` | Circulator pump |
| Pin 15 | GPIO22 | `r_P2_high` | Pump 2 — high speed |

The relay box energises its internal relays when the corresponding DB15 input is driven HIGH (3.3 V). All ESP32 GPIOs are configured as push-pull outputs with `inverted: false`.

### Temperature Sensors

| Sensor | GPIO | Divider Config | Ref Resistor | Location | Origin |
|--------|------|----------------|--------------|----------|--------|
| Inlet | 34 | DOWNSTREAM | 9.0 kΩ | Heater manifold inlet | **OEM** (relay box) |
| Outlet | 35 | DOWNSTREAM | 9.0 kΩ | Heater manifold outlet | **OEM** (relay box) |
| Spa | 39 | UPSTREAM | 10.0 kΩ | Spa body water | **External** (added) |

The inlet and outlet NTCs are part of the Apollo 11 relay box hardware, physically mounted on the heater element/manifold. Their signals pass through the DB15 connector. The spa body sensor is a separate NTC added to this project, wired directly to the ESP32, to provide an accurate reading of the actual spa water temperature for thermostat control.

All three use the same NTC calibration: `10.0 kΩ → 25°C`, `32.665 kΩ → 0°C`, `6.530 kΩ → 35°C`.

Filter chain: **median** (window 5) → **exponential moving average** (α = 0.15).

The spa sensor includes a user-adjustable calibration offset (`Spa Temp Offset` in HA) applied via a template sensor. The raw and corrected readings are both exposed to HA.

---

## Heater Safety Architecture

The heater circuit inside the Apollo 11 relay box is a **three-relay series chain**:

```
[Hi-Limit 1] ──── [Hi-Limit 2] ──── [Heater Coil Relay]
   DB15/2            DB15/3              DB15/4
   GPIO25            GPIO26              GPIO13
```

**All three relays must be energised** for current to flow through the heating element. Opening any single relay breaks the circuit. This provides multiple independent safety layers:

| Layer | Mechanism | Detail |
|-------|-----------|--------|
| **1 — Runtime gate** | Software | Thermostat only fires the heater when `startup_test_complete && !high_temp_alarm && !boot_failed` |
| **2 — Over-temp alarm** | Software | Any sensor > 42°C → heater circuit killed, pumps stay running for cooling, manual reset required |
| **3 — Hardware cutout** | Verified on boot | Startup self-test independently verifies each hi-limit relay can break the circuit |

### Virtual Heater

The `Virtual Heater` template switch is the **only approved way** to turn the heater on. It enforces the correct energisation/de-energisation sequence:

- **ON:** HL1 → HL2 → 500 ms → Pump 1 Low → 500 ms → Heater relay
- **OFF:** Heater relay → 500 ms → Pump 1 Low off → HL2 off → HL1 off

---

## Startup Self-Test

Runs automatically on every boot. The system **will not heat** until all tests pass. The test exploits the physics of stagnant water — with no flow, even a short heater pulse produces a large, easily-detected temperature rise on the OEM outlet NTC (which is physically attached to the heater element inside the relay box).

> **Why not measure inlet-to-outlet delta with flow?** The ADC/NTC sensor chain lacks the resolution to reliably detect the small temperature differential across the heater manifold when water is moving. Stagnant water eliminates this problem entirely.

### Test Sequence (~2 minutes)

#### Sensor Validation (~12s)
Primes the median/EMA filter chains with 5 rounds of manual reads. Verifies inlet and outlet are within the −10°C to 100°C sane range.

#### Step 1 — Hi-Limit 1 ON Only (5s)
- Circuit incomplete (HL2 and Heater relay both OFF)
- ✅ Verify **no temperature change**
- Proves HL1 alone cannot fire the heater

#### Step 2 — Both Hi-Limits ON, Heater OFF (5s)
- Circuit still incomplete (Heater relay OFF)
- ✅ Verify **no temperature change**
- Proves hi-limits alone cannot fire the heater

#### Step 3 — Full Circuit, Heater ON (pulse)
- All three relays energised via DB15
- Heater fires into stagnant water for `heater_pulse_ms` (default 2000 ms, tunable from HA, hard-clamped to 6000 ms max)
- ✅ Verify **temperature rise ≥ 0.5°C** on OEM outlet sensor
- Uses "check once, retry once" — if the first read window doesn't show enough rise, a second window is attempted
- 50°C safety abort at each window boundary
- **Heater relay stays ON** for the break tests

#### Step 4 — Hi-Limit 1 Break Test (~35s)
- Heater relay **still ON**. Hi-Limit 1 **opened**
- **Pump 1 High runs 15s** — flushes cool spa water through the manifold, clearing all residual heat from Step 3
- Pump stops. Water is stagnant and cool. Heater relay still energised
- Wait 10s — if HL1 is not breaking the circuit, the stagnant water heats up immediately
- ✅ Verify **no rise > 0.5°C**
- The pump flush also implicitly verifies water flow

#### Step 5 — Hi-Limit 2 Break Test (~35s)
- Same as Step 4 but for Hi-Limit 2
- **Safe relay transition:** HL2 opens first (while HL1 is still open), then HL1 closes — the circuit is **never complete** during the swap
- Pump flush 15s → stagnant 10s → verify no rise

#### Completion
All outputs OFF → `startup_test_complete = true` → circulator pump starts → HA notification sent.

#### Abort
Any failure → all outputs killed → `boot_failed = true` → HA notification. Use the **"Retry Startup Test"** button in HA after investigating.

---

## Runtime Operation

### Thermostat
ESPHome `thermostat` climate entity (`Spa Temp Controller`) using the calibrated external `spa_temp` sensor. Controls the heater via `virtual_heater` on/off with a triple safety gate.

| Parameter | Value |
|-----------|-------|
| Min heating off time | 60s |
| Min heating run time | 3s |
| Min idle time | 240s |
| Temperature range | 5°C – 40.6°C |

### High-Temperature Alarm
Triggers if **any** sensor (OEM inlet, OEM outlet, or external spa) exceeds 42°C:
- Heater circuit killed (all 3 relay signals dropped)
- Pumps **stay running** for circulation/cooling
- Virtual heater state updated via `publish_state()` (avoids triggering `turn_off_action`, which would kill the pump)
- Manual reset required via **"Reset Hi-Limit"** button in HA

### Filter Cycle
Automatic periodic water filtration using Pump 2 High.

| Parameter | Default | HA Adjustable |
|-----------|---------|---------------|
| Interval | 240 min (4 hr) | ✅ |
| Runtime | 60 min | ✅ |

Manual trigger available via **"Run Filter Cycle"** button. Timer text sensors display remaining cycle time and next scheduled run in `hh:mm` format.

---

## Home Assistant Entities

### Switches
| Entity | Function |
|--------|----------|
| Pump 1 Low | Heater flow pump (interlocked with High) |
| Pump 1 High | Jets (interlocked with Low) |
| Circ Pump | Circulator |
| Pump 2 High | Filter cycle pump |
| Virtual Heater | Safe heater on/off abstraction |

### Sensors
| Entity | Detail |
|--------|--------|
| Inlet Temperature | OEM heater manifold inlet (°C) |
| Outlet Temperature | OEM heater manifold outlet (°C) |
| Spa Temperature Raw | Uncorrected external spa body NTC reading (°C) |
| Spa Temperature | Corrected external spa reading (raw + offset) (°C) |
| Inlet/Outlet Voltage | Raw ADC readings (debug) |
| Inlet/Outlet Resistance | Calculated resistance (debug) |

### Binary Sensors
| Entity | Detail |
|--------|--------|
| Startup Test Complete | `true` after all tests pass |
| High Temperature Alarm | `true` when any sensor > 42°C |
| Hi-Limit Relay Triggered | `true` when alarm has fired (latched until reset) |
| Boot Failed | `true` if startup test failed |

### Buttons
| Entity | Function |
|--------|----------|
| Reset Hi-Limit | Clears alarm flags, allows heating to resume |
| Retry Startup Test | Re-runs the full startup test sequence |
| Run Filter Cycle | Manually triggers a filter cycle |

### Number Inputs (all numeric entry boxes)
| Entity | Range | Default | Purpose |
|--------|-------|---------|---------|
| Heater Test Pulse (ms) | 500–6000 | 2000 | Startup test heater-on duration |
| Spa Temp Offset (°C) | −5.0 to +5.0 | 0.0 | External spa NTC calibration correction |
| Filter Cycle Interval (min) | 60–1440 | 240 | Time between automatic filter runs |
| Filter Cycle Runtime (min) | 1–120 | 60 | Duration of each filter run |

### Text Sensors
| Entity | Detail |
|--------|--------|
| Filter Cycle Remaining | `hh:mm` countdown during active cycle |
| Next Filter Cycle | `hh:mm` until next scheduled cycle |

### Climate
| Entity | Detail |
|--------|--------|
| Spa Temp Controller | Thermostat (heat mode only, 5°C – 40.6°C) |

---

## Globals Reference

| ID | Type | Persists | Purpose |
|----|------|----------|---------|
| `startup_test_complete` | bool | no | Gate for thermostat heating |
| `high_temp_alarm` | bool | no | Over-temperature flag |
| `hi_limit_triggered` | bool | no | Latched alarm flag |
| `boot_testing_active` | bool | no | Suppresses interval sensor polling |
| `boot_failed` | bool | no | Blocks all heating |
| `heater_temp_baseline_inlet` | float | no | Test reference temperature (OEM sensor) |
| `heater_temp_baseline_outlet` | float | no | Test reference temperature (OEM sensor) |
| `heater_pulse_ms` | uint32 | yes | Startup test pulse duration |
| `spa_temp_offset` | float | yes | External spa NTC calibration offset |
| `temp_deadband` | float | no | Rise/fall detection threshold |
| `filter_last_run_time` | uint32 | yes | `millis()` timestamp of last filter run |
| `filter_cycle_enabled` | bool | yes | Automatic filter on/off |
| `filter_interval_minutes` | uint32 | yes | Minutes between filter cycles |
| `filter_runtime_minutes` | uint32 | yes | Minutes per filter cycle |
| `filter_running` | bool | no | Filter cycle currently in progress |

---

## Installation

1. **Prerequisites:** [ESPHome](https://esphome.io/) installed (CLI or HA add-on), Home Assistant instance.

2. **Secrets:** Create a `secrets.yaml` in the same directory:
   ```yaml
   wifi_ssid: "your_ssid"
   wifi_password: "your_password"
   ```

3. **Compile and upload:**
   ```bash
   esphome run spa-controller.yaml
   ```
   Or use the ESPHome dashboard in Home Assistant.

4. **Connect:** Plug the ESP32 into the DB15 terminal adapter, connect to the Apollo 11 relay box DB15 port. The relay box provides 3.3 V power.

5. **First boot:** The startup self-test runs automatically. Watch the logs to verify all 5 steps pass. The circulator pump starts and a notification appears in HA on success.

6. **Tuning:** Adjust `Heater Test Pulse (ms)` and `Spa Temp Offset (°C)` from the HA dashboard as needed. Changes persist across reboots.

---

## License

This project is provided as-is for personal use. No warranty expressed or implied.
