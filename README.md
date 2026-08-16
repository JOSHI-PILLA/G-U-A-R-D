# G.U.A.R.D
### Gas Unit for Air Risk Detection

IoT-based automated gas leak detection, alert, and emergency ventilation system built on the ESP32.

⚠️ **IoT Safety System**

---

## Overview

G.U.A.R.D is an embedded safety system that autonomously detects combustible/toxic gas leaks and responds without requiring human intervention. It continuously samples an MQ-2 gas sensor and, when a calibrated threshold is crossed, triggers a multi-channel response:

- Visual alarm (red/green status LEDs)
- Audible double-beep buzzer pattern
- Automatic actuation of a servo-driven emergency ventilation door

All events and live sensor data sync to a **Blynk IoT dashboard**, giving the user real-time monitoring and two remote control actions — **Manual Reset** and **Manual Override** — from anywhere with an internet connection.

The system prioritizes fail-safe behaviour: once an alarm is cleared via Manual Reset, it intentionally locks out further automatic alarms until a full Manual Override is issued, preventing an unventilated space from silently re-arming itself.

## Objectives

- Continuously monitor ambient gas concentration using an MQ-2 sensor interfaced with the ESP32's ADC
- Automatically trigger a visual and audible alarm when gas levels exceed a safe threshold
- Automatically open an emergency ventilation door (servo-actuated) to assist air circulation during a leak event
- Push real-time sensor data and alarm events to a cloud dashboard (Blynk) for remote visibility
- Provide two distinct remote controls — Manual Reset (silence + close, but stay locked) and Manual Override (full system reset) — to safely manage the alarm lifecycle
- Rate-limit push notifications to avoid alert fatigue during a sustained gas event

## System Architecture

The system follows a **sense → decide → actuate → report** loop, coordinated entirely on the ESP32 with the Blynk library handling cloud sync.

| Stage | Description |
|---|---|
| **Sense** | MQ-2 analog output → ESP32 ADC1 (GPIO 34) |
| **Decide** | Firmware compares reading against `GAS_THRESHOLD`, gated by system lock / manual override state |
| **Actuate** | Red/Green LEDs, buzzer (non-blocking double-beep pattern), SG90 servo (emergency door) |
| **Report** | Live gauge, alarm indicator, and rate-limited push notifications via the Blynk cloud dashboard |

## Hardware

| Component | Function | Interface / Pin |
|---|---|---|
| ESP32 (WROOM-32) | Main controller — sensor reading, logic, WiFi + Blynk cloud sync | — |
| MQ-2 Gas Sensor | Detects combustible/toxic gas concentration (analog output) | GPIO 34 (ADC1, input-only) |
| SG90 Micro Servo | Drives the emergency ventilation door | GPIO 18 (PWM) |
| Active Buzzer | Audible double-beep alarm pattern | GPIO 19 |
| Red LED | Visual alarm indicator | GPIO 21 |
| Green LED | Visual safe-state indicator | GPIO 22 |
| Blynk IoT Platform | Cloud dashboard: live gauge, alarm LED widget, remote controls, push alerts | WiFi (2.4 GHz) |

## Firmware Logic

### Sensor Warm-Up
The MQ-2 requires a heater warm-up period before readings stabilize. On boot, the firmware withholds all alarm logic for 20 seconds (`WARMUP_TIME`).

### Detection & Alarm State
Every 500 ms (`READ_INTERVAL`), the firmware reads the MQ-2 analog value and pushes it live to the Blynk gauge (V0). If the reading exceeds `GAS_THRESHOLD` (1800 on the 12-bit 0–4095 ADC scale) and the system is neither locked nor overridden, it enters the alarm state:

- Red LED ON, Green LED OFF
- Non-blocking double-beep buzzer pattern (150 ms on / 150 ms off, twice, then a pause) driven via `millis()`
- Emergency door servo actuates open (0° → 90°)
- Rate-limited push notification (60-second cooldown) via `Blynk.logEvent`

### Manual Reset (V2) — Silence & Close, Stay Locked
Silences the buzzer, closes the door, and returns LEDs to safe state — but also engages a system lock that suppresses further automatic alarms until a Manual Override is issued. This prevents the system from silently re-arming in a space that may still be unsafe.

### Manual Override (V3) — Full System Reset
Clears the lock and alert flag, silences the buzzer, closes the door, restores LEDs to safe, and resumes live automatic monitoring immediately. The dashboard toggle auto-resets to off, behaving as a momentary action.

### State Summary

| Trigger | Buzzer | Door | Monitoring |
|---|---|---|---|
| Gas > threshold | Double-beep ON | Opens | Active alarm |
| Manual Reset | OFF | Closes | Locked (no re-trigger) |
| Manual Override | OFF | Closes | Resumed (unlocked) |

## Blynk Dashboard

The dashboard mirrors the firmware's virtual pin map one-to-one.

| Widget | Pin | Behaviour |
|---|---|---|
| Gas Level (Gauge) | V0 | Live MQ-2 reading, 0–4095 range, updated every 500 ms |
| Alarm Status (LED) | V1 | Fills when gas exceeds threshold; clears in safe state |
| Manual Reset (Switch) | V2 | Silences buzzer, closes door, locks out auto-alarm until override |
| Manual Override (Switch) | V3 | Full reset — clears lock, resumes automatic monitoring, self-resets to off |

## Testing & Results

- **Sensor warm-up:** No false triggers during the 20-second MQ-2 heater warm-up window
- **Threshold crossing:** Alarm state (LEDs, buzzer, door, dashboard, push alert) triggers correctly once readings pass 1800
- **Notification rate-limiting:** Repeat `gas_alert` events suppressed within the 60-second cooldown during a sustained leak
- **Manual Reset:** Buzzer silences, door closes, system correctly withholds re-triggering while gas remains above threshold
- **Manual Override:** Full state reset and immediate resumption of monitoring, toggle auto-returns to off
- **Dashboard sync:** Device online/offline status, gauge, and alarm indicator reflect real-time firmware state

## Future Scope

- Field calibration of `GAS_THRESHOLD` against a certified reference gas source
- Secondary sensor (e.g. MQ-135 or temperature/smoke) for cross-validated multi-hazard detection
- Battery + charging circuit for backup operation during power outages
- Local buzzer/LED fallback if WiFi/Blynk connectivity drops
- Data logging and trend analysis on the Blynk dashboard for long-term air quality tracking

## Conclusion

G.U.A.R.D demonstrates a complete, low-cost embedded safety system that closes the loop between gas detection and physical emergency response — without requiring a human to be present. Its two-tier manual control design (Reset vs. Override) reflects a deliberate safety-first approach, combined with real-time cloud visibility through Blynk, making it a practical, extensible pattern for IoT-based hazard monitoring in homes, labs, or small industrial spaces.

## Author

**P. Joshi**
B.Tech, Electronics and Communication Engineering
