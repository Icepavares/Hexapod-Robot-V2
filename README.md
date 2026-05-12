# Hexapod V2 — Design Decisions & Problem Analysis

> Design review and upgrade rationale compared to V1.  
> Updated: May 2026

---

## Overview

This document summarizes every major design decision made for Hexapod V2, with reasons for each change compared to V1. The goal was to solve the reliability problems of V1, reduce wiring complexity, improve runtime, and build a platform that supports future AI features.

---

## V1 Summary (old model)

| Component | Spec |
|---|---|
| Main board | Arduino Mega |
| Communication | External Bluetooth module |
| Servo control | Direct GPIO via custom PCB |
| PWM driver | None (direct signal from Arduino) |
| Servos | DS3235 35kg (femur/tibia) + 20kg (coxa) |
| Battery | 3S 11.4V LiPo 4000mAh ~25C |
| Power | 3× step-down modules in parallel → 7.4V |
| Control input | Xbox controller via Bluetooth |
| Runtime | ~15 minutes |
| Body size | 15cm × 15cm circular frame |

---

## Problems Identified in V1

### Problem 1 — Coxa servo jitter

**Symptom:** Small coxa servos exhibited jitter during operation, especially when multiple legs moved simultaneously.

**Root causes identified (two combined):**

1. **No PWM driver chip.** Servo signal wires connected directly through Arduino GPIO and custom PCB traces. Long PCB traces add parasitic capacitance which rounds the PWM pulse edges. At 50Hz with 1–2ms pulses, even small capacitance causes pulse width errors the servo reads as position noise.

2. **Poor ground plane design.** The custom PCB used narrow ground traces instead of a full copper pour ground plane. Every current spike from a servo travelled backward through the shared ground trace and disturbed the voltage reference for every other servo simultaneously. This is especially visible on the coxa servos because they are smaller and react to smaller voltage deviations.

**V2 fix:** Switched to bus servos (see servo section). Eliminates both problems — no PWM signal at all, and the ground plane issue is irrelevant to a serial UART line.

---

### Problem 2 — Short battery runtime (~15 min)

**Root causes identified:**

1. **Low C rating on old pack.** A typical 25C 4000mAh LiPo has a real-world delivery closer to 60% of rated C. Under 20A peak draw, voltage sagged significantly, cutting usable capacity to roughly 60–70% of nominal — shortening runtime from a theoretical 24 minutes to the observed ~15 minutes.

2. **Step-down converter losses.** Buck converters are ~85–90% efficient. 10–15% of battery energy was wasted as heat in the converters before reaching the servos.

3. **Servos running at 7.4V require higher current.** At lower voltage, more current is needed to deliver the same mechanical power (P = V × I).

**V2 fix:** Switched to high-voltage bus servos running directly at 11.1V from the battery. Higher voltage reduces current draw by ~35–40% (same mechanical power, higher voltage). Removing the servo step-down eliminates converter losses. Combined effect: significantly longer runtime from the same or smaller battery.

---

### Problem 3 — Battery too large for the body frame

**Symptom:** The 4000mAh flat LiPo occupied more than half the body frame height, leaving little room for electronics.

**V2 fix:** Two slim 3S 1500mAh 45C packs wired in parallel, mounted one on each side of the body flanking the electronics bay. Total capacity 3000mAh, each pack approximately 68×35×28mm. Center of body frame freed entirely for electronics stack. Weight balanced symmetrically left and right.

---

### Problem 4 — Wiring complexity

**Symptom:** 18 servos each requiring 3 wires (signal, power, ground) = 54 individual wires. Difficult to assemble, debug, and maintain. Messy inside the 15cm frame.

**V2 fix:** Bus servos use a single daisy-chained UART line. All 18 servos share 3 wires total (TX/RX/GND) regardless of servo count. Each servo has a unique ID (1–18) addressed individually in software.

---

### Problem 5 — Limited compute for AI and mobile control

**Symptom:** Arduino Mega has no Wi-Fi, no camera support, no capability to run any AI model. Xbox controller over Bluetooth was the only control option.

**V2 fix:** Raspberry Pi 5 as main board. Built-in Wi-Fi and Bluetooth eliminates external Bluetooth module. WebSocket server over Wi-Fi connects to mobile phone app. Pi Camera v3 via CSI port. TensorFlow Lite / ONNX Runtime for on-device AI inference (object detection, pet recognition). Future AI HAT+ upgrade path available via PCIe port.

---

## V2 Final Component Decisions

### Main Board — Raspberry Pi 5 (2GB)

**Decision:** Pi 5 2GB at ฿2,800.

**Reason over Pi 4:** Cortex-A76 CPU (2.4GHz vs 1.8GHz) gives ~2.5× faster AI inference. Pi 4 runs YOLOv8 nano at ~8fps; Pi 5 runs it at ~20fps — the difference between laggy and live. PCIe 2.0 port enables AI HAT+ upgrade later.

