# Automated Storage & Retrieval System

## Overview

An Automated Storage & Retrieval System (AS/RS) with a stacker crane, programmed in Siemens TIA Portal V14 for an S7-1200 CPU 1214C DC/DC/DC and simulated in Factory I/O using the Automated Warehouse scene (v2.5.10) through S7-PLCSIM.

The system manages a **6 × 9 rack with 54 bins**, with Store and Retrieve cycles in Auto mode and a Manual mode for jogging. WinCC HMI provides operator controls, a live rack display, and alarms. The project runs in simulation with S7-PLCSIM without physical PLC hardware.

## Project download and demonstration

- [Download the complete TIA Portal project](Auto_Storage_and_Retrieval_System.zip).
- [Download the demonstration video](https://github.com/Pranesh-Karthik/Auto_Storage_and_Retrieval-System/releases/tag/project-demo) from the release assets and open `Video.Project.mp4` in a video player.
- [View the project file manifest](manifest.json) with file sizes and SHA-256 checksums.

The ZIP contains the full TIA project directory, including its database and supporting HMI files. Extract the ZIP before opening `Auto_Storage_and_Retrieval System.ap14`, and keep the extracted folder and its subfolders together.

The original demonstration video is 430,627,862 bytes (about 431 MB). Its SHA-256 checksum is `bb02239d5b724b0ba158a508063380a9156983e7bfa2811fcc428617a0c421bc`.

## Tools

- **TIA Portal V14** with LAD for OB1 and SCL for function blocks.
- **S7-PLCSIM** for PLC simulation.
- **Factory I/O v2.5.10** with the Automated Warehouse scene and its PLCSIM sync block.
- **WinCC HMI** for visualization and operator controls.

## Architecture

Each function is implemented in its own function block, with a separate function for fork movement.

| Block | Role |
| --- | --- |
| `Mode_manager [FB1]` | Selects Auto or Manual mode from HMI bits. |
| `Manual_mode [FB2]` | Handles jog control and computes and writes Target Position. |
| `Rack_Manager [FB3]` | Handles data and lookups, finds free bins and items, marks bins occupied or empty, and raises `ItemNotFound` and `ItemDuplicate` alarms. It never controls motion. |
| `Homing [FB4]` | Runs the homing sequence. |
| `MotionControl [FB5]` | Controls crane motion. |
| `FB_Store [FB6]` | Runs the Store step sequence. |
| `FB_Retrieve [FB7]` | Runs the Retrieve step sequence. |
| `FB_Auto [FB8]` | Runs one Store and one Retrieve instance, enforces mutual exclusion through `I_OtherBusy`, and selects the active instance's outputs. |
| `MAIN_LOGIC [FB9]` | Selects Manual or Auto mode and routes the selected outputs to the real PLC tags. |
| `FC_ForkMove [FC1]` | Handles fork movement. |

### Data blocks

- `DB_Global` contains HMI flags and shared state.
- `DB_Rack` contains an array of `UDT_Rack` records, each with `Occupied` (`Bool`) and `ItemID` (`Int`).

### Bin numbering

Bins are numbered **1–54**, from right to left within each row. Internally, the code uses row `X` (1–6) and column `Y` (0–9), where `0` represents Home. Conversion to a rack number happens only at the HMI/output boundary.

## Operation

### Store cycle

1. Detect a box.
2. Wait for the operator to enter an item ID and confirm.
3. Find a free bin.
4. Wait for the settle timer.
5. Move the fork left, lift the item, and retract.
6. Travel to the selected bin.
7. Extend the fork into the bin, lower the item, and retract.
8. Commit the bin's occupied state.
9. Return home.

### Retrieve cycle

1. Receive an operator request for an item ID.
2. Find the item's bin.
3. Travel to the bin.
4. Extend the fork, lift the item, and retract.
5. Commit the bin's empty state.
6. Return home.
7. Extend the fork over the unload conveyor, lower the item, and retract.
8. Wait for the item to exit.

Fork movements, lifts, and travel are timer-based. Travel time is calculated from the bin's row and column using per-axis step times plus a margin.

## HMI

The WinCC HMI includes the following screens and views.

- Root screen for Auto/Manual selection.
- Auto screen.
- Store screen.
- Retrieve screen.
- Manual jog screen with seven buttons.
- Live 54-cell rack grid bound to `DB_Rack`.
- Alarm view for item-not-found and duplicate-item errors.

## Challenges and solutions

### 1. Stale function block parameters

FB inputs retain their previous value when they are omitted from a call. Every `Rack_Manager` call therefore sets all four trigger flags explicitly, including calls on abort paths.

### 2. Brief sensor pulses

The entry and unload sensors pulse briefly. Their signals are latched, set by the trigger sensor and cleared by the confirm sensor.

### 3. Motion-complete flags

`Moving X/Z` reads `FALSE` for the first scan after a target change. Arrival is therefore determined by travel-time timers instead of those flags.

### 4. SCL time arithmetic

SCL does not allow `Int × Time` arithmetic. Travel time is computed in `DInt` milliseconds and converted once using `DINT_TO_TIME()`.

### 5. Fork clashes and conveyor momentum

The fork sequence has one state per action, with a timer for each action so the state machine advances only after the movement finishes. A settle timer after the conveyor stops prevents belt momentum from pushing the item into the fork.

### 6. OB1 network order

The Factory I/O sync block must be **Network 1 in OB1**, with `MAIN_LOGIC` in **Network 2**. Reversing this order caused a persistent sensor-reading issue.

### 7. Store/Retrieve concurrency

`I_OtherBusy` prevents Store and Retrieve from starting at the same time.

## How to run

1. Download and extract the project ZIP.
2. Open `Auto_Storage_and_Retrieval System.ap14` in TIA Portal V14.
3. Download the PLC program to S7-PLCSIM and set the simulated CPU to **RUN**.
4. Open Factory I/O and load the **Automated Warehouse** scene.
5. Select **Siemens S7-PLCSIM** as the Factory I/O driver and click **Connect**.
6. Start the HMI, choose **Auto**, and use the Store or Retrieve screen.
7. Spawn boxes manually in Factory I/O.

The separate saved simulation project is omitted at the project owner's request. The Factory I/O scene file is a planned addition to this repository.

## Archive verification

The project ZIP was checked for archive integrity. All 64 packaged files were compared against the original saved project using SHA-256 checksums, recorded in `manifest.json`.
