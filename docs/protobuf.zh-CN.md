# Protobuf 负载与字段映射

**简体中文 | [English](protobuf.md)**

字段编号提取自 smali 中生成的 `*_FIELD_NUMBER` 常量（`FitnessProto`、`FileProto` — classes14.dex）。状态：`CONFIRMED`（静态）；未做线上级验证（无 BLE 层抓包）。

---

## 请求类型

### `FitnessProto.TimeRangeRequest`（心率统计、运动统计、运动记录列表、健身）

| 字段 | # | 类型 | 备注 |
|-------|---|------|-------|
| `startTimestamp` | 1 | int32 | epoch 秒 |
| `endTimestamp` | 2 | int32 | epoch 秒 |
| `supportTlv` | 3 | int32 | `SportRecordListFetcher` 中设为 2；其他位置默认 0 |

### `FitnessProto.TypeRequest`

| 字段 | # | 类型 |
|-------|---|------|
| `type` | 1 | int32 |

由 `SportStatDataFetcher` 在"仅今天"分支使用（`setType(1)`）。

### `FitnessProto.StringRequest`

| 字段 | # | 类型 |
|-------|---|------|
| `value` | 1 | string |

由 `SportRecordFetcher` 使用（`setValue(sportId)`）。

### `FileProto.FileRequest`（文件传输通道）

| 字段 | # | 类型 | 备注 |
|-------|---|------|-------|
| `name` | 1 | string | 文件名 |
| `uri` | 2 | string | URI |
| `serviceId` | 3 | int32 | 运动记录子文件设为 4 |
| `state` | 4 | int32 | — |

---

## 响应类型

### `FitnessProto.HeartRateStatData`

| 字段 | # | 类型 | 备注 |
|-------|---|------|-------|
| `data` | 1 | repeated `HeartRateStat` | |

另有：`FitnessProtoV2.HeartRateStatDataV2`（V2 设备分支；`parseFrom` 有实证）。

### `FitnessProto.HeartRateStat`

| 字段 | # | 类型 |
|-------|---|------|
| `timestamp` | 1 | int32 |
| `avgWalkHeartRate` | 2 | int32 |
| `restHeartRate` | 3 | int32 |
| `sleepHeartRate` | 4 | int32 |

### `FitnessProto.SportStatData`

| 字段 | # | 类型 |
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

| 字段 | # | 类型 |
|-------|---|------|
| `data` | 1 | repeated `SportStatData` |

### `FitnessProto.FileNameData`（运动记录列表响应）

| 字段 | # | 类型 | 备注 |
|-------|---|------|-------|
| `startTime` | 1 | int32 | |
| `endTime` | 2 | int32 | |
| `fileName` | 3 | repeated string | |
| `moreData` | 4 | int32 | 分页标志 |
| `fileType` | 5 | bytes | |

---

## 观察到的数据流（可调用级）

### (0x5, 0x35) 心率统计 — 请求/响应
```
HRStatDataFetcher.startFetch() [c12]
 → TimeRangeDataFetcher.getTimeRangeRequest(0xe) → TimeRangeRequest{1,2}
 → toByteArray() → new MessageEvent(0x05, 0x35, data)
 → BTClient.sendMsg(event, callback)
 ← callback → onHeartRateStatData([B)
 ← FitnessProto.HeartRateStatData.parseFrom   （或 HeartRateStatDataV2）
```

### (0x5, 0x36) 运动统计
```
SportStatDataFetcher.startFetch() [c12]
 分支 A（仅今天）：(sid, cid) 由设备能力动态决定 + TypeRequest{type=1}
 分支 B（时间范围）：TimeRangeRequest(0xb) → MessageEvent(0x05, 0x36)
 ← SportStatData.parseFrom / SportStatList.parseFrom
```

### (0x4, 0x01) 运动记录列表
```
SportRecordListFetcher.fetchFileIndex() → TimeRangeRequest(supportTlv=2)
 → MessageEvent(0x04, 0x01) → sendMsg
 ← onFileIndexResult → FitnessProto.FileNameData.parseFrom
```

### (0x4, 0x02) 单条运动记录
```
SportRecordFetcher.requestRptFile() → StringRequest{value=sportId}
 → MessageEvent(0x04, 0x02) → sendMsg ；事先 registerFileListenerSync()
 随后拉取子文件：FileRequest{name,uri,serviceId=4} → MessageEvent(0x1a, 0x01)
 ← 内容经文件传输通道到达（不是 protobuf 消息响应）
```

### (0x4, 0x09) 健身
```
FitnessDataFetcher.requestFitnessFileList() → TimeRangeRequest(9)
 → MessageEvent(0x04, 0x09) → sendMsg ；事先挂好文件接收器
 ← 内容经文件传输通道到达
```

### (0x5, 0x8b) 日常活动 — 双向
```
RX: DailyActivityStatisticsProcessor.acceptMsgTypes()=(5,0x8b)
    onMessageReceived → event.getData()[B → SportStatList.parseFrom  [L372–L376]
TX: sendFusionDataToDevice(mac) → SportStatList.build().toByteArray()
    → MessageEvent(0x05, 0x8b)（数据长度为 0 时跳过）
```

---

## 注 / 限制

- 上述数值类型依据生成的访问器签名（I = int32、J = int64、String、ByteString）。这些消息类中未观察到 float/double/bool 字段。
- 没有显式 protobuf 类的命令（例如若干 `0x5` 域仅发送侧辅助类）负载语义为 `UNKNOWN`。
- `supportTlv` 的语义（TLV 分帧）`LIKELY` 与记录分页相关；未经线上验证。