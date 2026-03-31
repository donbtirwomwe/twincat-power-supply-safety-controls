# Honda PTF – Power Supply & Safety Controls

TwinCAT 3 PLC project for the Honda Power Transmission Facility (PTF) commissioning.  
Manages EtherCAT-connected power supply devices (DPUs and BICs) and TwinSAFE safety logic.

---

## Project Structure

```
PowerSupply_SafetyControls/
├── Test/
│   ├── MorpheePanel_PowerSupplyControl/   # Main PLC project (Port 851)
│   │   ├── POU/
│   │   │   ├── Main.TcPOU                 # Task entry point
│   │   │   ├── EthercatManager.TcPOU      # EtherCAT I/O processing & CAN command dispatch
│   │   │   ├── DeviceManager.TcPOU        # Per-device CAN polling (WriteDLC3/WriteDLC4)
│   │   │   ├── GroupManager.TcPOU         # BIC group setpoint coordination
│   │   │   ├── SafetyMgr.TcPOU            # TwinSAFE interface
│   │   │   └── SetpointTest.TcPOU         # Temporary commissioning test injection POU
│   │   ├── GVLs/
│   │   │   ├── GVL.TcGVL                  # Shared global state arrays
│   │   │   ├── GVL_Constant.TcGVL         # Constants
│   │   │   ├── GVL_Persistent.TcGVL       # Persistent variables
│   │   │   └── Safety_Tags.TcGVL          # Safety I/O tags
│   │   └── DUTs/
│   │       ├── panel302.TcDUT
│   │       ├── RxData.TcDUT
│   │       └── TxData.TcDUT
│   └── RH310_SAFETY/                      # TwinSAFE project
│       └── TwinSafeGroup1/
│           └── TwinSafeGroup1.sal         # Safety logic (EL1904, EL1918, EL2904)
```

---

## Device Overview

| Bus   | Device Index | Type | Protocol         | Notes                        |
|-------|-------------|------|------------------|------------------------------|
| Bus 1 | 0 – 3       | DPU  | CAN DLC3/DLC4    | Fixed scaling: VOUT/IOUT/TEMP ×0.1 (F=0.1), VIN F=1 |
| Bus 2 | 4 – 19      | BIC  | CAN DLC3/DLC4    | Dynamic scaling via SCALING_FACTOR nibbles |

---

## Key Components

### EthercatManager (`EthercatManager.TcPOU`)
- **ProcessInputs**: Reads `NumericInput` from EtherCAT, dispatches setpoints/flags to CAN devices.  
  In test mode (`GVL.UseTempBus1Direct = TRUE`), uses `TempBus1*` override arrays for DPU devices 0–3.
- **ApplyCommands**: Rate-limited command write (≥50 ms per device per DPU manual requirement).  
  Latches (`LastVoutSetpoint`, `LastIoutSetpoint`, `LastDirection`, `LastOperationOn`) only update on successful write.
- **ReadLimits**: Reads voltage/current setpoint min/max limits. In test mode, uses `TempBus1VoutSetMin/Max` and `TempBus1IoutSetMin/Max`.
- **UpdateOutputs**: Packs engineering-scaled values into `NumericOutput`.  
  DPU devices 0–3 use fixed scaling; BIC devices use dynamic `VoutScale/VinScale/TempScale` nibbles.

### DeviceManager (`DeviceManager.TcPOU`)
- CAN frame polling per device.
- DPU devices 0–3: poll interval = 5 cycles (50 ms at 10 ms task rate).
- BIC devices: standard poll interval.

### SetpointTest (`SetpointTest.TcPOU`) ⚠️ Temporary
Commissioning test POU for injecting setpoints without the HMI.  
**Remove before production deployment** along with all `UseTempBus1Direct` / `TempBus1*` references.

---

## Safety System (RH310_SAFETY)

TwinSAFE project targeting the RH310 safety controller.  
Alias devices:
- `EL1904` – 4 digital safety inputs
- `EL1918` – 8 digital safety inputs (FW2)
- `EL2904` – 4 digital safety outputs (×3 instances)
- `ErrorAcknowledgement`, `Run` – logical safety signals

---

## Known Commissioning Notes

- **DPU setpoints**: Must send raw value ×100 (e.g., 32 V → write `3200`); `ProcessInputs` divides by 100 before hardware scaling.
- **50 ms minimum command spacing**: Required by DPU CAN protocol; enforced in `ApplyCommands` and `DeviceManager.Poll`.
- **BIC group setpoints**: Group master/slave assignment and `SYSTEM_CONFIG` timing — debugging deferred; see `ProcessInputs` lines ~380–420.

### March 30, 2026 Update

- **BIC limits aligned to hardware (24 V platform)**:
  `MIN_VOLTAGE=19`, `MAX_VOLTAGE=28`, forward current cap `80.0 A`, reverse current cap `64.3 A`.
- **CC-mode voltage headroom added**:
  In BIC forward current mode, voltage command is capped at `26.6 V` to preserve ~5% headroom below 28 V max.
- **CAN command map corrections**:
  Fan reads corrected to `0x0070` (Fan1) and `0x0071` (Fan2), and BIDIRECTIONAL_CONFIG handled as 2-byte DLC4.
- **Grouped BIC control behavior**:
  In grouped non-maintenance mode, current demand is split across active healthy devices in each group and constrained by direction-dependent per-device limits.
- **Group feedback to Morphee without remap**:
  Group masters now publish aggregated group telemetry/status through existing master device channels (Vin/Vout, Iout, temp, fans, fault/status bits).
- **Manual mode/test interaction fix**:
  `SetpointTest` no longer forces individual BIC control unless test `Enable` is TRUE, so Morphee maintenance/non-maintenance mode selection is respected.

### March 31, 2026 Update

- **BIC voltage clamp behavior fixed**:
  In voltage mode, BIC setpoints now always respect hardware voltage bounds (`MIN_VOLTAGE`/`MAX_VOLTAGE`).
  Morphee operator limits (`VoutSetMin`/`VoutSetMax`) are applied only when non-zero, so missing HMI limits no longer collapse setpoint to zero.
- **Absolute limits exported to Morphee for operator UI clamping**:
  `NumericOutput[104..107]` now publish group absolute voltage bounds (x100):
  high word = absolute min, low word = absolute max.
- **Direction-aware absolute current limits exported per group**:
  `NumericOutput[112..115]` now publish group absolute current bounds (x10):
  high word = forward max, low word = reverse max.
  Both values are computed from active healthy device count in each group and BIC forward/reverse per-device hard limits.
- **Group capacity outputs remain at `NumericOutput[108..111]`**:
  Existing capacity mapping is unchanged and still reflects healthy-group capacity.

---

## Build & Deploy

1. Open `Test.sln` in TwinCAT XAE (VS 2019 / VS 2022 with TC3 shell).
2. Activate configuration on target (`TwinCAT RT x64`).
3. Build → Download → Run.

---

## Git

Branch: `main`  
Remote: `https://github.com/donbtirwomwe/twincat-power-supply-safety-controls.git`
