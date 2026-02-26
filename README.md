# Gantry System (v10) — STM32 Firmware + Python GUI

Dự án gồm **firmware điều khiển gantry 3 trục (X/Y/Z) trên STM32F103C8T6** và **GUI Python (CustomTkinter)** để điều khiển qua **UART**, đồng thời có **kênh TCP** để nhận lệnh/mode từ **Raspberry Pi** (Grasping / Tracking / Handover).

---

## 1) Kiến trúc tổng quan

**Luồng điều khiển chính (PC → STM32):**
- GUI (Python) gửi lệnh ASCII qua UART: `MOVE`, `JOG`, `HOME`, `STOP`, `TVEL`, `TSTOP`, `STATUS/?`, `ACK`
- STM32 parse lệnh, chạy state machine motion, trả về báo cáo dạng:
  - `STATE = <id> ERR = <mask> POS = x,y,z [SPD = vx,vy,vz]`

**Luồng điều khiển Tracking/Grasping (Raspberry Pi → PC → STM32):**
- Raspberry Pi gửi line text qua TCP (PiLink) tới GUI:
  - `Grasping_Mode`, `Approach_Done`, `Grasp_done`, `Hold_Mode`, `Handover_done`
  - `Move X+`, `Move Y-`, ... hoặc `STOP` (tracking)
- GUI quản lý **“4-button law” state machine** và chuyển đổi sang lệnh UART tương ứng (MOVE/JOG/TVEL/TSTOP/STOP).

---

## 2) Repo structure

```
.
├── gui/                 # Python GUI (CustomTkinter)
│   ├── GUIv10.py         # entry
│   ├── core/             # SerialManager, MotionController, PathEngine, PiLink, state...
│   ├── ui/               # layout/panels/widgets
│   ├── paths/            # ví dụ path JSON (tuỳ chọn)
│   └── fake.py           # fake TCP server mô phỏng Pi (dev/test)
└── firmware/            # STM32CubeIDE project (STM32F103C8T6)
    ├── App/              # protocol, motion, jog, homing, limit, system, comm...
    ├── Core/             # CubeMX generated core
    ├── Drivers/          # HAL/CMSIS
    ├── Gantry Controller v10.ioc
    └── STM32F103C8TX_FLASH.ld
```

## 3) Firmware (STM32F103C8T6)

### 3.1 Build / Flash
- Toolchain khuyến nghị: **STM32CubeIDE**
- Mở project trong `firmware/`
- Build & flash qua ST-LINK.

### 3.2 UART ASCII protocol (tóm tắt)
Các lệnh chính (đúng theo parser trong `App/protocol/protocol.c`):

- `STATUS` hoặc `?` : yêu cầu report
- `ACK` hoặc `RESET` : đồng bộ/ack
- `HOME` : homing
- `STOP` : dừng khẩn (stop motion)
- `TSTOP` : dừng tracking velocity mode
- `MOVE X<val> Y<val> Z<val>` : move absolute (trục nào không có thì bỏ)
  - Ví dụ: `MOVE X120.5 Y10.0`
- `JOG <axis><+|-><step>` : jog bước
  - Ví dụ: `JOG X+1.0`
- `TVEL X<vx> Y<vy> Z<vz>` : tracking velocity (mm/s), trục optional
  - Ví dụ: `TVEL X+40.0 Y-10.0 Z+0.0`

**Report format (GUI parse):**
- `STATE = <int> ERR = <int> POS = x,y,z [SPD = vx,vy,vz]`

---

## 4) GUI (Python / CustomTkinter)

### 4.1 Requirements
- Python 3.10+ (khuyến nghị)
- Packages:
  - `customtkinter`
  - `pyserial`
  - `pillow`
  - `matplotlib` (Plot3D panel có lazy-import)

Cài đặt nhanh:
```bash
pip install customtkinter pyserial pillow matplotlib
```

### 4.2 Run
Chạy entry:
```bash
python GUIv10.py
```

### 4.3 PiLink TCP (Raspberry Pi → GUI)
- GUI mở TCP client tới Pi (mặc định port trong code: `9999`)
- Pi gửi line text:
  - events: `Grasp_done`, `Grasp_fail`, `Handover_done`
  - mode: `Grasping_Mode`, `Approach_Done`, `Hold_Mode`
  - tracking: `Move X+` / `STOP`

> Có file `fake.py` để mô phỏng Pi khi test.

---

## 5) An toàn & giới hạn
- GUI dùng **soft-limits** cho slider: X=350, Y=300, Z=100 (mm) (có thể chỉnh trong code).
- Firmware có limit/homing/stop logic trong `App/limit`, `App/homing`, `App/motion`.
- Khi đóng app: GUI gửi `STOP` và disconnect UART.

---

## 6) Gợi ý nâng cấp
- Thiết kế protocol ASCII gọn, dễ debug (UART) + TCP text cho Pi.
- Kiến trúc GUI modular: `SerialManager` / `MotionController` / `PathEngine` / `PiLink` + UI panels.
- Firmware real-time: motion state machine + tracking velocity mode (TVEL/TSTOP) + limit/homing + error/report.
- Cơ chế an toàn: stop, lock motion khi grasping, soft-limit + limit switch.
