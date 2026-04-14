# Pick & Place — PLC Control & Production Monitoring

Industrial automation project: a fully programmed Pick & Place system with real-time production monitoring, built from scratch using **TIA Portal V17**, **PLCSIM (S7-1500)**, and **Factory I/O**.

> Designed and programmed as a personal project to demonstrate end-to-end PLC programming skills — from safety logic to sequence control to production analytics.

![Factory I/O Scene](docs/images/factory_io_scene.png)

---

## The Problem

In manufacturing lines with robotic pick-and-place systems, unplanned downtime is one of the most costly issues. Cycle time degradation — when a machine gradually slows down — often goes unnoticed until a full stop occurs. By the time maintenance is called, production has already been lost.

## The Solution

A complete PLC program that controls a Pick & Place cartesian robot and simultaneously monitors production KPIs in real time. The system measures every cycle, tracks statistical trends (average, min, max), and lays the groundwork for early degradation detection.

---

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│                    TIA Portal V17                    │
│                                                     │
│  ┌─────────┐   ┌─────────────┐   ┌──────────────┐  │
│  │  FC1     │   │  FC2        │   │  FC3         │  │
│  │  Control │   │  Secuencia  │   │  Monitoreo   │  │
│  │  (LAD)   │   │  (SCL)      │   │  (SCL)       │  │
│  └────┬─────┘   └──────┬──────┘   └──────┬───────┘  │
│       │                │                 │          │
│       └────────┬───────┴─────────────────┘          │
│                │                                    │
│         ┌──────┴──────┐                             │
│         │    DB1      │                             │
│         │  Pick&Place │                             │
│         │    Data     │                             │
│         └─────────────┘                             │
└─────────────────┬───────────────────────────────────┘
                  │ PLCSIM
                  │
         ┌────────┴────────┐
         │   Factory I/O   │
         │  Pick & Place   │
         │     Scene       │
         └─────────────────┘
```

---

## Program Structure

### FC1 — Control (Ladder)
Safety and master control logic:
- Start/Stop with self-latching circuit
- Emergency Stop handling (N.C. logic)
- Fault detection and counting
- Reset with E-Stop verification
- Forced output shutdown on stop (defense in depth)

### FC2 — Sequence (SCL)
State machine controlling the pick and place cycle:
- 10 main steps + sub-steps for motion synchronization
- Two-phase motion pattern: wait for axis start → wait for axis end
- Automatic cycle repetition
- Entry/Exit conveyor coordination

| Step | Action | Transition |
|------|--------|------------|
| 0 | Idle | Running = TRUE |
| 1 | Start entry conveyor | Sensor_Entry = TRUE |
| 2 | Stop entry conveyor | Immediate |
| 3/31 | Lower Z axis | Moving_Z start → end |
| 4 | Activate gripper | Item_Detected = TRUE |
| 5/51 | Raise Z axis | Moving_Z start → end |
| 6/61 | Move X to exit | Moving_X start → end |
| 7/71 | Lower Z at exit | Moving_Z start → end |
| 8 | Release + raise Z | Item_Detected = FALSE |
| 9/91 | Return to home | All axes stopped |

### FC3 — Monitoring (SCL)
Real-time production analytics:
- Cycle time measurement using TON timer as stopwatch
- Statistical tracking: last, average, min, max cycle times
- Machine state tracking (Idle / Running / Fault)
- Fault counting
- Piece counting

### DB1 — Pick & Place Data
Centralized data block with all process variables:

| Variable | Type | Purpose |
|----------|------|---------|
| Step | INT | Current sequence step |
| Running | BOOL | Machine operating |
| Fault | BOOL | Active fault flag |
| CycleCount | INT | Completed pieces |
| CycleTime_Last | DINT | Last cycle time (ms) |
| CycleTime_Avg | DINT | Moving average (ms) |
| CycleTime_Min | DINT | Best cycle time (ms) |
| CycleTime_Max | DINT | Worst cycle time (ms) |
| MachineState | INT | 0=Idle, 1=Run, 2=Fault |
| FaultCount | INT | Accumulated faults |

---

## I/O Mapping

### Inputs
| Tag | Address | Description |
|-----|---------|-------------|
| Start | %I0.0 | Start pushbutton |
| Stop | %I0.1 | Stop pushbutton |
| Reset | %I0.2 | Reset pushbutton |
| E_Stop | %I0.3 | Emergency stop (N.C.) |
| Auto_Manual | %I0.4 | Mode selector |
| Sensor_Entry | %I0.5 | Entry conveyor sensor |
| Sensor_Exit | %I0.6 | Exit conveyor sensor |
| Item_Detected | %I0.7 | Gripper item detection |
| Moving_X | %I1.0 | X axis in motion |
| Moving_Z | %I1.1 | Z axis in motion |

### Outputs
| Tag | Address | Description |
|-----|---------|-------------|
| Conveyor_Entry | %Q0.0 | Entry conveyor motor |
| Conveyor_Exit | %Q0.1 | Exit conveyor motor |
| Move_X | %Q0.2 | X axis command |
| Move_Z | %Q0.3 | Z axis command (simple effect) |
| Grab | %Q0.4 | Gripper command |

---

## Screenshots

| Factory I/O Scene | TIA Portal Blocks | Watch Table |
|---|---|---|
| ![Scene](docs/images/factory_io_scene.png) | ![Blocks](docs/images/tia_blocks.png) | ![Watch](docs/images/watch_table.png) |

| FC1 — Ladder Logic | FC2 — SCL Sequence |
|---|---|
| ![FC1](docs/images/fc1_ladder.png) | ![FC2](docs/images/fc2_scl.png) |

---

## Key Technical Decisions

- **Two-phase motion pattern**: Factory I/O axis signals (`Moving_X/Z`) are TRUE during motion, requiring a start-detection sub-step before the completion check — prevents same-scan false transitions.
- **Separate safety layer**: FC1 handles all safety independently from the sequence. If FC2 has a bug, FC1 still forces all outputs OFF on emergency stop.
- **TON as stopwatch**: Used a TON timer with a long PT as a cycle chronometer instead of system clock reads — simpler and reliable in PLCSIM Standard.
- **Exit conveyor always running**: Prevents product accumulation at the output — a real production concern, not just a simulation detail.

---

## Future Work (Phase 2)

The project is designed for expansion with a Python data layer:

- **PLC Communication**: Connect via `python-snap7` (with PLCSIM Advanced) or OPC UA for real-time data acquisition
- **Database**: Store production logs and cycle history in MySQL
- **Dashboard**: Streamlit dashboard showing OEE, cycle time trends, degradation alerts
- **Predictive alerts**: Moving average analysis to detect cycle time degradation before failure

---

## Tools & Technologies

| Layer | Technology |
|-------|------------|
| PLC Programming | TIA Portal V17, SCL, Ladder |
| Simulation | PLCSIM Standard, Factory I/O |
| PLC | Siemens S7-1500 (simulated) |
| Future: Data | Python, snap7, MySQL |
| Future: Dashboard | Streamlit |

---

## Author

**Anderson Uceta** — Mechatronics Engineer  
[LinkedIn](https://www.linkedin.com/in/anderson-joiada-uceta-b%C3%A1ez-691041145/) · [Portfolio](https://andersonuz.github.io) · Santo Domingo, DR
