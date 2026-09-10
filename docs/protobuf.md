# Protobuf Payloads & Field Maps

Field numbers extracted from generated `*_FIELD_NUMBER` constants in smali (`FitnessProto`, `FileProto` — classes14.dex). Status: `CONFIRMED` (static); wire-level verification not performed (no BLE-layer capture).

---

## Request Types

### `FitnessProto.TimeRangeRequest` (HR stats, Sport stats, Sport record list, Fitness)

| field | # | type | notes |
|-------|---|------|-------|
| `startTimestamp` | 1 | int32 | epoch seconds |
| `endTimestamp` | 2 | int32 | epoch seconds |
| `supportTlv` | 3 | int32 | set to 2 in `SportRecordListFetcher`; default 0 elsewhere |

### `FitnessProto.TypeRequest`

| field | # | type |
|-------|---|------|
| `type` | 1 | int32 |

Used by `SportStatDataFetcher` in the "today only" branch (`setType(1)`).

### `FitnessProto.StringRequest`

| field | # | type |
|-------|---|------|
| `value` | 1 | string |

Used by `SportRecordFetcher` (`setValue(sportId)`).

### `FileProto.FileRequest` (file transfer channel)

| field | # | type | notes |
|-------|---|------|-------|
| `name` | 1 | string | file name |
| `uri` | 2 | string | URI |
| `serviceId` | 3 | int32 | set to 4 for sport-record sub-files |
| `state` | 4 | int32 | — |

---

## Response Types

### `FitnessProto.HeartRateStatData`

| field | # | type | notes |
|-------|---|------|-------|
| `data` | 1 | repeated `HeartRateStat` | |

Also present: `FitnessProtoV2.HeartRateStatDataV2` (V2 device branch; `parseFrom` attested).

### `FitnessProto.HeartRateStat`

| field | # | type |
|-------|---|------|
| `timestamp` | 1 | int32 |
| `avgWalkHeartRate` | 2 | int32 |
| `restHeartRate` | 3 | int32 |
| `sleepHeartRate` | 4 | int32 |

### `FitnessProto.SportStatData`

| field | # | type |
|-------|---|------|
| `totalCalorie` | 1 | int32 |
| `totalStep` | 2 | int32 |
| `totalDistance` | 3 | int32 |
| `totalFloor` | 4 | int32 |
| `totalExercise` | 5 | int32 |
| `timestamp` | 6 | int32 |
| `activityCount` | 7 | int32 |
| `staticCalorie` | 8 | int32 |
| `sedentaryCount` | 9 | int32 |
| `sedentaryTime` | 10 | int32 |
| `totalAmountOfExercise` | 11 | int32 |

### `FitnessProto.SportStatList`

| field | # | type |
|-------|---|------|
| `data` | 1 | repeated `SportStatData` |

### `FitnessProto.FileNameData` (sport record list response)

| field | # | type | notes |
|-------|---|------|-------|
| `startTime` | 1 | int32 | |
| `endTime` | 2 | int32 | |
| `fileName` | 3 | repeated string | |
| `moreData` | 4 | int32 | pagination flag |
| `fileType` | 5 | bytes | |

---

## Observed Data Flows (callable-level)

### (0x5, 0x35) HR stats — request/response
```
HRStatDataFetcher.startFetch() [c12]
 → TimeRangeDataFetcher.getTimeRangeRequest(0xe) → TimeRangeRequest{1,2}
 → toByteArray() → new MessageEvent(0x05, 0x35, data)
 → BTClient.sendMsg(event, callback)
 ← callback → onHeartRateStatData([B)
 ← FitnessProto.HeartRateStatData.parseFrom   (or HeartRateStatDataV2)
```

### (0x5, 0x36) Sport stats
```
SportStatDataFetcher.startFetch() [c12]
 branch A (today only): dynamic (sid, cid) from device capability + TypeRequest{type=1}
 branch B (range): TimeRangeRequest(0xb) → MessageEvent(0x05, 0x36)
 ← SportStatData.parseFrom / SportStatList.parseFrom
```

### (0x4, 0x01) Sport record list
```
SportRecordListFetcher.fetchFileIndex() → TimeRangeRequest(supportTlv=2)
 → MessageEvent(0x04, 0x01) → sendMsg
 ← onFileIndexResult → FitnessProto.FileNameData.parseFrom
```

### (0x4, 0x02) Sport record (single)
```
SportRecordFetcher.requestRptFile() → StringRequest{value=sportId}
 → MessageEvent(0x04, 0x02) → sendMsg ; registerFileListenerSync() beforehand
 followed by sub-file pulls: FileRequest{name,uri,serviceId=4} → MessageEvent(0x1a, 0x01)
 ← content arrives through the file-transfer channel (not a protobuf message response)
```

### (0x4, 0x09) Fitness
```
FitnessDataFetcher.requestFitnessFileList() → TimeRangeRequest(9)
 → MessageEvent(0x04, 0x09) → sendMsg ; file receiver armed beforehand
 ← content arrives through the file-transfer channel
```

### (0x5, 0x8b) Daily activity — bidirectional
```
RX: DailyActivityStatisticsProcessor.acceptMsgTypes()=(5,0x8b)
    onMessageReceived → event.getData()[B → SportStatList.parseFrom  [L372–L376]
TX: sendFusionDataToDevice(mac) → SportStatList.build().toByteArray()
    → MessageEvent(0x05, 0x8b) (skipped when data size = 0)
```

---

## Notes / Limits

- Numeric types above follow the generated accessor signatures (I = int32, J = int64, String, ByteString). No float/double/bool fields were observed in these message classes.
- Payload semantics for commands without an explicit protobuf class (e.g. several `0x5` TX-only helpers) are `UNKNOWN`.
- `supportTlv` semantics (TLV framing) are `LIKELY` related to record pagination; not verified on wire.