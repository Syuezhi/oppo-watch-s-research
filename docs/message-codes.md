# Message Codes — serviceId / commandId Dictionary

All entries recovered via **smali register-backtracking** (constant extraction from `MessageEvent` constructors and `MsgType.create(sid, cid)` calls), with send/receive cross-validation where noted. Status: `CONFIRMED` unless marked otherwise.

- `MessageEvent(serviceId, commandId, byte[])` — see `protobuf.md` / `architecture.md`
- `RX` = watch → phone (handler: `MsgProcessor`), `TX` = phone → watch (constructor call site), `BIDI` = observed both directions
- Class path prefix for RX handlers: `com.heytap.device.data.sporthealth.receive.*` (classes12) unless stated

---

## Service Map

| svc | domain |
|-----|--------|
| 0x01 | DND / account / phone-display |
| 0x04 | workout & fitness devices |
| 0x05 | health data (main) |
| 0x0d | watchface |
| 0x19 | location / AGPS |
| 0x1a | file transfer |

---

## Full Dictionary

### svc = 0x01

| cmd | dir | class | payload | business |
|-----|-----|-------|---------|----------|
| 0x6d | RX | `DNDMsgProcessor` | `DNDProto$DoNotDisturb` | Do-not-disturb |
| 0x6c / 0x6b / 0xa8 / 0x6e | TX | `device/sleep/DoNotDisturbRepository$Companion` (L146/529/890/931) | — | DND control family |
| 0xa7 | BIDI | `AccountTicketProcessor` / `DeviceAccountServiceImpl` (L574) | — | Account ticket |
| 0x84 | BIDI | `PhoneDisplayMsgProcessor` / `DisplayChangeReceiver$Companion` (L178) | — | Phone display state |

### svc = 0x04

| cmd | dir | class | payload | business |
|-----|-----|-------|---------|----------|
| 0x0e | BIDI | `AirPressureProcessor` / `AirPressureRepository` (L176) | `WorkoutProto$AltitudeRequest` | air pressure / altitude |
| 0x3e | BIDI | `ExerciseIntensityProcessor` / `$handleExerciseIntensitySyncResponse$1` (L413) | `WorkoutProto$sport_record_intensity_multi_cell` | exercise intensity |
| 0x45 | RX | `ExerciseLoadRequestProcessor` | — | exercise load request |
| 0x44 | TX | `ExerciseLoadRatioSender` (L946) | — | exercise load ratio |
| 0x0c | RX | `Run3KmTimeProcessor` | `WorkoutProto$Run3KmTime` | 3 km result |
| 0x22 | RX | `MotionStateProcessor` | `WorkoutProto$MotionState` | motion state |
| 0x15 / 0x16 / 0x2e | RX | `OperationProcessor` | — | operation family (semantics UNKNOWN) |
| 0x32 | RX | `HFGpsFileMsgProcessor` | — | high-fidelity GPS file |
| 0x37 | TX | `HFGpsFileRepositoryKt` (L1433) | — | GPS file send |
| 0x09 | TX | `FitnessDataFetcher` (L357) | `TimeRangeRequest` | fitness data pull |
| 0x01 | TX | `SportRecordListFetcher` (L606) | `TimeRangeRequest` | sport record list |
| 0x02 | TX | `SportRecordFetcher` (L1715) | `StringRequest` | single sport record |

### svc = 0x05 (main)

