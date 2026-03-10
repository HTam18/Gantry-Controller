# Gantry System (v10)

**STM32 Firmware + Python GUI for a 3-Axis Gantry Robot**

This project combines:
- **STM32F103C8T6 firmware** for real-time control of a 3-axis gantry system (**X / Y / Z**)
- A **Python GUI built with CustomTkinter** for PC-side control over **UART**
- A **TCP communication channel** for receiving high-level commands and operating modes from a **Raspberry Pi**

The system is designed for motion control workflows such as **Grasping**, **Tracking**, and **Handover**, with a modular architecture on both the firmware and GUI sides.

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Repository Structure](#repository-structure)
- [Firmware](#firmware)
  - [Build and Flash](#build-and-flash)
  - [UART ASCII Protocol](#uart-ascii-protocol)
  - [Status Report Format](#status-report-format)
- [Python GUI](#python-gui)
  - [Requirements](#requirements)
  - [Installation](#installation)
  - [Run](#run)
  - [PiLink TCP Interface](#pilink-tcp-interface)
- [Operating Modes](#operating-modes)
- [Safety and Limits](#safety-and-limits)
- [Development Notes](#development-notes)
- [Future Improvements](#future-improvements)
- [License / Usage Note](#license--usage-note)

---

## Overview

The Gantry System (v10) is a combined embedded and desktop-control project for a **3-axis gantry platform**.

### Main features
- Real-time gantry control on **STM32F103C8T6**
- UART-based ASCII command interface for easy debugging
- Python desktop GUI for manual motion control and monitoring
- TCP-based communication with Raspberry Pi for higher-level coordination
- Motion state machine with support for:
  - Absolute movement
  - Jog motion
  - Homing
  - Emergency stop
  - Tracking velocity mode
- Safety-oriented design with:
  - Soft limits
  - Homing logic
  - Stop logic
  - Limit handling

---

## System Architecture

### 1. Main control path (PC → STM32)
The Python GUI sends ASCII commands to the STM32 firmware over UART, including:

`MOVE`, `JOG`, `HOME`, `STOP`, `TVEL`, `TSTOP`, `STATUS`, `?`, `ACK`

The STM32 firmware:
- Parses incoming commands
- Executes motion control logic through its state machine
- Returns formatted status reports to the GUI

### 2. High-level coordination path (Raspberry Pi → PC → STM32)
The Raspberry Pi sends text commands to the Python GUI over TCP.
These commands may represent:
- Operating mode transitions
- Grasping / handover events
- Tracking direction commands

The GUI acts as the coordinator between Raspberry Pi and STM32 by translating TCP messages into the corresponding UART commands.

### 3. “4-button law” state machine
The GUI manages a higher-level state machine for workflows such as:
- `Grasping_Mode`
- `Approach_Done`
- `Hold_Mode`
- `Handover_done`
- `Grasp_done`

Tracking commands such as `Move X+`, `Move Y-`, or `STOP` are converted into motion commands such as `TVEL`, `TSTOP`, or `STOP`.

---

## Repository Structure

```text
.
├── gui/                 # Python GUI (CustomTkinter)
│   ├── GUIv10.py        # Application entry point
│   ├── core/            # SerialManager, MotionController, PathEngine, PiLink, state...
│   ├── ui/              # Layouts, panels, widgets
│   ├── paths/           # Example path JSON files (optional)
│   └── fake.py          # Fake TCP server for development/testing
└── firmware/            # STM32CubeIDE project (STM32F103C8T6)
    ├── App/             # Protocol, motion, jog, homing, limit, system, comm...
    ├── Core/            # CubeMX-generated core
    ├── Drivers/         # HAL / CMSIS
    ├── Gantry Controller v10.ioc
    └── STM32F103C8TX_FLASH.ld
```

---

## Firmware

### Build and Flash
Recommended toolchain: **STM32CubeIDE**

1. Open the project inside the `firmware/` directory
2. Build the firmware
3. Flash it to the STM32F103C8T6 using **ST-LINK**

---

### UART ASCII Protocol
The firmware parser is implemented in:

`App/protocol/protocol.c`

Supported commands include:

#### Status and synchronization
- `STATUS` or `?` → request a status report
- `ACK` or `RESET` → synchronization / acknowledge

#### Motion control
- `HOME` → execute homing sequence
- `STOP` → emergency stop / stop motion immediately
- `TSTOP` → stop tracking velocity mode

#### Absolute move
- `MOVE X<val> Y<val> Z<val>`
- Axes are optional

Example:
```bash
MOVE X120.5 Y10.0
```

#### Jog move
- `JOG <axis><+|-><step>`

Example:
```bash
JOG X+1.0
```

#### Tracking velocity mode
- `TVEL X<vx> Y<vy> Z<vz>`
- Velocity unit: **mm/s**
- Axes are optional

Example:
```bash
TVEL X+40.0 Y-10.0 Z+0.0
```

---

### Status Report Format
The STM32 sends reports in a format parsed by the GUI:

```text
STATE = <int> ERR = <int> POS = x,y,z [SPD = vx,vy,vz]
```

Where:
- `STATE` = current firmware motion/controller state
- `ERR` = error bitmask
- `POS` = current X/Y/Z position
- `SPD` = optional X/Y/Z speed values

---

## Python GUI

### Requirements
Recommended Python version:
- **Python 3.10+**

Required packages:
- `customtkinter`
- `pyserial`
- `pillow`
- `matplotlib` *(used with lazy import for the 3D plot panel)*

---

### Installation

```bash
pip install customtkinter pyserial pillow matplotlib
```

---

### Run

From the `gui/` directory or project root, run:

```bash
python GUIv10.py
```

---

### PiLink TCP Interface
The GUI opens a TCP client connection to the Raspberry Pi.
The default port is configured in the source code (currently `9999`).

The Raspberry Pi may send text lines such as:

#### Events
- `Grasp_done`
- `Grasp_fail`
- `Handover_done`

#### Modes
- `Grasping_Mode`
- `Approach_Done`
- `Hold_Mode`

#### Tracking commands
- `Move X+`
- `Move X-`
- `Move Y+`
- `Move Y-`
- `STOP`

For development and testing, a simulated Pi server is available:

```bash
python fake.py
```

---

## Operating Modes

The system supports multiple coordinated workflows between the GUI, STM32, and Raspberry Pi.

### Manual motion control
- Absolute positioning via `MOVE`
- Incremental movement via `JOG`
- Homing via `HOME`

### Tracking mode
- Continuous directional movement using `TVEL`
- Controlled stop using `TSTOP` or `STOP`

### Grasping / Handover coordination
- High-level mode/event messages received from Raspberry Pi
- GUI-side state management through the “4-button law” logic
- Translation of high-level events into UART motion commands

---

## Safety and Limits

### GUI soft limits
The GUI currently uses software limits for motion sliders:
- **X = 350 mm**
- **Y = 300 mm**
- **Z = 100 mm**

These values can be adjusted in the source code if needed.

### Firmware-side safety logic
Safety-related logic is implemented in:
- `App/limit`
- `App/homing`
- `App/motion`

This includes:
- Limit handling
- Homing logic
- Stop logic
- Motion-state protection

### Shutdown behavior
When the GUI is closed:
- A `STOP` command is sent to the STM32
- The UART connection is disconnected safely

---

## Development Notes

- The UART protocol is intentionally ASCII-based for easier debugging and testing
- The GUI is structured in a modular way with components such as:
  - `SerialManager`
  - `MotionController`
  - `PathEngine`
  - `PiLink`
  - UI panels/widgets
- The firmware is organized around a real-time motion state machine
- Tracking is implemented using a dedicated velocity mode (`TVEL` / `TSTOP`)

### Current display note
The current GUI code is configured specifically for a **1366×768 display with the taskbar hidden**.

### Raspberry Pi 5 note
The current codebase does **not yet include the extended firmware support for Raspberry Pi 5**.

---

## Future Improvements

Possible next steps for the project:

- Add a more formal command specification document
- Expand error reporting and diagnostics
- Improve GUI scaling for multiple screen resolutions
- Add configuration files for limits, ports, and runtime parameters
- Extend Raspberry Pi integration and protocol coverage
- Add data logging and trajectory recording
- Improve watchdog and fault-recovery handling
- Add automated test utilities for protocol validation

---

## License / Usage Note

This repository is currently presented as a project/documentation codebase.
Add your preferred license here if you plan to publish or distribute it publicly.

Examples:
- MIT License
- Apache 2.0
- GPLv3

---

## Contact / Project Context

This project was developed as a combined embedded-control and desktop-control system for a 3-axis gantry platform using:
- **STM32F103C8T6** for low-level motion control
- **Python CustomTkinter GUI** for operator control and monitoring
- **Raspberry Pi** for high-level coordination modes such as grasping, tracking, and handover

