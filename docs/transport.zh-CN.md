# 传输层与安全模型

**简体中文 | [English](transport.md)**

范围：OHealth 内的 `com.oplus.wearable.linkservice` 栈（classes5.dex），加上应用侧消息总线（`BTClient`，classes13）。除标注外状态均为 `CONFIRMED` 静态。

---

## 1. 分层

```
BTClient                     (classes13, 应用侧总线: 队列 + 响应表 + 监听分发)
    │
MessageTransferManager       (classes5, com.oplus.wearable.linkservice.message)
    │   sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)  [L274]
    │   handleMessage(ModuleInfo, [B)                                       [L96]
    │
TransportLayerV1Wrapper      (classes5, dataprocessor/wrap) — 加密包装点
ClientConsultHelper          (classes5, transport/consult) — 协商期加密
    │
SecurityManager              (classes5, security)
    │   encrypt([B,[B,String) / encrypt([B,[B,[B,String)  [L312/L337]
    │   decrypt([B,[B,String) / decrypt([B,[B,[B,String)  [L151/L176]
    │   generatorKey(J,J,String) [L473] · queryKey/saveKey/clearKey [L677/775/108]
    │
com.oplus.wearable.crypto.Cipher.getInstance(String) → ICipher.encrypt/decrypt([B,[B,[B)
    │
WearableServer / IWearableService (AIDL) / IpcBtService   （见 ipc.md）
```

## 2. 加密模型

- `MessageEvent.mEncryptOption`（int）常量：
  - `ENCRYPT_DEVICE_DEFAULT = 0`（构造函数默认）
  - `ENCRYPT_NO = 1`
  - `ENCRYPT_YES = 2`
  - `ENCRYPT_UNCOMPRESS = 3`
- **在还原出的全部代码中，只有两个业务域显式调用 `setEncryptOption`：**
  - `OTAUpdateManager`（classes13；5 处调用点）
  - `EsimSubManager`（classes13；1 处调用点）
- 其余业务全部保持默认值（`0`），即加密与否由传输侧策略决定——**不是**业务层决定。
- 实际 `encrypt`/`decrypt` 调用点：
  - `TransportLayerV1Wrapper`：L477（加密）、L1168/L1431（解密）
  - `ClientConsultHelper`：L2359/L2848（加密）、L738/L983/L1499（解密）
- **未提取任何密钥材料，未破解任何加密。** 密钥 API（`saveKey`、`queryKey`、`generatorKey`）管理按设备生成的密钥材料；其存储介质未追查。

## 3. 请求 / 响应记账（应用侧）

- 发送：`BTClient.sendMsg(MessageEvent[, MsgCallback])` → 包装成 `CallbackMsg` → `sendMsgQueue`（LinkedBlockingQueue）→ `checkSendMsgThread()` → `sendMsgInner()`（打印 `"Send Msg:"` 日志）。
- 等待：`addToWaitResponse(CallbackMsg)` → `waitResponseMsgMap` 以 `messageKey()`（字符串拼接 `serviceId + commandId`）为键，值为 FIFO `LinkedList<TimeoutMsg>`；超时用 `mainThreadHandler.postDelayed(waitResponseTimeout)` 投递。
- 响应：`dispatchMsgResponse(mac, MessageEvent)` 查同一键，`removeFirst()`，取消超时，投递回调。
- 超时/失败钩子：`onSendMsgFail`、`TimeoutMsg.run`。

## 4. 物理链路（观测，未解码）

- 测试机上手表经**经典蓝牙（BR/EDR）**连接（经 `dumpsys bluetooth_manager`：`ACL BR/EDR:Y LE:N`）。
- 蓝牙栈记录到与手表之间持续但非连续的数据往来；消息节奏未在 HCI 层捕获。
- **未做 BLE GATT 抓包。** 关于 GATT characteristics、基于 UUID 的协议、帧格式的任何陈述都将是推测——见 `rejected-approaches.md`。

## 5. 文件传输路径（LIKELY）

classes5.dex 中观察到的类：`file/FileTransferManager`、`file/transfer/FTCommandSender`、`FTTaskQueueManager`、`FileTransferReadSender`、`readwriter/DataSenderImpl`、`FDDataSenderImpl`，以及 `FTSend` protobuf（`FTSendRequestResponse`）。与运动记录和健身数据使用的两条文件级流程吻合（见 `protobuf.md`）。内部分帧方式为 `UNKNOWN`。

## 6. 限制

- 传输帧头/TLV 布局：`UNKNOWN`。
- 任一业务负载在线上是否加密，取决于未做动态插桩的传输策略：`UNKNOWN`（业务默认 = 设备默认）。
- `WearableServer` ↔ 系统级传输层的关系（最终谁驱动 BT socket）未在上述类之外完整追踪：`LIKELY` 位于 `com.heytap.health:transport` 进程内。