# Honda PTF - Power Supply & Safety Controls

TwinCAT 3 PLC project for the Honda PTF commissioning system. The solution manages EtherCAT I/O, CAN-connected power supplies, grouped BIC behavior, and TwinSAFE interlocks for the Morphee panel power-supply application.

This README is intended to be both a commissioning manual and a developer maintenance guide for the current project state.

## Recent Updates (2026-04-28)

### TwinSAFE / FSoE

- Updated custom alias-device files:
    - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_1.sds
    - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_2.sds
    - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_3.sds
    - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_4.sds
- Enabled mapping on all four connections:
    - MapInputs=true
    - MapOutputs=true
- Increased watchdog from 100 to 500.
- Configured ComErrAck as AliasDevice mapping.
- Updated RH310 safety target metadata CRC values.

### EtherCAT / Mapping

- Added auto-generated type INFODATA_3260476488 for connection info payload mapping.
- Added Connection Info Data PDO (0x1BF9) entries for Message_14 through Message_17 on EL1918.
- Updated EL1918 SyncManager output offset (001d38... to 001d40...).
- Updated CanMsgOffset values after mapping changes.
- Updated EL1918 and EL2904 safety link mappings.

### EAP Message Routing (Current)

| Message | EAP -> EL1918 (Tx into ConnectionInputs) | EL1918 -> EAP (Rx from ConnectionOutputs) |
|---|---|---|
| 14 | Panel_300 (Publisher) Pub-Var 36 -> ConnectionInputs Message_14 TxPDO | ConnectionOutputs Message_14 RxPDO -> Panel_300 (Subscriber) Pub-Var 29 |
| 15 | Panel_301LH (Publisher) Pub-Var 38 -> ConnectionInputs Message_15 TxPDO | ConnectionOutputs Message_15 RxPDO -> Panel_301LH (Subscriber) Sub-Var 46 |
| 16 | Panel_301RH (Publisher) Pub-Var 31 -> ConnectionInputs Message_16 TxPDO | ConnectionOutputs Message_16 RxPDO -> Panel_301RH (Subscriber) Sub-Var 12 |
| 17 | ConnectionInputs Message_17 TxPDO -> Panel_PGS (Publisher) Pub-Var 62 | ConnectionOutputs Message_17 RxPDO -> Panel_PGS (Subscriber) Sub-Var 20 |

### Generated Artifacts

- Regenerated MorpheePanel_PowerSupplyControl.tmc and updated TmcHash in Test/Test.tsproj.
- Project CRC/hash updates in generated XML are expected side effects.

### Validation Focus

- Verify TwinSAFE FSoE links are healthy after download.
- Verify EAP pub/sub data updates bidirectionally for Message_14 to Message_17.
- Verify no watchdog or communication-acknowledgement faults remain active.

## 1. Purpose

The project provides a single PLC runtime that:

- polls 20 power devices,
- exposes status and commands through EtherCAT process data,
- coordinates grouped BIC devices as shared power-supply banks,
- applies safety-stop behavior before command dispatch,
- publishes status, limits, and aggregated group feedback back to Morphee.

The main PLC project is MorpheePanel_PowerSupplyControl on PLC port 851. The safety project is RH310_SAFETY.

## 2. Repository Layout

```text
PowerSupply_SafetyControls/
|-- README.md
|-- Test.sln
`-- Test/
    |-- Test.tsproj
    |-- MorpheePanel_PowerSupplyControl/
    |   |-- MorpheePanel_PowerSupplyControl.plcproj
    |   |-- DUTs/
    |   |   |-- panel302.TcDUT
    |   |   |-- RxData.TcDUT
    |   |   `-- TxData.TcDUT
    |   |-- GVLs/
    |   |   |-- GVL.TcGVL
    |   |   |-- GVL_Constant.TcGVL
    |   |   |-- GVL_Persistent.TcGVL
    |   |   `-- Safety_Tags.TcGVL
    |   |-- POU/
    |   |   |-- Main.TcPOU
    |   |   |-- DeviceManager.TcPOU
    |   |   |-- EthercatManager.TcPOU
    |   |   |-- GroupManager.TcPOU
    |   |   |-- SafetyMgr.TcPOU
    |   |   `-- SetpointTest.TcPOU
    |   `-- VISUs/
    `-- RH310_SAFETY/
        |-- RH310_SAFETY.splcproj
        `-- TwinSafeGroup1/
            `-- TwinSafeGroup1.sal
```

## 3. System Architecture

### 3.1 Main execution order

Main.TcPOU runs the control flow in this order each PLC cycle:

