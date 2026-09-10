# 架构 — Watch S ↔ OHealth 通信

**简体中文 | [English](architecture.md)**

**证据等级**：`CONFIRMED` = 静态 smali 分析 + 跨 dex 交叉验证和/或动态确认；`LIKELY` = 强静态证据，未完全验证；`UNKNOWN` = 无法结论。

---

## 1. 系统总览

```
OPPO Watch S (OWWE262)
        │  私有链路（测试机实测：经典蓝牙 BR/EDR）
        ▼
OHealth  com.heytap.health
        │
        ├─ BTClient (com.heytap.health.devicemanager.btclient)      [classes13]
        │
        ├─ MessageTransferManager (com.oplus.wearable.linkservice.message)  [classes5]
        │
        ├─ linkservice 栈 (com.oplus.wearable.linkservice.*)     [classes5]
        │     WearableServer / WearableApiManager / IWearableService
        │     transport 包装层 (TransportLayerV1Wrapper, ClientConsultHelper)
        │     security (SecurityManager → com.oplus.wearable.crypto.Cipher)
        │
        ▼
Watch S
```

测试机上观察到的进程：`com.heytap.health`（主进程）、`com.heytap.health:transport`、`com.heytap.health:once`。`CONFIRMED`

---

## 2. 发送管线（手机 → 手表）

| # | 步骤 | 类 · 证据 |
|---|------|-----------|
| 1 | 业务层构建 protobuf，`XxxProto.newBuilder()…build().toByteArray()` | 如 `SleepStateMsgProcessor` c12 |
| 2 | 包装为 `new MessageEvent(serviceId, commandId, data)` | `MessageEvent.smali`（classes5）L123；构造函数 `(II[B)` 默认值：encryptOption=0、cacheOption=1、transport=2（BT） |
| 3 | `BTClient.sendMsg(MessageEvent)`（也有带 `MsgCallback` 的重载） | classes13 `BTClient.smali` L1477 / L1505 → L1436（入队）→ L1520 `sendMsgInner` |
| 4 | 队列：`sendMsgQueue`（LinkedBlockingQueue）+ `checkSendMsgThread` 调度线程 | classes13 `BTClient.smali` L1455–1461 |
| 5 | `MessageTransferManager.sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)` | classes5 `MessageTransferManager.smali` L274 |
| 6 | 传输层施加可选加密（`TransportLayerV1Wrapper.encrypt` → `SecurityManager.encrypt`） | classes5 `TransportLayerV1Wrapper.smali` L477；`SecurityManager.smali` L312/L337 |
| 7 | 物理链路 | 经典蓝牙（测试机 `dumpsys bluetooth_manager` 观察到 BR/EDR 已连接） |

**请求/响应配对**（`CONFIRMED`）：响应在 `BTClient.dispatchMsgResponse` 中通过 `waitResponseMsgMap` 匹配，键 = `String.valueOf(serviceId) + commandId`（字符串拼接，无分隔符），同键 FIFO，超时通过 `postDelayed` 投递（`addToWaitResponse` L538；`messageKey` L942；`dispatchMsgResponse` L715）。

---

## 3. 接收管线（手表 → 手机）

| # | 步骤 | 类 · 证据 |
|---|------|-----------|
| 1 | 传输层投递消息 | → `BTClient.onMessageReceived(mac, MessageEvent)` classes13 L981 |
| 2 | 响应匹配（若有调用方在等待） | `dispatchMsgResponse` L715 → 经 `mainThreadHandler` 回调 |
| 3 | 广播给已注册监听器 | `dispatchToListener` L810 → `ReceiveMsgListener` 列表 |
| 4 | 业务分发 | `MsgProcessor.onMessageReceived(mac, MessageEvent)`（classes12 `com.heytap.device.data.sporthealth.receive.*`） |
| 5 | 负载解码 | `event.getData()` → `[B` → `XxxProto.parseFrom([B)` |

**处理器注册**（`CONFIRMED`）：`ReceiveMsgManager.registerMsgListener()`（classes12，L450）遍历处理器，读取 `acceptMsgTypes()` → `MsgType.create(sid, cid)` → 通过 `BTClient.addMsgListener` 注册。

观察到的处理器示例：`DNDMsgProcessor` (1,0x6d)、`SleepStateMsgProcessor` (5,0x20)、`Spo2ManualMsgProcessor` (5,0x19)、`WeightMsgProcessor` (5,0x42)、`WearRecordMsgProcessor` (5,0x40)、`DailyActivityStatisticsProcessor` (5,0x8b)、`MenstrualCycleMsgProcessor` (5,0xf8)、`SnoreActiveStateProcessor` (5,0xcb)、`OsaResultDaysProcessor` (5,0xc8) ……（完整列表见 `message-codes.md`）。

---

## 4. 关键类

| 类 | Dex | 角色 |
|-------|-----|------|
| `com.heytap.health.devicemanager.btclient.BTClient` | classes13 | 应用侧消息总线：发送队列、响应表、监听分发 |
| `com.oplus.wearable.linkservice.sdk.common.MessageEvent` | classes5 | 线上消息：(serviceId, commandId, data) + 选项 |
| `com.oplus.wearable.linkservice.message.MessageTransferManager` | classes5 | 传输侧收发管理器 |
| `com.oplus.wearable.linkservice.WearableServer` | classes5 | 对外暴露的服务（filter `…action.WEARABLE`；signature 权限） |
| `com.oplus.wearable.linkservice.WearableApiManager` | classes5 | 客户端 API 管理器；`getIWearableServiceBnInterface()` 返回本地 `IWearableService.Stub` |
| `com.oplus.wearable.linkservice.sdk.IWearableService` | classes5 | AIDL 接口面（sendMessage、connect、bond 操作、文件操作……） |
| `TransportLayerV1Wrapper` / `ClientConsultHelper` | classes5 | 加密包装点（`SecurityManager.encrypt/decrypt`） |
| `com.oplus.wearable.linkservice.security.SecurityManager` | classes5 | `encrypt/decrypt/generatorKey/queryKey/saveKey` —— 密钥按设备生成 |
| `MsgProcessor` 家族 | classes12 | 业务消息处理器（接收侧） |
| `BTClient$CallbackMsg`、`$TimeoutMsg` | classes13 | 请求/响应记账 |

---

## 5. 未确认项

- 传输包装之后的精确字节布局（帧头、TLV）—— `UNKNOWN`
- 物理传输选择逻辑（RFCOMM 还是其他）—— `UNKNOWN`（仅观察到 BR/EDR 存在）
- 手表侧实现 —— 超出范围，`UNKNOWN`
- OTA/eSIM 之外各业务类型是否适用加密 —— `LIKELY` 默认不加密（encryptOption=0 默认值），未经线上抓包验证
