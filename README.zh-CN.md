# OPPO Watch S 逆向工程与集成研究

**简体中文 | [English](README.md)**

关于 **OPPO Watch S（OWWE262）** 与 **OHealth（com.heytap.health）** 之间通信架构的研究笔记——协议还原、IPC 权限分析、第三方 App 集成路径评估。

---

## 状态

| 领域 | 结果 |
|------|------|
| 协议层（serviceId/commandId、MessageEvent、protobuf） | **基本还原**（smali 寄存器回溯 + 交叉验证） |
| 第三方 App 的传输/通道层 | **封锁** —— signature 级权限（`com.heytap.wearable.linkservice.permission.WEARABLE: signature`） |
| Health Connect 路线 | **在所测设备上不成立** —— OHealth 不向 HC 写入任何手表数据 |
| OHealth 广播触发 | **仅本地刷新** —— 不触发手表同步 |
| **OHealth→HealthConnect→第三方 路线总评级** | **C —— 在所测配置上不可行** |

---

## 测试设备

- **手表**：OPPO Watch S，型号 **OWWE262**（已配对、已连接、走经典蓝牙）
- **手机**：OPPO / ColorOS 设备（系统含 HealthFitness APEX 模块）
- **OHealth**：`com.heytap.health` v1.0.3（单个 base.apk、无 split 拆分；17 个 dex；约 139 MB / 133 MiB）
- **工具**：baksmali（Ubuntu/proot）、smali 寄存器回溯脚本、Shizuku shell（用于动态测试）

---

## 摘要（结论）

1. **手表通过 OHealth 的 `linkservice` 栈使用一套私有消息协议通信**，不走任何标准 GATT 蓝牙规范。消息单元为 `MessageEvent(serviceId, commandId, byte[])`，承载 protobuf 负载，请求/响应机制为回调式（`waitResponseMsgMap`，键 = serviceId+commandId 拼接）。
2. **约 40+ 条业务命令被还原**，含完整收发链：心率统计、运动统计、运动记录、睡眠、血氧、体重、穿戴记录、生理周期、打鼾/OSA、腕温、表盘、AGPS、文件传输等。详见 `docs/message-codes.md`。
3. **通道对第三方 App 封锁。** OHealth 对外入口（`WearableServer`，action `com.heytap.wearable.linkservice.action.WEARABLE`）受 **signature 级权限**保护；其余候选（`IpcBtService` ×2）均未导出。`IWearableService` 存在且接口丰富，但 **“接口存在 ≠ 第三方可调”**。
4. **广播实验未能触发手表同步。** `com.heytap.health.action_data_refresh` 会到达接收者并触发本地数据刷新（实证：`SportHealthDataAPI.readSportHealthData`），但之后没有 BLE/传输层活动。另外四个健康/睡眠广播无可观测的手表侧动作。
5. **本设备的 Health Connect 中没有手表数据。** OHealth 声明 **0 条** `android.permission.health.*` 权限；`com.oplus.healthservice`（v16.2.10）含 **0 处** Health Connect 引用；全部 17 个 dex 含 **0 处** `HealthConnect` 引用；动态监测无任何写入。HC 机制存在，但没有手表数据源。**评级 C。**

---

## 架构

```
OPPO Watch S (OWWE262)
        │  私有链路（实测为经典蓝牙：BR/EDR 已连接）
        ▼
OHealth (com.heytap.health) — 3 个进程：main / :transport / :once
        │
        ├── BTClient.sendMsg(MessageEvent)          [com.heytap.health.devicemanager.btclient]
        │        ▼
        ├── MessageTransferManager.sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)
        │        ▼
        └── com.oplus.wearable.linkservice transport（WearableServer / IWearableService / IpcBtService）
                 ▼
            Watch S
```

接收方向：

```
transport → BTClient.onMessageReceived → dispatchMsgResponse → dispatchToListener
→ ReceiveMsgListener → MsgProcessor.onMessageReceived(mac, MessageEvent)
→ event.getData()[B → XxxProto.parseFrom([B) → 业务层
```

细节见 `docs/architecture.md`、`docs/transport.md`。

---

## 协议发现

- `MessageEvent(II[B)`：`(serviceId, commandId, data)`；默认值：`encryptOption=0（设备默认）`、`cacheOption=1`、`transport=2（蓝牙）`、`priority=middle`。
- 仅 **OTA** 与 **eSIM** 负载显式设置 `encryptOption`；加密（当被使用时）在传输层施加（`TransportLayerV1Wrapper` → `SecurityManager.encrypt/decrypt` → `com.oplus.wearable.crypto.Cipher`）。**未提取任何密钥；未破解任何加密。**
- 完整命令字典见 `docs/message-codes.md`。