1. Poll each DeviceManager instance.
2. Run SetpointTest.
3. Refresh group membership by executing GroupManager.
4. Apply safety state in SafetyMgr.UpdateSafety.
5. Refresh group masters in GroupManager.UpdateGroupMasters.
6. Refresh group capacity in GroupManager.UpdateGroupCapacity.
7. Decode commands and send CAN writes in EthercatManager.ProcessInputs.
8. Publish output/status data through EthercatManager.UpdateOutputs.

That order matters. Safety and group state are updated before command dispatch, so ProcessInputs sees the latest ramp-down, shutdown, and group-master information.

### 3.2 Primary function blocks

Main.TcPOU
- Creates 20 DeviceManager instances.
- Performs first-run initialization.
- Owns the scan order for polling, safety, grouping, command dispatch, and output publishing.

DeviceManager.TcPOU
- Handles low-level CAN read/write interaction for a single device.
- Maps logical device indices to Bus1, Bus2, and Bus3 PDO structures.
- Polls readback/status frames.
- Sends DLC3 and DLC4 commands.

EthercatManager.TcPOU
- Converts EtherCAT NumericInput words into per-device control state.
- Handles grouped-BIC request decoding.
- Applies command pacing and write sequencing.
- Sends operation, direction, voltage, current, and configuration commands.
- Publishes telemetry, status, and group-level limits back to NumericOutput.

GroupManager.TcPOU
- Maintains group membership.
- Determines group masters.
- Computes available group capacity from healthy active members.

SafetyMgr.TcPOU
- Translates safety inputs into system state and device shutdown flags.
- Forces stop and shutdown behavior before command issue.

SetpointTest.TcPOU
- Temporary commissioning helper used to inject setpoints without Morphee/HMI.
- Useful during bench testing.
- Not intended as a production control path.

## 4. Device Model

### 4.1 Device index map

| Device range | Bus | Type | Notes |
|---|---|---|---|
| 0..3 | Bus 1 | DPU | Individual devices, fixed scaling behavior |
| 4..11 | Bus 2 | BIC | Group-controlled or individually controlled depending on mode |
| 12..19 | Bus 3 | BIC | Group-controlled or individually controlled depending on mode |

### 4.2 Group map

Grouped BIC control is organized by fixed device ranges:

| Group | Devices | Default NumericInput base |
|---|---|---|
| 0 | 4..7 | 10 |
| 1 | 8..11 | 17 |
| 2 | 12..15 | 24 |
| 3 | 16..19 | 31 |

The default group block size is 7 words per group. The runtime also keeps boot-time GroupMembership defaults aligned to the same fixed mapping so grouped behavior is stable even before dynamic refresh.

### 4.3 Bus mapping summary

| Bus structure | Device indices |
|---|---|
| Bus1_Devices / Bus1_Devices_DLC3 / Bus1_Devices_DLC4 | 0..3 |
| Bus2_Devices / Bus2_Devices_DLC3 / Bus2_Devices_DLC4 | 4..11 |
| Bus3_Devices / Bus3_Devices_DLC3 / Bus3_Devices_DLC4 | 12..19 |

## 5. Core Data Interfaces

### 5.1 EtherCAT input model

GVL.NumericInput is the main inbound control surface from Morphee.

For grouped BIC control, each group consumes a 7-word block:

1. Vout setpoint
2. Iout setpoint
3. flags word
4. Vout minimum
5. Vout maximum
6. Iout minimum
7. Iout maximum

The default bases are stored in GVL.GroupInputBase = [10, 17, 24, 31].

### 5.2 EtherCAT output model

GVL.NumericOutput publishes:

- device telemetry,
- device status flags,
- group master outputs,
- group capacity,
- absolute voltage limits,
- absolute current limits.

The project uses group-master channels to return aggregated grouped-BIC feedback without requiring an external remap in Morphee.

### 5.3 Temporary commissioning paths

The codebase contains temporary override paths for lab and field commissioning:

- GVL.UseTempBus1Direct with TempBus1* arrays for direct DPU injection.
- GVL.UseTempGroupInputs with TempGroupInput for direct grouped-input injection.
- SetpointTest.TcPOU for temporary command injection.

These paths are useful during commissioning but should be treated as temporary support logic rather than final production behavior.

## 6. Control Behavior

### 6.1 DPU behavior

DPUs are devices 0..3. The DPU protocol requires at least 50 ms between commands. The project enforces this in two places:

- DeviceManager polling for DPU devices is slowed to match the protocol requirement.
- EthercatManager ApplyCommands only sends DPU writes when the minimum interval has elapsed.

Important commissioning note: DPU write commands are not echoed back by the device. Successful control must be verified through subsequent read polling, not by expecting the write packet itself to appear as feedback.

### 6.2 BIC grouped behavior

For grouped BIC operation, the project uses shared group decoding and then applies the result to the four devices in each group. The current implementation is designed to keep behavior consistent across all groups rather than relying on each group being wired slightly differently.