| cmd | dir | class | payload | business |
|-----|-----|-------|---------|----------|
| 0x20 | BIDI | `SleepStateMsgProcessor` (L316) | `FitnessProto$SleepStateChangeNotify` | sleep state |
| 0x19 | BIDI | `Spo2ManualMsgProcessor` (L122) | `FitnessProto$Spo2Data` | SpO2 |
| 0x40 | BIDI | `WearRecordMsgProcessor` (L568) | `FitnessProto$WearRecordData` | wear record |
| 0x42 | BIDI | `WeightMsgProcessor` (L608) | `FitnessProto$IntRequest` | weight |
| 0x41 | RX | `SportHealthDataNotifyProcessor` | `FitnessProto$DeviceDataChangedNotify` | data-changed notify |
| 0x27 / 0x28 / 0x2b | RX | `FamilyDataShareMsgProcessor` | `FitnessProto$PushData` | family data share |
| 0xf5 | BIDI | `UserInfoProcessor` (`$handleUserInfoRsp$1` L811) | `UserInfoProto$UserInfo` | user info |
| 0xa1 | BIDI | `SyncAccountBodyInfoProcessor` (L1327/1412) | `FitnessProto$UserBodyInfoRequest` | body info sync |
| 0xf8 | RX | `MenstrualCycleMsgProcessor` | — | menstrual cycle |
| 0xe9 | TX | `MenstrualCyclePredictiveDataHelper$Companion` (L577) | — | menstrual prediction |
| 0xcb | BIDI | `SnoreActiveStateProcessor` (L491) | `FitnessProto$OsaStateRsp` | snore state |
| 0xc8 | RX | `OsaResultDaysProcessor` | `FitnessProto$OsaDataReq` | OSA result |
| 0xc9 | TX | `OsaResultDaysUtil$Companion$sendDataToDevice$1` (L517) | — | OSA data |
| 0x6d | RX | `CardiovascularPrepareRemindProcessor` | `FitnessProto$TypeRequest` | cardiovascular remind |
| 0x71 | RX | `ScienceInfoProcessor` | — | science info |
| 0x7c | TX | `WristTemperatureHelper$Companion` (L492) | — | wrist temperature |
| 0x86 | BIDI | `SymptomRequestProcessor` (L296) | — | symptom request |
| 0x8b | BIDI | `DailyActivityStatisticsProcessor` (`$send…$1` L303) | `FitnessProto$SportStatList` | daily activity |
| 0xa5 | TX | `PhoneScreenStateHelper$Companion` (L411) | — | phone screen state |
| 0xdc | RX | `OpenPageMsgProcessor` | `FitnessProto$OpenPage` | open page |
| 0xde | TX | `SleepRestRepository` (L1787) | — | sleep rest |
| 0xe6 | TX | `WTTOOBEUtil$Companion$sendDataToDevice$1` (L274) | — | OOBE flow |
| 0xea | TX | `SendBPDataToDeviceHelper$Companion` (L311) | — | blood pressure data |
| 0xff | TX | `GeoFenceUtil$Companion` (L342) | — | geo fence |
| 0x48 | RX | `SleepRestSettingProcessor` | `FitnessProto$RemindPopUp` | sleep rest setting |
| 0x49 / 0x4f / 0x4a / 0xcc / 0xcd / 0xd1 | RX | `SleepModelSettingProcessor` | — | sleep mode setting |
| 0x4a–0x51 | TX | `SleepModeBTRepository$Companion` (L145–1422) | — | sleep mode control family |
| 0x35 | TX | `HRStatDataFetcher` (L382) | `TimeRangeRequest` | HR stats pull |
| 0x36 | TX | `SportStatDataFetcher` (L457) | `TimeRangeRequest` / `TypeRequest` | sport stats pull |
| 0x34 | TX | `SportHealthDataRepository` (L241) | — | sport health data |
| 0x77 | TX | `BloodSugarDeviceDataFetcher` (L413) | — | blood sugar (device) |
| 0xdb / 0xe5 | TX | `DataSyncRepository` (L496/2800/1455) | — | data sync |
| 0x45 / 0x46 / 0x5f / 0x68 | TX | `DataSyncServiceImpl` (L991/1519/1645/1313) | — | data sync service |
| 0x10d | BIDI | `SleepPhoneStateProcessor` (L99) | — | phone state during sleep |
| 0x32 | BIDI | `SportHealthSettingProcessor` (L280) | `FitnessProto$StringRequest` | sport health settings |
| 0xf6 | RX | `DevSkipTodayProcessor` | `FitnessProtoV2$SkipToday` | skip today |

### svc = 0x0d (watchface)

| cmd | dir | class | business |
|-----|-----|-------|----------|
| 0x06–0x0c | TX | `bandface/watchface/model/BandFaceManager` (L574–2258, 8 call sites) | watchface management |

### svc = 0x19 (location)

| cmd | dir | class | payload | business |
|-----|-----|-------|---------|----------|
| 0x05 / 0x08 | RX | `AGPSMsgProcessor` | `LocationProto$AGPSRequest` | AGPS |
| 0x06 / 0x07 | TX | `AGPSRepositoryKt` (L981) / `AGPSTransferTask$run$1` (L404) | — | AGPS send |
| 0x01 / 0x02 | RX | `LocationMsgProcessor` | — | location |
| 0x03 / 0x09 | TX | `LocationClient` (L3186/3306) | — | location client |
| 0x0a | BIDI | `PhoneMotionStateMsgProcessor` (L336) | `LocationProto$MotionStateRequest` | phone motion state |

### svc = 0x1a (file transfer)

| cmd | dir | class | business |
|-----|-----|-------|----------|
| 0x01 / 0x02 | TX | `FileDataFetcher` (L365/424), `SportRecordFetcher` (L501) | file transfer requests (`FileProto$FileRequest`) |

---

## Notes

- `MsgType.create(a, b)` mapping is `(serviceId, commandId)`; verified against multi-sample cross-checks (SleepState / DailyActivity / PhoneDisplay).
- Multi-command processors (FamilyDataShare ×3, Operation ×3/4, SleepModelSetting ×4) listed in full.
- Commands present in code but NOT yet linked to a processor or a send-site are intentionally omitted (e.g. parts of the Operation family — semantics UNKNOWN).
- Watchface codes 0x06–0x0c are TX-only observations (no RX handler identified).