**Reason for 2GB over 4GB:** At ฿1,200 less, 2GB is sufficient for this project when running headless (no desktop environment). Peak RAM usage estimated at ~1.4GB: OS ~400MB, WebSocket server ~100MB, camera pipeline ~200MB, AI model ~400MB, headroom ~300MB. Headless operation keeps OS footprint under 400MB comfortably.

**Reason 4GB not needed now:** The AI HAT+ 2 (future upgrade) has its own dedicated 8GB RAM — it does not use the Pi's system memory. So adding the AI HAT+ later does not require upgrading the Pi RAM.

**Future upgrade path:** Pi 5 2GB → add AI HAT+ 26 TOPS (~฿3,000) when AI features are prioritized. PCIe port must remain physically accessible in body design.

---

### Servos — Hiwonder HX-35H Bus Servo (×18)

**Decision:** Replace all DS3235 PWM servos with HX-35H serial bus servos.

**Reason:** HX-35H runs at 11.1V directly from the 3S battery with no step-down converter required. At 11.1V vs 7.4V, the same mechanical power requires ~35–40% less current (P = V × I). This eliminates the high-current servo buck converter, the PCA9685 PWM driver, the ESP32 motion controller, and 54 individual signal wires.

**Additional benefits:**
- Each servo reports temperature, voltage, and position feedback in real time — allows the Pi to detect overloaded or overheating joints
- Daisy-chain wiring: all 18 servos on one UART line
- Same 35kg torque as DS3235

**Code reuse:** All kinematics logic (IK calculations, gait sequencer, joint angle mapping) from V1 Arduino code is fully reusable. Only the lowest-level servo command function changes — `myServo.write(angle)` becomes `setServoPosition(ID, angle, time)` via UART packet. Everything above that is identical logic.

**Cost:** ~฿600 per servo × 18 = ฿10,800. Higher than DS3235 (~฿300–400 each) but eliminates ESP32, PCA9685, and high-current buck converter costs.

---

### Servo Control — UART Half-Duplex Buffer (SN74HCT241)

**Decision:** One small buffer chip on a minimal PCB, not a full HAT.

**Reason:** Bus servos use half-duplex UART (single shared wire for TX and RX). The Pi's UART is full-duplex (separate pins). The SN74HCT241 combines the Pi's TX and RX into one wire safely, preventing the Pi's TX from fighting its own RX. Without this chip the Pi's UART pins risk damage. Cost ~฿20. The full custom HAT PCB with ESP32 and PCA9685 is no longer needed — replaced by this one small chip and servo connector headers.

**PCB design note:** Ground plane copper pour on both layers is still required. Stitching vias every 5mm. Even though bus servo signal is less sensitive to ground noise than PWM, clean ground is good practice and protects the Pi GPIO.

---

### Power — Single 5V/5A Buck Converter

**Decision:** One buck converter only (11.1V → 5V, 5A) for Pi and logic. No servo step-down.

**Reason:** HX-35H servos run directly from 11.1V battery. The only step-down needed is for the Pi (5V/3A max) plus IMU and OLED (~0.5A). Total logic draw well under 5A. Recommended chip: RT8389GSP or TPS5430. This is the same approach used by Hiwonder's own JetHexa robot.

**V1 comparison:** V1 used three XL4016 step-down modules wired in parallel to try to handle 20A servo current. XL4016 is rated 8A max on spec sheet but reliable real-world use is 5–6A. Three in parallel also caused uneven current sharing. V2 eliminates this entire rail.

---

### Battery — 2× 3S 1500mAh 45C LiPo in Parallel

**Decision:** Two slim flat LiPo packs (3S, 1500mAh, 45C minimum, e.g. Gens Ace or Tattu), wired in parallel.

**Reason:** Each pack is approximately 68×35×28mm and ~124g. Two packs = 3000mAh total, ~248g total, mounted one on each side of the body frame. This frees the center body for electronics — directly solving the V1 space problem where the battery occupied more than half the frame.

**Why 45C minimum:** 45C on 3Ah = 135A peak delivery capability. Robot peak draw with HV servos is ~11A. This enormous headroom means near-zero internal resistance heating under load, resulting in stable voltage throughout discharge — no voltage sag, no jitter, full usable capacity.

**Why 3S not 2S:** 2S (7.4–8.4V) has insufficient headroom for efficient 5V buck conversion as the pack discharges. 3S (9–12.6V) gives comfortable input range across the full discharge curve.

**Why not 18650 cells:** User preference based on experience — 18650 cells have shorter perceived cycle life in this application and the weight of 6 cells (270g+) is comparable to the slim LiPo solution without the mounting flexibility advantage being needed here.

**Estimated runtime:** ~18–20 minutes active walking. Increase to 2× 2200mAh packs (same physical size class) for ~25–28 minutes if needed.

---

### AI Acceleration — Deferred (Pi 5 CPU only for now)