Current grouped behavior includes:

- fixed group ranges 4..7, 8..11, 12..15, 16..19,
- safety-aware output and operation gating,
- current split across active healthy members,
- direction-aware forward and reverse current handling,
- repeated current refresh during steady state,
- group-master based feedback publishing.

### 6.3 Voltage mode vs current mode

The code tracks Voltage, Current, Standby, OutputOn, OperationOn, and Direction per device.

Current implementation behavior:

- standby drives effective setpoints to zero,
- safety shutdown forces OutputOn and OperationOn false,
- grouped current requests can activate current mode even when upstream flags are incomplete,
- voltage mode clamps to hardware limits,
- current mode uses direction-aware current limits.

### 6.4 CC mode send sequence

The current code sequence for BIC current-control operation is:

1. Ensure direction command is issued when needed.
2. Send the CC voltage ceiling once when the required voltage changes.
3. Continue sending current commands on subsequent paced opportunities.

Periodic direction refresh no longer clears the remembered voltage/current latches unless the direction actually changed. That prevents the voltage command from repeatedly stealing the pacing slot and starving current refreshes.

### 6.5 Setpoint scaling

Relevant current scaling rules:

- DPU temporary injection values are entered in centi-units. Example: 32.00 V is written as 3200.
- Group current requests from Morphee are handled as total group demand and split per active device.
- BIC engineering scaling is based on runtime scale factors, with current logic using CurrentScale and falling back when needed.

## 7. Safety Behavior

SafetyMgr is authoritative for safe-operation gating.

Observed and implemented behavior:

- SystemState = 0 means normal operation.
- SystemState = 1 means orange stop / controlled limitation.
- SystemState = 2 means red stop.
- SystemState = 3 means E-stop condition.
- SystemRampScale is applied to reduce effective setpoints during stop handling.
- SafetyShutdown per device forces command inhibition.

In practice, when SystemState enters red stop or the ramp scale reaches zero, grouped outputs should not continue commanding active voltage/current operation.

## 8. Build, Download, and Run

### 8.1 Tooling

Expected environment:

- TwinCAT XAE installed in Visual Studio,
- access to the TwinCAT runtime target,
- EtherCAT configuration matching the deployed hardware,
- TwinSAFE configuration available for RH310.

### 8.2 Standard workflow

1. Open Test.sln in TwinCAT XAE.
2. Load the Test.tsproj project.
3. Verify that the PLC project MorpheePanel_PowerSupplyControl and the RH310_SAFETY project both load correctly.
4. Select the correct target runtime.
5. Activate the configuration if the target mapping changed.
6. Build the PLC project.
7. Download to the runtime.
8. Start the PLC in Run mode.
9. Confirm EtherCAT alive status and device polling.

### 8.3 First checks after download

Verify these items first:

1. GVL.EthercatAlive becomes true.
2. Device polling updates Vin, Vout, Iout, and flags.
3. GroupMembership remains aligned to the expected fixed BIC ranges.
4. Safety state is normal before command tests.
5. NumericInput values are landing on the expected group blocks.

## 9. Commissioning Guide

### 9.1 Baseline startup checklist

Before any command test:

1. Verify the correct target is selected.
2. Confirm no active safety trip is holding the system in shutdown.
3. Confirm polling is healthy and no widespread CommFault is present.
4. Confirm the relevant device or group has OutputOn and OperationOn enabled.
5. Confirm the requested mode matches the intended test.

### 9.2 Grouped BIC commissioning steps

Recommended approach:

1. Start with one group and verify all four device indices in that group.
2. Confirm the group master is sensible and the slaves are healthy.
3. Apply a small voltage-mode request first and verify the group accepts it.
4. Switch to current mode and verify the send order: direction, voltage ceiling if changed, then current.
5. Verify the total requested current is split across active healthy members.
6. Check NumericOutput feedback from the group-master channels.

### 9.3 DPU commissioning steps

1. Use conservative command rates.
2. Do not expect immediate echo of write commands.
3. Verify readback values through polling.
4. If a fault clears, recover by toggling operation off then on after the condition is removed.

### 9.4 Using SetpointTest and temporary overrides

Use these only when direct HMI/Morphee control is unavailable or when isolating a mapping issue.

Recommended discipline:

1. Enable the temporary path intentionally.
2. Record which override path is active.
3. Disable it after the test.
4. Do not leave temporary overrides enabled for production commissioning handoff.

## 10. Troubleshooting

### 10.1 POU or PLC folders do not appear in TwinCAT

Check the solution and project metadata first. This repository previously required repair of solution and PLC linkage metadata before the PLC tree appeared correctly in the IDE.

### 10.2 Voltage commands work but current does not

Check the following in order:

