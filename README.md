# ArduPilot Custom Configuration — MicoAir H743 V2

Port of all PX4 custom modifications to ArduPilot (ArduCopter 4.7.x).
Flash the stock **MicoAir743v2** ArduCopter firmware from Mission Planner,
then apply the parameter file `micoair_h743v2_custom.param`.
No hardware changes are needed — same wiring as PX4 custom build.

---

## Summary of Ported Changes

| PX4 Change | ArduPilot Equivalent |
|---|---|
| UART5 disabled (PB5 freed) | PB5 not assigned to UART5_RX in hwdef |
| Safety button on PB5 (PULLUP, active-low) | `PB5 SAFETY_IN INPUT PULLUP` in hwdef |
| 3s hold to disarm safety, 1s to re-arm | `BRD_SAFETY_DEFLT=1`, hold time via `BRD_SAFETY_MASK` |
| Safety is a toggle switch | ArduPilot: first press disarms safety, second press re-arms |
| TFmini enabled | `SERIAL3_PROTOCOL=9`, `RNGFND1_TYPE=20` |
| GPS on dedicated UART | `SERIAL1_PROTOCOL=5` (GPS) |
| M9N rate fixed to 8Hz (125ms) | `GPS1_RATE_MS=125` |
| DShot disabled (no need) | Not applicable — ArduPilot uses DShot by default, disable via `MOT_PWM_TYPE` |

---

## 1. Firmware Installation

1. Open **Mission Planner → Install Firmware**
2. Select **MicoAir743v2** from the board list
3. Flash **ArduCopter 4.7.x**
4. After reboot, go to **Config → Full Parameter List**
5. Load `micoair_h743v2_custom.param` (click "Load from file")
6. Write all parameters and reboot

---

## 2. Hardware Wiring (unchanged from PX4 setup)

| Function | Pin | UART | Notes |
|---|---|---|---|
| GPS (M9N) | PA9/PA10 | UART1/SERIAL1 | 115200 baud |
| OSD (MSP) | PA2/PA3 | UART2/SERIAL2 | DisplayPort |
| TFmini | PD8/PD9 | UART3/SERIAL3 | 115200 baud |
| MAVLink2 | PA0/PA1 | UART4/SERIAL4 | Companion/GCS |
| **Safety switch** | **PB5** | **GPIO** | **GND to trigger (active-low)** |
| RC Input | PC7 | UART6/SERIAL6 | SBUS/CRSF/etc. |
| ESC Telem | PE7 | UART7/SERIAL7 | RX only |
| Bluetooth | PE0/PE1 | UART8/SERIAL8 | 115200 baud |

> **⚠️ NOTE on PB5:** In the stock ArduPilot hwdef, PB5 is `UART5_RX`.
> The custom hwdef (`hwdef_custom.dat`) removes it from UART5 and defines it as `SAFETY_IN INPUT PULLUP`.
> This means you need to **rebuild the firmware** from source using `hwdef_custom.dat` to get safety button support.
> If you flash the stock firmware, the safety button won't work (PB5 will be UART5_RX).
> **However, all other parameters still apply to the stock firmware.**

---

## 3. Mission Planner Parameters

These parameters are in `micoair_h743v2_custom.param`.
Apply them via Mission Planner → Config → Full Parameter List → Load from file.

### GPS (M9N CUAV Neo 3)
| Parameter | Value | Reason |
|---|---|---|
| `GPS1_TYPE` | 1 | u-blox auto |
| `GPS1_RATE_MS` | 125 | **8Hz — fixes M9N 16-satellite hardware cap** |
| `SERIAL1_PROTOCOL` | 5 | GPS |
| `SERIAL1_BAUD` | 115 | 115200 baud |

### TFmini Rangefinder
| Parameter | Value | Reason |
|---|---|---|
| `SERIAL3_PROTOCOL` | 9 | Rangefinder |
| `SERIAL3_BAUD` | 115 | 115200 baud |
| `RNGFND1_TYPE` | 20 | Benewake TFmini |
| `RNGFND1_MIN_CM` | 30 | Minimum range 30cm |
| `RNGFND1_MAX_CM` | 1200 | Maximum range 12m |
| `RNGFND1_ORIENT` | 25 | Downward-facing |

### Safety Button
| Parameter | Value | Reason |
|---|---|---|
| `BRD_SAFETY_DEFLT` | 1 | Safety enabled at boot |
| `BRD_SAFETYOPTION` | 3 | Require safety before arm + allow toggle |

> **Note:** ArduPilot's safety button behavior is: press once to disable safety (allow arming), press again to re-enable safety. This matches the PX4 toggle behavior.
> Hold time for ArduPilot safety button is ~2 seconds by default (not configurable without code change, unlike our PX4 3s/1s custom).

### Motor Output (replaces PX4 DSHOT disable)
| Parameter | Value | Reason |
|---|---|---|
| `MOT_PWM_TYPE` | 6 | DShot300 (or set to 0 for standard PWM) |

---

## 4. Building Custom Firmware (for Safety Button on PB5)

To get the safety button working, you must build ArduCopter with `hwdef_custom.dat` and apply the safety switch inversion patch.

```bash
cd ~/ardupilot
# Apply the active-low safety switch inversion patch
git apply /home/verya/Ardupilot-Custom/micoair_h743v2_safety_invert.patch

# Copy the custom hardware definition
cp /home/verya/Ardupilot-Custom/hwdef_custom.dat libraries/AP_HAL_ChibiOS/hwdef/MicoAir743v2/hwdef.dat

# Configure and build
./waf configure --board MicoAir743v2
./waf copter
# Output: build/MicoAir743v2/bin/arducopter.apj
```

Flash `arducopter.apj` via Mission Planner (Custom firmware → select file).

---

## 5. GPS Rate Fix Explanation

The u-blox M9N limits itself to **16 satellites** when running at ≥10Hz.
ArduPilot's default for the M9N is **5Hz** (`GPS_MAX_RATE_MS = 200ms`) — which already avoids the 16-sat cap.

**However**, the parameter `GPS1_RATE_MS` can be set freely. Setting it to **125ms (8Hz)** gives you:
- Faster position updates than stock 5Hz
- Still below the 10Hz threshold that triggers the 16-sat cap
- More satellites tracked than at 10Hz

This matches the fix applied to PX4 (`rate_meas = 125`).

---

## 6. What Doesn't Need Code Changes (Parameter Only)

These PX4 changes map directly to Mission Planner parameters with **no firmware rebuild**:

- ✅ GPS type, baud, rate
- ✅ TFmini rangefinder on any UART
- ✅ MAVLink port assignments
- ✅ Safety button behavior (`BRD_SAFETY*`)
- ✅ Motor output type (DShot vs PWM)
- ✅ RC input protocol

## 7. What Requires Firmware Rebuild

- ⚠️ **Safety button on PB5**: needs `hwdef_custom.dat` to remove PB5 from UART5_RX and define it as `SAFETY_IN`