---

## 消息协议（可调用级命令）

| svc | cmd | 业务 | 请求 protobuf | 响应 protobuf |
|-----|-----|------|----------------|----------------|
| 0x05 | 0x35 | 心率统计 | `FitnessProto.TimeRangeRequest` | `HeartRateStatData`（+V2） |
| 0x05 | 0x36 | 运动统计 | `TimeRangeRequest` / `TypeRequest` | `SportStatData` / `SportStatList` |
| 0x04 | 0x01 | 运动记录列表 | `TimeRangeRequest`（supportTlv=2） | `FileNameData` |
| 0x04 | 0x02 | 运动记录 | `StringRequest`（sportId） | 文件传输通道 |
| 0x04 | 0x09 | 健身数据 | `TimeRangeRequest` | 文件传输通道 |
| 0x05 | 0x8b | 日常活动 | `SportStatList`（双向） | `SportStatList` |

字段级细节见 `docs/protobuf.md`。

---

## IPC 边界（摘要）

| 组件 | 暴露情况 | 保护 |
|------|----------|------|
| `WearableServer` | 有 filter（`...action.WEARABLE`） | `com.heytap.wearable.linkservice.permission.WEARABLE` → **signature** |
| `IpcBtService`（linkservice.transport） | 未导出 | — |
| `IpcBtService`（accessory.connectivity） | 未导出 | — |
| `IWearableService`（AIDL） | 仅进程内 / 同签名可达 | signature |

**接口存在 ≠ 第三方可调。** 细节见 `docs/ipc.md`。

---

## Health Connect 调查

详见独立报告：`docs/health-connect.md`。

**结论：在所测设备上，Watch S → OHealth → Health Connect 这条数据链不存在。** 第三方 App 本身可以在其他地方使用 Health Connect（经用户授权）——但这里 HC 里没有任何手表数据可读。

---

## 已排除的方案

完整清单与证据见 `docs/rejected-approaches.md`：

1. Health Connect 路线 —— ❌（无源数据）
2. `action_data_refresh` 广播 —— ❌ 仅本地刷新
3. `ACTION_SLEEP_STAT_REFRESH` —— ❌ 本地/卡片逻辑
4. `action_STEP_GOAL` / `SLEEP_REMIND_ACTION` / `PhoneSleepMeasure.notify` —— ❌ 无可观测手表动作
5. 直接绑定 `WearableServer` —— ❌ signature 权限
6. UUID 猜测（FTMS / NUS-like / A5xx / A6xx / A7xx / AA15 / BB15）—— ❌ 不可靠；均非 Watch S 协议

---

## 可能的集成路径

| 路径 | 可行性 | 说明 |
|------|--------|------|
| 直连 OHealth IPC | ❌ | signature 锁死 |
| Bridge APK → OHealth IPC | ❌ | 任何非 OPPO 签名的 App 都受同一 signature 限制 |
| 自建手表传输通道 | ⚠️ 不切实际 | 需要重实现配件/链路协商 + 密钥材料；未尝试 |
| **UI 自动化**（无障碍 / 自动点击 OHealth 界面） | ✅ 可行 | **间接、仅界面层——不是 API 接入**；驱动 OHealth 自己的界面（无障碍/点击），不触碰协议 |
| **OHealth 数据导出**（用户手动）+ 文件读取 | ✅ 可行 | 离线快照 |
| OHealth 云端（web 端点） | ❓ 未知 | 未研究 |

---

## 复现

完整实验日志、使用过的命令（baksmali 流水线、Shizuku/adb shell 观察）与原始证据指引见 `docs/experiments.md` 与 `research/timeline.md`。本仓库不包含任何专有 APK、反编译源码或密钥材料。

---

## 局限

- 结论 **与具体配置相关**：所测 ColorOS 版本上的 OHealth v1.0.3。其他版本可能有差异。
- 协议还原 **以静态为主**，能动态验证的地方做了动态验证（logcat）。部分负载语义标注为 LIKELY/UNKNOWN。
- 未进行 BLE 层抓包；物理链路仅在操作系统蓝牙层面观察。
- 未攻击加密；未绕过任何签名。

## 免责声明

本项目仅用于 **互操作性研究与教育目的**。不含专有代码、不含 APK 文件、不含反编译源码、不含凭证、不含个人数据。所有设备地址、账号标识与私有路径均已脱敏。请勿用于违反任何服务条款或当地法律的行为。

## 许可证

MIT —— 见 `LICENSE`。
