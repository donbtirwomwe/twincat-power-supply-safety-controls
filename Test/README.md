# Power Supply Safety Controls (Test)

## Recent Changes (2026-04-28)

### TwinSAFE / FSoE Updates
- Updated the custom FSoE alias connection files:
  - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_1.sds
  - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_2.sds
  - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_3.sds
  - RH310_SAFETY/TwinSafeGroup1/Alias Devices/Custom FSoE Connection_4.sds
- Enabled FSoE data mapping on each connection:
  - MapInputs=true
  - MapOutputs=true
- Increased FSoE watchdog time from 100 to 500.
- Enabled communication error acknowledgement mapping (ComErrAck Type=AliasDevice).
- Updated RH310 safety target configuration CRC/project CRC values.

### EtherCAT and Project Mapping Updates
- Added auto-generated type INFODATA_3260476488 for connection info payloads.
- Added "Connection Info Data" PDO (0x1BF9) for Message_14 to Message_17 in EL1918.
- Updated EL1918 SyncManager output offset (001d38... -> 001d40...).
- Updated multiple CanMsgOffset values after remapping.
- Updated link mappings between EL1918 and EL2904 safety routes.
- Updated EAP panel publisher/subscriber routing links.

### EAP Message Mapping (Current)

| Message | EAP -> EL1918 (Tx into ConnectionInputs) | EL1918 -> EAP (Rx from ConnectionOutputs) |
|---|---|---|
| 14 | Panel_300 (Publisher) Pub-Var 36 -> ConnectionInputs Message_14 TxPDO | ConnectionOutputs Message_14 RxPDO -> Panel_300 (Subscriber) Pub-Var 29 |
| 15 | Panel_301LH (Publisher) Pub-Var 38 -> ConnectionInputs Message_15 TxPDO | ConnectionOutputs Message_15 RxPDO -> Panel_301LH (Subscriber) Sub-Var 46 |
| 16 | Panel_301RH (Publisher) Pub-Var 31 -> ConnectionInputs Message_16 TxPDO | ConnectionOutputs Message_16 RxPDO -> Panel_301RH (Subscriber) Sub-Var 12 |
| 17 | ConnectionInputs Message_17 TxPDO -> Panel_PGS (Publisher) Pub-Var 62 | ConnectionOutputs Message_17 RxPDO -> Panel_PGS (Subscriber) Sub-Var 20 |

### Generated/Derived File Updates
- MorpheePanel_PowerSupplyControl/MorpheePanel_PowerSupplyControl.tmc was regenerated.
- Test.tsproj TmcHash and related generated CRC/hash values were updated.

### Local IDE Metadata
- .vs/Test/v15/.suo changed (local Visual Studio user metadata).

## EtherCAT Terminal I/O Usage

### EL2798 (8-Channel Digital Output)
- **Location**: Term 52 (EL2904) - Module 1 (FSOES)
- **Input Source**: `GVL.NumericInput[6]` (8-bit flag register from EtherCAT)
- **Mapping**: Each bit controls one output channel
  - GVL.EL2798_InputFlags: BYTE (receives flags from NumericInput[6])
  - FOR i := 0 TO 7:
    - GVL.EL2798_Output[i] := bit i of EL2798_InputFlags
- **Purpose**: 8-channel discrete on/off control
- **Data Flow**: EtherCAT Input → NumericInput[6] → EL2798_InputFlags → EL2798_Output[0..7]

### EL3174 (4-Channel Analog Input, ±10V)
- **Location**: Term 15 (EL3174)
- **Input Channels**: 4 differential analog inputs (0-10V range)
- **Raw Values**: Mapped to GVL as INT (16-bit)
  - GVL.EL3174_Channel1_Value AT %I*
  - GVL.EL3174_Channel2_Value
  - GVL.EL3174_Channel3_Value
  - GVL.EL3174_Channel4_Value
- **Alarm Detection**: Each channel compared against 5V threshold
  - Threshold formula: 5V = (5.0 * 10) / 32768 (16-bit scaling)
  - Channel 1: **Yellow alarm** (from GT, triggers when > 5V)
  - Channels 2-4: Optional alarms (when > 5V)
  - GVL.EL3174_Channel1_Alarm through Channel4_Alarm (BOOL)
- **Output Mapping**: Alarm flags packed into GVL.NumericOutput[101]
  - Bit 6: Channel 1 Alarm (Yellow - from GT)
  - Bit 7: Channel 2 Alarm (Optional)
  - Bit 8: Channel 3 Alarm (Optional)
  - Bit 9: Channel 4 Alarm (Optional)
- **Data Flow**: EtherCAT Analog Input → Raw INT Values → Threshold Comparison → Alarm Flags → NumericOutput[101]

## Validation Notes
- Most CRC/hash and generated XML/TMC diffs are expected side effects from safety and mapping edits.
- Recommended post-download checks:
  - TwinSAFE FSoE links are healthy and stable.
  - EAP panel pub/sub values update in both directions for Messages 14-17.
  - No watchdog or communication acknowledgement faults remain active.
