# 消息代码 — serviceId / commandId 字典

**简体中文 | [English](message-codes.md)**

全部条目通过 **smali 寄存器回溯** 还原（从 `MessageEvent` 构造函数和 `MsgType.create(sid, cid)` 调用中提取常量），并按要求做了收发交叉验证。除特别标注外，状态均为 `CONFIRMED`。

- `MessageEvent(serviceId, commandId, byte[])` —— 见 `protobuf.md` / `architecture.md`
- `RX` = 手表 → 手机（处理器：`MsgProcessor`），`TX` = 手机 → 手表（构造函数调用点），`BIDI` = 双向均有观察
- RX 处理器的类路径前缀：`com.heytap.device.data.sporthealth.receive.*`（classes12），另有说明除外

---

## 服务映射

| svc | 域 |
|-----|--------|
| 0x01 | 勿扰 / 账户 / 手机显示 |
| 0x04 | 运动与健身器械 |
| 0x05 | 健康数据（主域） |
| 0x0d | 表盘 |
| 0x19 | 定位 / AGPS |
| 0x1a | 文件传输 |

---

## 完整字典

### svc = 0x01

| cmd | 方向 | 类 | 负载 | 业务 |
|-----|-----|-------|---------|----------|
| 0x6d | RX | `DNDMsgProcessor` | `DNDProto$DoNotDisturb` | 勿扰 |
| 0x6c / 0x6b / 0xa8 / 0x6e | TX | `device/sleep/DoNotDisturbRepository$Companion` (L146/529/890/931) | — | 勿扰控制族 |
| 0xa7 | BIDI | `AccountTicketProcessor` / `DeviceAccountServiceImpl` (L574) | — | 账户票据 |
| 0x84 | BIDI | `PhoneDisplayMsgProcessor` / `DisplayChangeReceiver$Companion` (L178) | — | 手机显示状态 |

### svc = 0x04

| cmd | 方向 | 类 | 负载 | 业务 |
|-----|-----|-------|---------|----------|
| 0x0e | BIDI | `AirPressureProcessor` / `AirPressureRepository` (L176) | `WorkoutProto$AltitudeRequest` | 气压 / 海拔 |
| 0x3e | BIDI | `ExerciseIntensityProcessor` / `$handleExerciseIntensitySyncResponse$1` (L413) | `WorkoutProto$sport_record_intensity_multi_cell` | 运动强度 |
| 0x45 | RX | `ExerciseLoadRequestProcessor` | — | 运动负荷请求 |
| 0x44 | TX | `ExerciseLoadRatioSender` (L946) | — | 运动负荷比 |
| 0x0c | RX | `Run3KmTimeProcessor` | `WorkoutProto$Run3KmTime` | 3公里成绩 |
| 0x22 | RX | `MotionStateProcessor` | `WorkoutProto$MotionState` | 运动状态 |
| 0x15 / 0x16 / 0x2e | RX | `OperationProcessor` | — | 操作族（语义 UNKNOWN） |
| 0x32 | RX | `HFGpsFileMsgProcessor` | — | 高保真 GPS 文件 |
| 0x37 | TX | `HFGpsFileRepositoryKt` (L1433) | — | GPS 文件发送 |
| 0x09 | TX | `FitnessDataFetcher` (L357) | `TimeRangeRequest` | 健身数据拉取 |
| 0x01 | TX | `SportRecordListFetcher` (L606) | `TimeRangeRequest` | 运动记录列表 |
| 0x02 | TX | `SportRecordFetcher` (L1715) | `StringRequest` | 单条运动记录 |

### svc = 0x05（主域）

