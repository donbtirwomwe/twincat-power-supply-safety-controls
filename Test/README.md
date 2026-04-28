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

## Validation Notes
- Most CRC/hash and generated XML/TMC diffs are expected side effects from safety and mapping edits.
- Recommended post-download checks:
  - TwinSAFE FSoE links are healthy and stable.
  - EAP panel pub/sub values update in both directions for Messages 14-17.
  - No watchdog or communication acknowledgement faults remain active.
