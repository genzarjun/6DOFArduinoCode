# Robotic Arm Controller – System Overview

A 6-DOF (6 degrees of freedom) robotic arm controlled wirelessly via an Xbox controller. The controller connects to an Android app, which sends commands over Bluetooth Low Energy (BLE) to an Arduino board that drives the arm’s servos.

---

## System Architecture

```
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  Xbox Controller │ ──▶  │  Android App     │ ──▶  │  Arduino R4 WiFi │
│  (Physical Input)│  USB │  (BLE Central)   │ BLE  │  (BLE Peripheral) │
└──────────────────┘      └──────────────────┘      └────────┬─────────┘
                                                              │
                                                              ▼
                                                     ┌──────────────────┐
                                                     │ Adafruit 16-Ch   │
                                                     │ PWM Servo Shield │
                                                     └────────┬─────────┘
                                                              │
                                                              ▼
                                                     ┌──────────────────┐
                                                     │  6 Servo Motors  │
                                                     │  (Robotic Arm)   │
                                                     └──────────────────┘
```

---

## Components

### 1. Xbox Controller (Input Device)

The Xbox controller is the primary input. It connects to the Android device (typically over USB or Bluetooth, depending on setup). The app listens for these buttons:

- **A** – Approach object
- **B** – Grab object
- **X** – Drop at target position
- **Y** – Release object
- **START** – Reset to home position

Each press triggers a preprogrammed motion sequence on the arm.

### 2. Android App (BLE Bridge)

- **Technology:** Android app using BLE GATT
- **Main tasks:**
  1. Connect the Xbox controller and receive button events
  2. Scan for the Arduino BLE peripheral
  3. Forward controller commands to the Arduino over BLE
  4. Provide a UI for connection status and optional manual servo calibration

**BLE setup:**
- **Service UUID:** `12345678-1234-1234-1234-123456789ABC`
- **Characteristic UUID:** `87654321-4321-4321-4321-CBA987654321` (Read/Write)
- **Arduino advertisement name:** `Arduino Robotic Arm`

The app uses wake locks and a persistent notification to keep the BLE connection active when the screen is off.

### 3. Arduino R4 WiFi (Controller)

- **Board:** Arduino UNO R4 WiFi
- **Libraries:** ArduinoBLE, Adafruit PWM Servo Driver
- **Role:** BLE peripheral that receives commands and drives the servo shield

On startup it:
1. Initializes the Adafruit PWM Servo Shield
2. Moves all servos to a home position
3. Starts BLE advertising
4. Processes incoming commands and runs the corresponding motion sequences

### 4. Adafruit 16-Channel PWM Servo Shield

- **Role:** Drives up to 16 servos using I²C (address 0x40)
- **PWM:** 50 Hz, pulse width mapped from 150–600 (roughly 0°–180°)
- **Wiring:** Stacked on top of the Arduino via I²C

### 5. Servo Motors (6DOF Arm)

Six standard servos provide 6 degrees of freedom:

| Servo | Joint            | Shield Port | DOF Description                          |
|-------|------------------|-------------|------------------------------------------|
| 0     | Base             | Port 1      | Rotation around the vertical axis        |
| 1     | Shoulder         | Port 4      | Upper-arm lift (forward/back)            |
| 2     | Elbow            | Port 7      | Lower-arm bend                           |
| 3     | Wrist            | Port 8      | Wrist bend                               |
| 4     | Wrist rotation   | Port 11     | Wrist twist around forearm axis          |
| 5     | Gripper          | Port 15     | Open/close end-effector                  |

Together these form a 6-DOF robotic arm that can reach positions and orientations in 3D space.

---

## How It Works: End-to-End Flow

1. **User presses a button** (e.g., A) on the Xbox controller.
2. **Android** receives it via `onKeyDown()` (e.g. `KEYCODE_BUTTON_A`).
3. **App** sends the corresponding string over BLE (e.g. `"BTN_A"`).
4. **Arduino** receives it via `commandCharacteristic` and calls `processCommand()`.
5. **Arduino** runs the matching sequence (e.g. `executeSequenceApproach()`).
6. **Arduino** uses `moveServoSmooth()` to step servos toward target angles.
7. **Shield** drives each servo via PWM on its mapped port.
8. **Arm** moves through the programmed sequence.

---

## Pick-and-Place Sequences

| Button | Action   | Sequence Summary                                      |
|--------|----------|--------------------------------------------------------|
| **A**  | Approach | Base→0°, Shoulder→0°, Elbow→32°, Wrist→108°, Wrist Rot→85°, Gripper→98° |
| **B**  | Grab     | Gripper→115° (close)                                  |
| **X**  | Drop     | Base→80°, Shoulder→110°, Wrist→97°, Wrist Rot→100°   |
| **Y**  | Release  | Gripper→0° (open)                                    |
| **START** | Reset | All servos back to home position                     |

Typical workflow: **A** (approach) → **B** (grab) → **X** (move to drop) → **Y** (release) → **START** (reset).

---

## BLE Command Protocol

Commands sent from the app to the Arduino:

| Command   | Description                    |
|-----------|--------------------------------|
| `BTN_A` / `A`   | Run approach sequence         |
| `BTN_B` / `B`   | Run grab sequence             |
| `BTN_X` / `X`   | Run drop sequence             |
| `BTN_Y` / `Y`   | Run release sequence          |
| `BTN_START` / `RESET` | Reset to home      |
| `STATUS`        | Print current servo positions |
| `MAP`           | Print servo-to-port mapping   |

---

## Servo Movement

- **Instant:** `moveServoInstant()` – direct move to target angle (e.g. for quick adjustments)
- **Smooth:** `moveServoSmooth()` – moves 1° at a time with `SMOOTH_DELAY` (15 ms) between steps
- **Range:** 0°–180° with safety checks
- **PWM mapping:** 0°→150, 180°→600 (SERVOMIN–SERVOMAX)

---

## Home Position (Initialization)

On power-up, the arm moves to:

| Servo | Angle |
|-------|-------|
| Base | 17° |
| Shoulder | 40° |
| Elbow | 40° |
| Wrist | 126° |
| Wrist Rotation | 56° |
| Gripper | 0° (open) |

---

## File Structure

```
RoboticArmController/
├── ArduinoTest.ino          # Arduino firmware (BLE + servo control)
├── app/
│   └── src/main/
│       ├── java/.../MainActivity.java    # Android BLE + Xbox handling
│       ├── res/layout/activity_main.xml  # App UI
│       └── AndroidManifest.xml           # Bluetooth permissions
└── README.md                # This documentation
```

---

## Summary

This system creates a remote-controlled 6-DOF robotic arm for pick-and-place tasks. The Xbox controller provides an ergonomic interface; the Android app acts as a BLE bridge; and the Arduino plus Adafruit shield execute predefined motion sequences on six servos. The result is a fully wireless control chain from gamepad to mechanical motion.