1. Verify safety is not forcing standby or shutdown.
2. Verify OutputOn and OperationOn are both true.
3. Verify the device is actually in current mode.
4. Verify direction is correct for the intended power-flow path.
5. Verify the pacing slot is not being consumed by repeated direction or voltage prerequisites.
6. Verify current feedback through polling, not by assuming the write packet was accepted.

### 10.3 Group 3 works but other groups do not

Typical suspects:

1. Wrong NumericInput base mapping.
2. Group membership or group-master mismatch.
3. Health differences between groups.
4. Safety state affecting one subset of devices.

This project now prefers fixed group ranges and shared group decode logic to reduce that class of inconsistent behavior.

### 10.4 PLC restart makes setpoints work temporarily

That symptom usually points to state-latch, configuration, direction, or device-side acceptance timing rather than a missing PDO write. Review:

- direction state,
- system configuration state,
- whether voltage/current latches were reset correctly,
- whether current commands resume after the one-time voltage prerequisite.

### 10.5 FSoE master-slave unknown command in DATA state and watchdog faults

Observed error example (one case):

- Term 7 (EL6910): (0x2082) The 3. connection has received an unknown FSoE-Cmd in state DATA.
- Intermittent watchdog faults at the same time.

Scope:

- This can affect master-slave safety communications across multiple FSoE links, not only Term 7.
- Bad commands or watchdog loss can force the CX8110 into fail-safe behavior.

Likely cause in this project context:

- EAP traffic uses UDP and is sensitive to network noise and congestion.
- When EAP traffic shares a busy/noisy switch segment, packet quality/timing can degrade and TwinSAFE/FSoE communication can become unstable.

Recommended fix (validated during commissioning):

1. Keep EAP devices on their own dedicated switch segment.
2. Avoid mixing unrelated high-traffic or noisy devices on the same switch path.
3. Retest TwinSAFE link health and watchdog status after isolating the network path.
4. If fail-safe was triggered on the CX8110, perform a reset by cycling power.

Expected result:

- Unknown FSoE-Cmd DATA-state errors clear across affected links.
- Watchdog faults stop recurring under normal load.
- CX8110 remains in normal operation without repeated fail-safe resets.

## 11. Current Version Notes

### March 30-31, 2026

- BIC hardware limits aligned for the 24 V platform.
- Group current demand split across active healthy devices.
- Group feedback published through master channels.
- Voltage clamp behavior fixed so missing operator limits do not collapse setpoints to zero.
- Absolute group voltage and current limits exported to NumericOutput.

### April 1, 2026

- Confirmed no dedicated DPU fault-reset CAN command.
- Retained recovery strategy of clearing the fault condition and toggling operation off then on.

### April 14, 2026

- Restored DPU pacing to the documented 50 ms minimum.
- Re-aligned grouped BIC handling to fixed device ranges.
- Added periodic grouped-BIC current refresh so steady-state current commands continue to be resent.

### April 15, 2026

- Grouped request decoding is shared across all BIC groups in EthercatManager.ProcessInputs.
- Safety shutdown now clamps OutputOn and OperationOn before command dispatch.
- Nonzero setpoint writes require active output and operation state.
- Current-mode behavior now favors sending the CC voltage prerequisite only when it changes and otherwise uses paced send opportunities for current commands.
- Periodic direction refresh no longer resets remembered voltage/current setpoints unless direction actually changes.
- DeviceManager no longer overwrites logical setpoint state with encoded CAN payload values during DLC4 writes.

## 12. Git and Change Control

Current working branch is main.

Recommended source-control practice for this project:

1. Commit PLC source, solution metadata, and generated artifacts only when they are needed for reproducible TwinCAT builds.
2. Avoid committing user-specific IDE state when possible.
3. Record commissioning behavior changes in this README when command sequencing or safety semantics change.

Repository remote:

- https://github.com/donbtirwomwe/twincat-power-supply-safety-controls.git

## 13. Files Most Relevant to Maintenance

- Test/MorpheePanel_PowerSupplyControl/POU/Main.TcPOU
- Test/MorpheePanel_PowerSupplyControl/POU/EthercatManager.TcPOU
- Test/MorpheePanel_PowerSupplyControl/POU/DeviceManager.TcPOU
- Test/MorpheePanel_PowerSupplyControl/POU/GroupManager.TcPOU
- Test/MorpheePanel_PowerSupplyControl/POU/SafetyMgr.TcPOU
- Test/MorpheePanel_PowerSupplyControl/GVLs/GVL.TcGVL

## 14. Open Items

Items still requiring hardware validation:

1. Reverse-direction current acceptance in all grouped BIC scenarios.
2. Whether CurrentScale is always populated correctly from all devices online.
3. Full field validation that restart-only acceptance symptoms are eliminated by the current direction/latch sequencing changes.
