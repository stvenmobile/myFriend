# myFriend 🪄🤖

A kid-friendly, desk-sized companion (“the Friend”) that reacts to a **gesture wand**.  
- **Wand:** Arduino Nano 33 BLE recognizes gestures (wave, circle, flick, tickle) and broadcasts them via BLE.  
- **Friend:** ESP32-S3 (or CYD 2.8") shows animated eyes, moves head/arms via servos, and plays sounds — reacting to wand gestures and touch.

> This README refactors and structures the initial outline you provided. (Source: uploaded README draft.)

---

## Table of Contents
- [Goals](#goals)
- [System Architecture](#system-architecture)
- [Bill of Materials](#bill-of-materials)
- [Electrical & Power](#electrical--power)
- [Comms Protocol (BLE)](#comms-protocol-ble)
- [Getting Started](#getting-started)
- [Repo Layout](#repo-layout)
- [Build Notes & Tips](#build-notes--tips)
- [Roadmap](#roadmap)
- [License](#license)

---

## Goals
- **Delight 9–12 y/o** with immediate, physical feedback (lights, motion, sound).
- **Simple first build**, deep upgrade path (ML gestures, eyes tracking, story modes).
- **Robust power** and wiring so it survives kid handling.

---

## System Architecture

**Wand (Arduino Nano 33 BLE)**  
- Inputs: Onboard IMU (accelerometer/gyro), optional touch button  
- Outputs/Comms: BLE (GATT notifications), optional NeoPixel ring + piezo  
- Power: 1‑cell LiPo + charger + 5V boost  
- Role: Recognize gestures and stream orientation

**Friend (ESP32‑S3 / CYD 2.8" Display)**  
- Inputs: BLE from wand, capacitive “head touch”, optional ToF distance  
- Outputs: Animated eyes/mouth, 2× SG90 (arms), 1× MG90S (head pan), audio via I2S amp  
- Power: 5V 3–4A adapter (separate 5V rail for servos)  
- Role: React to gestures/touch; track wand with eyes/head; play sounds

**Typical Behaviors**  
- **Wake** via head touch → eyes open, yawn sound, arms stretch  
- **Wave** → Friend waves back  
- **Circle** → “Charging spell” animation + sparkle sound  
- **Flick** → Head nod or shake (direction-based)  
- **Tickle** → Laugh sound + quick arm flails  
- **Eyes follow wand** using yaw/pitch data

---

# 🧩 Display & Hardware Configuration

### Hybrid Display Architecture

**myFriend** uses a **two-display system** for modularity and realism:

| Display                                         | Role         | Processor | Typical Content                                          |
| ----------------------------------------------- | ------------ | --------- | -------------------------------------------------------- |
| **ESP32-S3 CYD 3.5" (ESP32-3248S035C)**         | “Face”       | ESP32-S3  | Eyes, expressions, servos, gestures, touch responses     |
| **7" HDMI IPS (connected to Jetson Orin Nano)** | “Info Panel” | Jetson    | Text, math problems, images, calendar, dialogue captions |

This hybrid design keeps the **face responsive and self-contained**, while the **Jetson** handles high-level thinking, speech, and extended visual content.

---

## 🪞 CYD Face (ESP32-3248S035C)

**Module:**

> 3.5" ESP32-3248S035C Capacitive Touch (ST7796 480×320) — Wi-Fi + Bluetooth Development Board
> [OpenHASP Hardware Guide](https://www.openhasp.com/0.7.0/hardware/sunton/esp32-3248s035/)
> [GitHub Reference: ardnew/ESP32-3248S035](https://github.com/ardnew/ESP32-3248S035)
> [Discussion: openHASP #384](https://github.com/HASwitchPlate/openHASP/discussions/384)

### Key Features

* **Display:** ST7796 480×320 TFT LCD (SPI interface)
* **Touch:** GT911 capacitive controller (I²C)
* **Processor:** ESP32-S3 (8 MB PSRAM typical)
* **Connectivity:** Wi-Fi, BLE, USB-C
* **Power Draw:** ~160 mA typical at 5 V (exclude servos)

### Recommended Libraries

* [LovyanGFX](https://github.com/lovyan03/LovyanGFX) or [Arduino_GFX](https://github.com/moononournation/Arduino_GFX)
* [TAMC_GT911](https://github.com/TAMCTec/TAMC_GT911) (touch)
* [Async MQTT Client](https://github.com/marvinroger/async-mqtt-client)
* [NimBLE-Arduino](https://github.com/h2zero/NimBLE-Arduino) (wand BLE link)

### Functional Roles

* Runs the **“face engine”**: eyes, pupils, blinking, expressions, and simple overlays.
* Drives **PCA9685** for servos (arms, head).
* Handles **BLE** link to the wand (gesture + orientation).
* Publishes/consumes **MQTT** topics for Jetson interaction.
* Optional: plays short local SFX via **MAX98357A I²S amp**.

### Electrical Notes

* **Display SPI** and **touch I²C** are already pinned internally.

  * Confirm SDA/SCL (often GPIO 19/20 on this board).
  * Backlight control pin: typically **GPIO 45** (PWM-capable).
* **PCA9685 (servos)**:

  * SDA = GPIO 8, SCL = GPIO 9 (or shared I²C bus).
  * VCC = 3.3 V, V+ = 5 V.
* **Audio (MAX98357A)**: I²S BCLK = GPIO 14, LRCLK = GPIO 13, DIN = GPIO 12.
* **Power:** single 5 V 3 A supply, separate servo rail with ≥ 1000 µF bulk capacitor.

---

## 🧠 Jetson HDMI Info Panel

* **Hardware:** 7" HDMI IPS + USB capacitive touch (800×480 or higher)
* **Role:** Displays extended information, dialogue captions, photos, or learning content.
* **Software:** Chromium kiosk or PyQt app subscribing to MQTT telemetry.
* **Integration:**

  * Publishes `myfriend/friend/cmd/*` messages (e.g., `say`, `anim`, `expression`)
  * Subscribes to `myfriend/friend/wand` and `myfriend/friend/sensors` for real-time interaction feedback.

---

## 🔗 Communication Overview

| Link          | Direction                 | Protocol                                          | Function |
| ------------- | ------------------------- | ------------------------------------------------- | -------- |
| Wand → Face   | BLE (NimBLE)              | Gestures, orientation                             |          |
| Face ↔ Jetson | MQTT (Wi-Fi)              | Commands, telemetry, expressions, speech triggers |          |
| Jetson → HDMI | Direct (HDMI + USB touch) | Visual information and extended UI                |          |

---

## 📂 Firmware Layout Update

```
firmware/
├─ wand/                   # Arduino Nano 33 BLE (gesture IMU)
└─ friend-esp32s3-cyd/     # CYD 3.5" ESP32-S3 (face + servos + BLE + MQTT)
```

---

## 🧱 Build Priorities

1. **CYD Face bring-up**

   * Verify display + touch (LovyanGFX + GT911).
   * Show blinking eyes.
   * Add BLE gesture listener.
   * Add MQTT client (connects to Jetson broker).
2. **PCA9685 servo control** for head/arms.
3. **Audio playback** via MAX98357A.
4. **Jetson integration** (speech + HDMI UI).

---




## Bill of Materials

### Wand
- Arduino **Nano 33 BLE**
- 1S **LiPo** (500–850 mAh)
- **TP4056** LiPo charger (with protection)
- **MT3608** boost (3.7V→5V)
- Optional: WS2812 8‑LED ring, piezo, momentary button

### Friend
- **ESP32‑S3** dev board *or* **ESP32‑S3 CYD 2.8"** (display)
- **PCA9685** 16‑ch servo driver (3.3V logic, 5V servo rail)
- **Servos:** 2× SG90 (arms), 1× **MG90S** (head pan)
- **I2S Amp:** MAX98357A + 3W 4–8Ω speaker (or DFPlayer Mini as alternative)
- Optional: **TTP223** touch sensor; **VL53L0X/L1X** ToF
- **PSU:** 5V 3–4A
- 3D‑printed/simple enclosure + hinges

---

## Electrical & Power

### Wand
- Feed Nano via **5V pin** from MT3608 boost. (VIN expects >4.5V; avoids BLE brownouts.)  
- Switch on battery or boost output; expose the TP4056 port for charging.

### Friend
- **Servos on their own 5V rail** (shared GND with MCU). Add **≥1000µF** bulk cap near servo rail.  
- PCA9685 at 50 Hz; VCC=3.3V, V+=5V.  
- Keep I2S amp wiring short; separate audio GND return if possible.

---

## Comms Protocol (BLE)

**Peripheral (Wand)**  
Custom GATT service with three characteristics (Notify enabled):
- `gesture_id` (uint8) — 0=None, 1=Wake, 2=Wave, 3=Circle, 4=Flick, 5=Tickle, …
- `orientation` — either quaternion (4×float32) or packed yaw/pitch (2×int16, mdeg)
- `battery_pct` (uint8)

Notify `orientation` at **20–30 Hz** during motion; lower rate when idle.

**Central (Friend)**  
- Auto‑connect; subscribe to notifications.  
- Lost link → idle animation; touch remains active.

---

## Getting Started

1. **Clone the repo**
   ```bash
   git clone https://github.com/stvenmobile/myFriend.git
   cd myFriend
   ```

2. **Project structure (create folders)**
   ```bash
   mkdir -p firmware/wand firmware/friend hardware/printed assets/sounds docs
   ```

3. **Toolchains**
   - **Wand:** Arduino IDE (or CLI) with `Arduino_LSM9DS1`, `ArduinoBLE`, `Adafruit_NeoPixel` (optional)
   - **Friend:** PlatformIO or Arduino with `NimBLE-Arduino`, `Adafruit_PWMServoDriver` (PCA9685), display lib for CYD, `ESP32 I2S`

4. **First flash**
   - Flash a simple **BLE beacon** on the wand (sends a fixed gesture_id).  
   - Flash **BLE client + eyes idle** on the friend. Verify connect/notify → change face on gesture.

5. **Wire minimal hardware**
   - Friend: ESP32‑S3 + PCA9685 + one servo + speaker; power from 5V adapter.  
   - Wand: Nano 33 BLE + LiPo + TP4056 + MT3608.

---

## Repo Layout

```
myFriend/
├─ firmware/
│  ├─ wand/          # Nano 33 BLE sketches (BLE service + gesture)
│  └─ friend/        # ESP32-S3/CYD sketches (eyes, servos, audio, BLE client)
├─ hardware/
│  ├─ printed/       # STL/STEP and assembly drawings
│  └─ wiring/        # PDFs/PNGs: pinouts, harness diagrams
├─ assets/
│  ├─ sounds/        # WAV/MP3
│  └─ sprites/       # Eye/mouth sprites if used
├─ docs/             # Design notes, troubleshooting
├─ .gitignore
└─ README.md
```

---

## Build Notes & Tips

- **Servos are noisy:** use a separate 5V rail and star‑grounding; bulk caps tame brownouts.  
- **MG90S** for head pan (better holding torque); SG90 are fine for arms.  
- **Eyes follow wand:** low‑pass yaw for head pan; map yaw/pitch to pupil offsets and clamp.  
- Start **rule‑based gestures**; switch to **Edge Impulse** later for ML recognition.  
- If powering Friend from battery later, plan for **2–3A peaks** or servos will reset the MCU.

---

## Roadmap
- [ ] Minimal BLE pair (gesture→face change)  
- [ ] Eyes/pupil tracking from yaw/pitch  
- [ ] Head pan + one arm motion (PCA9685)  
- [ ] Audio reactions (I2S amp)  
- [ ] Touch wake / tickle behavior  
- [ ] Rule‑based → ML gesture upgrade (Edge Impulse)  
- [ ] Printable enclosure + kid‑safe assembly guide

---

## License
MIT — see [LICENSE](LICENSE).