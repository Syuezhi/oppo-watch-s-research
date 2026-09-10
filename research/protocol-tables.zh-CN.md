# 研究 — 原始协议表

**简体中文 | [English](protocol-tables.md)**

从 smali 分析（寄存器回溯）中机器提取的表。保留以供复现。

---

## 表 1 — `acceptMsgTypes()` 注册（42 条）

来源：`com.heytap.device.data.sporthealth.receive.*`（classes12.dex），通过 `MsgProcessor$MsgType;->create(II)` 的寄存器回溯提取。

| 处理器 | serviceId | commandId |
|-----------|-----------|-----------|
| AGPSMsgProcessor | 0x19 | 0x05, 0x08 |
| AccountTicketProcessor | 0x01 | 0xa7 |
| AirPressureProcessor | 0x04 | 0x0e |
| CardiovascularPrepareRemindProcessor | 0x05 | 0x6d |
| DNDMsgProcessor | 0x01 | 0x6d |
| DailyActivityStatisticsProcessor | 0x05 | 0x8b |
| DevSkipTodayProcessor | 0x05 | 0xf6 |
| ExerciseIntensityProcessor | 0x04 | 0x3e |
| ExerciseLoadRequestProcessor | 0x04 | 0x45 |
| FamilyDataShareMsgProcessor | 0x05 | 0x27, 0x28, 0x2b |
| HFGpsFileMsgProcessor | 0x04 | 0x32 |
| LocationMsgProcessor | 0x19 | 0x01, 0x02 |
| MenstrualCycleMsgProcessor | 0x05 | 0xf8 |
| MotionStateProcessor | 0x04 | 0x22 |
| OpenPageMsgProcessor | 0x05 | 0xdc |
| OperationProcessor | 0x04 | 0x15, 0x16, 0x2e (+1) |
| OsaResultDaysProcessor | 0x05 | 0xc8 |
| PhoneDisplayMsgProcessor | 0x01 | 0x84 |
| PhoneMotionStateMsgProcessor | 0x19 | 0x0a |
| Run3KmTimeProcessor | 0x04 | 0x0c |
| ScienceInfoProcessor | 0x05 | 0x71 |
| SleepModelSettingProcessor | 0x05 | 0x49, 0x4a, 0x4f, 0xcc, 0xcd, 0xd1 |
| SleepPhoneStateProcessor | 0x05 | 0x10d |
| SleepRestSettingProcessor | 0x05 | 0x48 |
| SleepStateMsgProcessor | 0x05 | 0x20 |
| SnoreActiveStateProcessor | 0x05 | 0xcb |
| Spo2ManualMsgProcessor | 0x05 | 0x19 |
| SportHealthDataNotifyProcessor | 0x05 | 0x41; 0x04 → 0x27 |
| SportHealthSettingProcessor | 0x05 | 0x32 |
| SymptomRequestProcessor | 0x05 | 0x86 |
| SyncAccountBodyInfoProcessor | 0x05 | 0xa1 |
| UserInfoProcessor | 0x05 | 0xf5 |
| WearRecordMsgProcessor | 0x05 | 0x40 |
| WeightMsgProcessor | 0x05 | 0x42 |

## 表 2 — `MessageEvent` 构造点（发送侧，63 条）

来源：classes12.dex，通过 `MessageEvent;-><init>(II[B)` 的寄存器回溯提取。