**Decision:** Start with Pi 5 CPU only. Plan PCIe port access in body design for future AI HAT+ addition.

**Reason:** Pi 5 CPU runs YOLOv8 nano at ~20fps — sufficient for basic pet detection and object recognition at launch. Adding AI HAT+ 26 TOPS (~฿3,000) later is a plug-in upgrade requiring no hardware redesign if PCIe port is kept accessible.

**TOPS explained:** TOPS (Tera Operations Per Second) measures how many trillion arithmetic operations a chip can perform per second. AI models consist almost entirely of matrix multiplications — more TOPS means faster inference or ability to run larger models. Pi 5 CPU delivers ~2–3 TOPS. AI HAT+ delivers 26 TOPS. The difference is ~20fps vs 30+fps on the same model, or ability to run a significantly more capable model at the same framerate.

**PCIe explained:** PCIe (Peripheral Component Interconnect Express) is a high-speed dedicated data bus on the Pi 5 for expansion hardware. The AI HAT+ connects via PCIe — not USB — giving it a private high-bandwidth path to the processor with no latency from shared bus contention. Pi 4 has no PCIe port, making AI HAT+ Pi 5 exclusive.

---

### State Indicator — OLED Display (SSD1306 128×64)

**Decision:** Small OLED on the robot head/front, driven via I2C from Pi.

**Reason:** Shows battery voltage, Wi-Fi connection status, current gait mode, and AI detection label without requiring a laptop or phone to inspect robot state. IMU (MPU-6050) connected on same I2C bus for balance and tilt sensing.

---

### Mobile Control

**Decision:** WebSocket server on Pi, connecting to a phone app over Wi-Fi.

**Reason:** Pi 5 has built-in Wi-Fi — no external Bluetooth module needed (eliminated from V1). WebSocket over Wi-Fi gives lower latency than Bluetooth for joystick-style control, works at longer range, and allows the same connection to carry both control commands and camera streaming. Phone replaces Xbox controller.

---

## V1 vs V2 Component Summary

| Component | V1 | V2 | Reason for change |
|---|---|---|---|
| Main board | Arduino Mega | Raspberry Pi 5 2GB | Wi-Fi, AI inference, camera |
| Communication | External BT module | Built-in Wi-Fi | Eliminated external module |
| Servo type | DS3235 PWM 7.4V | HX-35H bus 11.1V | Lower current, no step-down, clean wiring |
| Servo control | Direct GPIO + custom PCB | UART half-duplex + buffer chip | Jitter fix, wiring simplification |
| PWM driver | None (caused jitter) | Not needed | Bus servo has no PWM |
| Motion controller | Arduino Mega itself | Not needed | Pi handles bus servo UART directly |
| Secondary MCU | — | Not needed | Bus servo eliminates need for ESP32 |
| Servo step-down | 3× XL4016 parallel | Not needed | HV servo runs direct from battery |
| Logic step-down | — | 1× 5V/5A buck | Pi power only |
| Battery | 1× 4000mAh 25C 3S | 2× 1500mAh 45C 3S parallel | Size, weight, voltage stability |
| Control input | Xbox controller BT | Phone app Wi-Fi | No external hardware needed |
| Camera | USB camera → PC | Pi Camera v3 CSI | On-board, no PC needed |
| AI | None | TFLite on Pi (+ HAT+ later) | Pet recognition, environment awareness |
| State display | None | OLED 128×64 | Robot state visible without code |
| IMU | None | MPU-6050 | Balance feedback |
| Runtime | ~15 min | ~18–20 min | Higher voltage, no converter losses |
| Wiring complexity | 54 servo wires | 3 wires (daisy chain) | Bus servo architecture |

---

## Architecture Diagram

```
Battery (3S 11.1V)
├── Direct → HX-35H Bus Servos ×18 (daisy-chained, IDs 1–18)
└── Buck 5V/5A
    └── Raspberry Pi 5 (2GB)
        ├── UART half-duplex → SN74HCT241 buffer → servo bus
        ├── CSI → Pi Camera v3
        ├── I2C → MPU-6050 IMU
        ├── I2C → SSD1306 OLED
        ├── Wi-Fi → Mobile phone app (WebSocket)
        └── PCIe [reserved] → AI HAT+ (future)
```

---

## Open Questions / Future Work

- [ ] Finalize body frame redesign for 15cm circular plate with side battery pockets
- [ ] PCIe connector physical access in body design (required for AI HAT+ upgrade)
- [ ] Port V1 Arduino kinematics to Python on Pi (only servo command layer needs rewriting)
- [ ] Design minimal buffer PCB (SN74HCT241 + servo headers + 5V buck)
- [ ] Mobile app design (WebSocket joystick interface + camera feed)
- [ ] Gait tuning for new servo feedback data (temperature/position monitoring)
- [ ] Decide: AI HAT+ 13 TOPS vs 26 TOPS when budget allows

---

*Design review conducted May 2026. All component prices in Thai Baht (฿) at time of writing.*
