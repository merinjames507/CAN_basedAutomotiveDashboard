# CAN Bus Automated Dashboard System

A multi-ECU vehicle network built on **PIC18F4580** microcontrollers communicating over a CAN bus architecture. The system decentralizes sensor processing—broadcasting speed, gear, RPM, engine temperature, indicator status, and collision alerts across two sender nodes—to feed a central node that drives a live LCD dashboard with real-time telemetry and hazard warnings.

---

## Overview

| ECU | Role | Transmits |
|-----|------|-----------|
| ECU1 | Speed & Gear Control | Speed (ADC potentiometer), Gear (keypad), Collision flag |
| ECU2 | Sensor Node | RPM, Engine Temperature, Indicator (L/R/Hazard) |
| ECU3 | Dashboard Display | Receives all messages, renders live LCD display + indicator LEDs |

---

## System Architecture

```
[ ECU1 ] ──┐
           ├──── CAN Bus (500 kbps) ────> [ ECU3 Dashboard LCD ]
[ ECU2 ] ──┘
```

All three nodes operate on a shared CAN bus, where ECU3 passively monitors the network and filters incoming frames by message ID to decode telemetry and update the dashboard in real time.

---

## CAN Message IDs

| Message ID | Sender | Payload |
|------------|--------|---------|
| `0x10` | ECU1 | D2 = Speed (0–100), D3 = Gear (N/1–6/R) |
| `0x30` | ECU2 | D0:D1 = RPM (0–9999, big-endian) |
| `0x40` | ECU2 | D1 = Engine Temperature (°C) |
| `0x50` | ECU2 | D1 = Indicator state (Off/Left/Right/Hazard) |
| `0x20` | ECU1 | Collision alert — triggers collision mode on all nodes |

---

## Hardware

- **Microcontroller:** PIC18F4580 (×3)
- **IDE:** MPLAB X
- **Compiler:** XC8
- **Clock:** 20 MHz crystal
- **CAN pins:** RB2 (TX), RB3 (RX)
- **Display:** Character LCD (16×2) on ECU3
- **Input:** 4×4 Matrix Keypad on ECU1 & ECU2
- **Sensors:** ADC potentiometer (speed on AN4, RPM on AN channel, temperature on separate channel)
- **Indicators:** Left/Right LEDs driven by ECU3 ISR (Timer2-based blink)

---

## Features

- Real-time frame transmission across a shared CAN bus utilizing hardware message filtering by ID.
- 7-gear simulation: N → 1 → 2 → 3 → 4 → 5 → 6 → R (keypad-controlled)
- High-accuracy sensor acquisition converting raw analog signals into scaled RPM and engine temperature metrics.
- ISR-managed turn signals/hazards alongside an emergency system override (0x70 frame) forcing all ECUs into a safe state upon collision.
- ECU3 decodes incoming network frames to display speed, gear, RPM, temperature, and indicator status in real time.

---

## Project Structure

```
├── Ecu1.X/
│   ├── ecu1_main.c   
|   ├── ecu1_sensor.c / ecu1_sensor.h      # Speed/gear logic, CAN TX
│   ├── can.c / can.h                      # CAN init, transmit, receive
│   ├── adc.c / adc.h                      # ADC driver
│   ├── matrix_kp.c / matrix_kp.h          # Keypad driver
│   └── clcd.c / clcd.h                    # LCD driver
│
├── Ecu2.X/
│   ├── ecu2_main.c                        # RPM, temperature, indicator logic
│   ├── ecu2_sensor.c / ecu2_sensor.h      # Sensor read functions
│   ├── msg_id.h                           # Shared CAN message ID definitions
│   ├── can.c / can.h
│   ├── adc.c / adc.h
|   ├── uart.c / uart.h 
│   └── clcd.c / clcd.h
│
├── Ecu3.X/
|   ├── main.c          # Dashboard receive + LCD render + indicator ISR
|   ├── can.c / can.h
|   ├── message_handler.c
|   ├── msg_id.h
|   └── clcd.c / clcd.h
```

---

## Getting Started

### Prerequisites

- [MPLAB X IDE](https://www.microchip.com/mplab/mplab-x-ide)
- [XC8 Compiler](https://www.microchip.com/en-us/tools-resources/develop/mplab-xc-compilers)
- PICkit 3/4 or compatible programmer

### Building & Flashing

1. Open each `.X` folder as a separate MPLAB X project.
2. Build the project (`F11`) — pre-built `.hex` files are in `dist/default/production/`.
3. Flash each `.hex` to its respective PIC18F4580 using your programmer.
4. Connect all three CAN bus lines (CANH, CANL, GND) together through a CAN transceiver (e.g., MCP2551).
5. Terminate the bus with 120Ω resistors at both ends.

### CAN Bus Wiring

```
ECU1 RB2 (TX) ──> MCP2551 TXD        ECU2 RB2 (TX) ──> MCP2551 TXD
ECU1 RB3 (RX) <── MCP2551 RXD        ECU2 RB3 (RX) <── MCP2551 RXD
                     │                                     │
              MCP2551 CANH ─────────── CANH ───────────────┤─── 120Ω ───┐
              MCP2551 CANL ─────────── CANL ───────────────┘─── 120Ω ───┘
                                  (ECU3 same wiring)
```

---

## CAN Configuration

- **Baud rate:** ~500 kbps (20 MHz clock, BRP=4, Prop+PS1+PS2 = 12 TQ)
- **Mode:** ECAN Mode 0 (legacy)
- **Filter 0:** ECU3 disables acceptance filtering (`RXB0CON = 0x20`) to passively capture all network traffic (`0x10` Speed, `0x20` Gear, `0x30` RPM, `0x40` Temp, `0x50` Indicators)

---

## Controls (Keypad)

**ECU1:**
- SW1 — Shift gear up
- SW2 — Shift gear down
- SW3 — Trigger collision mode

**ECU2:**
- SW1 — Left indicator toggle
- SW2 — Right indicator toggle
- SW3 — Hazard lights toggle

---

## LCD Display (ECU3)

```
Line 1:  SPD G RPM  T  ID
Line 2:   75 3 3500 85  L
          ↑  ↑  ↑   ↑   ↑
       Speed │  RPM Temp Indicator
           Gear
```

---

## License

This project is for educational purposes. Feel free to use and adapt with attribution.