| 类（路径片段） | 行号 | svc | cmd |
|-----------------------|------|-----|-----|
| DataSyncRepository$sendDailyActivityStateDataToDevice$1$1 | 496 | 0x5 | 0xdb |
| DataSyncRepository | 1455 / 2800 | 0x5 | 0xe5 / 0xdb |
| DeviceAccountServiceImpl | 574 | 0x1 | 0xa7 |
| DataSyncServiceImpl | 991 / 1313 / 1519 / 1645 | 0x5 | 0x45 / 0x68 / 0x46 / 0x5f |
| WearingStatusManager | 447 | 0x1 | 0x38 |
| ExerciseLoadRatioSender | 946 | 0x4 | 0x44 |
| BloodSugarDeviceDataFetcher | 413 | 0x5 | 0x77 |
| FileDataFetcher | 365 / 424 | 0x1a | 0x1 / 0x2 |
| FitnessDataFetcher | 357 | 0x4 | 0x9 |
| HRStatDataFetcher | 382 | 0x5 | 0x35 |
| SportRecordFetcher | 501 / 1715 | 0x1a / 0x4 | 0x1 / 0x2 |
| SportStatDataFetcher | 457 | 0x5 | 0x36 |
| SportRecordListFetcher | 606 | 0x4 | 0x1 |
| AGPSTransferTask$run$1 | 404 | 0x19 | 0x7 |
| AGPSRepositoryKt | 981 | 0x19 | 0x6 |
| AirPressureRepository | 176 | 0x4 | 0xe |
| DisplayChangeReceiver$Companion | 178 | 0x1 | 0x84 |
| DailyActivityStatisticsProcessor$send…$1 | 303 | 0x5 | 0x8b |
| ExerciseIntensityProcessor$handle…$1 | 413 | 0x4 | 0x3e |
| LocationClient | 3186 / 3306 | 0x19 | 0x3 / 0x9 |
| Spo2ManualMsgProcessor | 122 | 0x5 | 0x19 |
| SnoreActiveStateProcessor | 491 | 0x5 | 0xcb |
| SymptomRequestProcessor | 296 | 0x5 | 0x86 |
| WeightMsgProcessor | 608 | 0x5 | 0x42 |
| HFGpsFileRepositoryKt | 1433 | 0x4 | 0x37 |
| PhoneMotionStateMsgProcessor | 336 | 0x19 | 0xa |
| SleepPhoneStateProcessor | 99 | 0x5 | 0x10d |
| SleepStateMsgProcessor | 316 | 0x5 | 0x20 |
| SportHealthSettingProcessor | 280 | 0x5 | 0x32 |
| UserInfoProcessor$handleUserInfoRsp$1 | 811 | 0x5 | 0xf5 |
| SyncAccountBodyInfoProcessor | 1327 / 1412 | 0x5 | 0xa1 |
| WearRecordMsgProcessor | 568 | 0x5 | 0x40 |
| SportHealthDataRepository | 241 | 0x5 | 0x34 |
| GeoFenceUtil$Companion | 342 | 0x5 | 0xff |
| MenstrualCyclePredictiveDataHelper$Companion | 577 | 0x5 | 0xe9 |
| OsaResultDaysUtil$sendDataToDevice$1 | 517 | 0x5 | 0xc9 |
| PhoneScreenStateHelper$Companion | 411 | 0x5 | 0xa5 |
| WTTOOBEUtil$sendDataToDevice$1 | 274 | 0x5 | 0xe6 |
| WristTemperatureHelper$Companion | 492 | 0x5 | 0x7c |
| DoNotDisturbRepository$Companion | 146 / 529 / 890 / 931 | 0x1 | 0x6c / 0x6b / 0xa8 / 0x6e |
| SleepModeBTRepository$Companion | 145 / 193 / 313 / 335 / 448 / 957 / 1184 | 0x5 | 0x4d / 0x4c / 0x51 / 0x4a / 0x4b / 0x4e / 0x50 |
| SleepRestRepository | 1787 | 0x5 | 0xde |
| BandFaceManager | 574 / 631 / 737 / 933 / 960 / 1759 / 1999 / 2258 | 0xd | 0x9 / 0xa / 0xb / 0x7 / 0x6 / 0xa / 0xc / 0x8 |
| SendBPDataToDeviceHelper$Companion | 311 | 0x5 | 0xea |

*（表已截取为唯一的类/行号条目；多路径文件产生的重复项已合并）*

---

## 注

- 使用 Python 寄存器回溯脚本在 baksmali 输出上提取（分析时可用 classes2–14）。
- 此处的 `svc`/`cmd` 值为**构造函数处挂载的原值**：第一个 int 参数 → `mServiceId`，第二个 → `mCommandId`（依据 `MessageEvent.<init>(II[B)` 的字段存储，已在多组样本上验证）。
- 仅列出方法窗口内可解析常量 ≥2 个的条目；畸形/歧义位置已排除。