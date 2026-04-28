# Recent Updates

Date: 2026-04-28

## Safety (TwinSAFE/FSoE)
- Updated all custom FSoE alias connections:
  - `Custom FSoE Connection_1.sds`
  - `Custom FSoE Connection_2.sds`
  - `Custom FSoE Connection_3.sds`
  - `Custom FSoE Connection_4.sds`
- Enabled FSoE signal mapping (`MapInputs=true`, `MapOutputs=true`) on all four custom connections.
- Increased FSoE watchdog from `100` to `500`.
- Configured communication error acknowledgement (`ComErrAck`) to `Type="AliasDevice"` with input mapping values.
- Updated TwinSAFE target metadata CRC values in `RH310_SAFETY/TargetSystemConfig.xml`.

## EtherCAT / Project Configuration
- Added a new auto-generated info datatype (`INFODATA_3260476488`) used for connection info signals.
- Added `Connection Info Data` PDO (`0x1BF9`) entries for messages 14-17 in the EL1918 area.
- Updated SyncManager value for EL1918 output process data (offset change from `001d38...` to `001d40...`).
- Updated several CAN message offsets (`CanMsgOffset`) to reflect new mapping layout.
- Updated I/O link mappings between EL1918/EL2904 terminals and safety message routes.
- Updated EtherCAT Automation Protocol link mappings for panel publisher/subscriber message routing.

## PLC Build/Generated Artifacts
- Regenerated PLC TMC hash and references:
  - `MorpheePanel_PowerSupplyControl.tmc`
  - `Test.tsproj` (`TmcHash` updated)
- Project/system CRC values changed as expected after mapping edits.

## Local Environment Changes
- `.vs/Test/v15/.suo` changed (user-local Visual Studio metadata).

## Notes
- Most of the XML/TMC hash/CRC updates are generated side effects from configuration and safety mapping changes.
- If required, validate safety communication online after download, especially FSoE mappings and watchdog behavior.
