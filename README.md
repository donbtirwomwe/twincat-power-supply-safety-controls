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

---

## Build & Deploy

1. Open `Test.sln` in TwinCAT XAE (VS 2019 / VS 2022 with TC3 shell).
2. Activate configuration on target (`TwinCAT RT x64`).
3. Build → Download → Run.

---

## Git

Branch: `main`  
Remote: `https://github.com/donbtirwomwe/twincat-power-supply-safety-controls.git`