| cmd | 方向 | 类 | 负载 | 业务 |
|-----|-----|-------|---------|----------|
| 0x20 | BIDI | `SleepStateMsgProcessor` (L316) | `FitnessProto$SleepStateChangeNotify` | 睡眠状态 |
| 0x19 | BIDI | `Spo2ManualMsgProcessor` (L122) | `FitnessProto$Spo2Data` | 血氧 |
| 0x40 | BIDI | `WearRecordMsgProcessor` (L568) | `FitnessProto$WearRecordData` | 穿戴记录 |
| 0x42 | BIDI | `WeightMsgProcessor` (L608) | `FitnessProto$IntRequest` | 体重 |
| 0x41 | RX | `SportHealthDataNotifyProcessor` | `FitnessProto$DeviceDataChangedNotify` | 数据变更通知 |
| 0x27 / 0x28 / 0x2b | RX | `FamilyDataShareMsgProcessor` | `FitnessProto$PushData` | 家庭数据共享 |
| 0xf5 | BIDI | `UserInfoProcessor`（`$handleUserInfoRsp$1` L811） | `UserInfoProto$UserInfo` | 用户信息 |
| 0xa1 | BIDI | `SyncAccountBodyInfoProcessor` (L1327/1412) | `FitnessProto$UserBodyInfoRequest` | 身体信息同步 |
| 0xf8 | RX | `MenstrualCycleMsgProcessor` | — | 生理周期 |
| 0xe9 | TX | `MenstrualCyclePredictiveDataHelper$Companion` (L577) | — | 经期预测 |
| 0xcb | BIDI | `SnoreActiveStateProcessor` (L491) | `FitnessProto$OsaStateRsp` | 打鼾状态 |
| 0xc8 | RX | `OsaResultDaysProcessor` | `FitnessProto$OsaDataReq` | OSA 结果 |
| 0xc9 | TX | `OsaResultDaysUtil$Companion$sendDataToDevice$1` (L517) | — | OSA 数据 |
| 0x6d | RX | `CardiovascularPrepareRemindProcessor` | `FitnessProto$TypeRequest` | 心血管提醒 |
| 0x71 | RX | `ScienceInfoProcessor` | — | 科学信息 |
| 0x7c | TX | `WristTemperatureHelper$Companion` (L492) | — | 腕温 |
| 0x86 | BIDI | `SymptomRequestProcessor` (L296) | — | 症状请求 |
| 0x8b | BIDI | `DailyActivityStatisticsProcessor`（`$send…$1` L303） | `FitnessProto$SportStatList` | 日常活动 |
| 0xa5 | TX | `PhoneScreenStateHelper$Companion` (L411) | — | 手机屏幕状态 |
| 0xdc | RX | `OpenPageMsgProcessor` | `FitnessProto$OpenPage` | 打开页面 |
| 0xde | TX | `SleepRestRepository` (L1787) | — | 睡眠休息 |
| 0xe6 | TX | `WTTOOBEUtil$Companion$sendDataToDevice$1` (L274) | — | OOBE 流程 |
| 0xea | TX | `SendBPDataToDeviceHelper$Companion` (L311) | — | 血压数据 |
| 0xff | TX | `GeoFenceUtil$Companion` (L342) | — | 地理围栏 |
| 0x48 | RX | `SleepRestSettingProcessor` | `FitnessProto$RemindPopUp` | 睡眠休息设置 |
| 0x49 / 0x4f / 0x4a / 0xcc / 0xcd / 0xd1 | RX | `SleepModelSettingProcessor` | — | 睡眠模式设置 |
| 0x4a–0x51 | TX | `SleepModeBTRepository$Companion` (L145–1422) | — | 睡眠模式控制族 |
| 0x35 | TX | `HRStatDataFetcher` (L382) | `TimeRangeRequest` | 心率统计拉取 |
| 0x36 | TX | `SportStatDataFetcher` (L457) | `TimeRangeRequest` / `TypeRequest` | 运动统计拉取 |
| 0x34 | TX | `SportHealthDataRepository` (L241) | — | 运动健康数据 |
| 0x77 | TX | `BloodSugarDeviceDataFetcher` (L413) | — | 血糖（设备） |
| 0xdb / 0xe5 | TX | `DataSyncRepository` (L496/2800/1455) | — | 数据同步 |
| 0x45 / 0x46 / 0x5f / 0x68 | TX | `DataSyncServiceImpl` (L991/1519/1645/1313) | — | 数据同步服务 |
| 0x10d | BIDI | `SleepPhoneStateProcessor` (L99) | — | 睡眠期间手机状态 |
| 0x32 | BIDI | `SportHealthSettingProcessor` (L280) | `FitnessProto$StringRequest` | 运动健康设置 |
| 0xf6 | RX | `DevSkipTodayProcessor` | `FitnessProtoV2$SkipToday` | 跳过今日 |

### svc = 0x0d（表盘）

| cmd | 方向 | 类 | 业务 |
|-----|-----|-------|----------|
| 0x06–0x0c | TX | `bandface/watchface/model/BandFaceManager` (L574–2258，8处调用点) | 表盘管理 |

### svc = 0x19（定位）

| cmd | 方向 | 类 | 负载 | 业务 |
|-----|-----|-------|---------|----------|
| 0x05 / 0x08 | RX | `AGPSMsgProcessor` | `LocationProto$AGPSRequest` | AGPS |
| 0x06 / 0x07 | TX | `AGPSRepositoryKt` (L981) / `AGPSTransferTask$run$1` (L404) | — | AGPS 发送 |
| 0x01 / 0x02 | RX | `LocationMsgProcessor` | — | 定位 |
| 0x03 / 0x09 | TX | `LocationClient` (L3186/3306) | — | 定位客户端 |
| 0x0a | BIDI | `PhoneMotionStateMsgProcessor` (L336) | `LocationProto$MotionStateRequest` | 手机运动状态 |

### svc = 0x1a（文件传输）

| cmd | 方向 | 类 | 业务 |
|-----|-----|-------|----------|
| 0x01 / 0x02 | TX | `FileDataFetcher` (L365/424)、`SportRecordFetcher` (L501) | 文件传输请求（`FileProto$FileRequest`） |

---

## 注

- `MsgType.create(a, b)` 的映射即 `(serviceId, commandId)`；已用多组样本交叉核对（SleepState / DailyActivity / PhoneDisplay）。
- 多命令处理器（FamilyDataShare ×3、Operation ×3/4、SleepModelSetting ×4）完整列出。
- 代码中存在但尚未关联到处理器或发送点的命令有意省略（例如 Operation 族的部分条目 —— 语义 UNKNOWN）。
- 表盘命令 0x06–0x0c 为仅发送侧观察（未识别到 RX 处理器）。
