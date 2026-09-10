# 协议总览

**简体中文 | [English](protocol.md)**

Watch S 消息协议在 OHealth linkservice 栈上的承载方式。本页是索引；细节在链接的各文档中。

---

## 1. 消息单元

每条业务消息都是一个 `MessageEvent`：

```
MessageEvent(serviceId: int, commandId: int, data: byte[])
```

| 字段 | 类型 | 默认值 | 含义 |
|-------|------|---------|---------|
| `mServiceId` | int | — | 业务域（见 `message-codes.md`） |
| `mCommandId` | int | — | 域内命令号 |
| `mData` | byte[] | — | protobuf 负载（`XxxProto.toByteArray()`），或文件通道相关数据 |
| `mEncryptOption` | int | 0 = 设备默认 | 1=不加密，2=加密，3=加密-不压缩 |
| `mCacheOption` | int | 1 = 默认 | 0x10 = 不缓存 |
| `mPriority` | int | middle | |
| `mSeq` | int | 0 | 序列号 |
| `transport` | int | 2 = BT | 1 = WiFi |

来源：`com.oplus.wearable.linkservice.sdk.common.MessageEvent`（classes5.dex）。

## 2. 两条管线

**手机 → 手表**
```
业务 → protobuf.toByteArray() → new MessageEvent(sid, cid, data)
        → BTClient.sendMsg() → 发送队列 → MessageTransferManager.sendMessage()
        → transport（可选加密）→ 手表
```
**手表 → 手机**
```
transport → BTClient.onMessageReceived() → dispatchMsgResponse（回调匹配）
        → dispatchToListener → ReceiveMsgListener / MsgProcessor.onMessageReceived()
        → event.getData()[B → XxxProto.parseFrom()
```
完整细节：`architecture.md`、`transport.md`。

## 3. 注册与路由

- 处理器实现 `MsgProcessor`（`onMessageReceived(mac, MessageEvent)`），通过 `acceptMsgTypes() → MsgType.create(sid, cid)` 声明自己的 id。
- `ReceiveMsgManager.registerMsgListener()`（classes12，L450）把所有处理器经 `BTClient.addMsgListener(...)` 注册进去。
- 响应配对：`waitResponseMsgMap` 以 `String.valueOf(serviceId) + commandId` 为键；FIFO；超时经 `postDelayed`。

## 4. 命令族

完整字典见 `message-codes.md`。域如下：

| svc | 域 | 命令示例 |
|-----|--------|-----------------|
| 0x01 | 勿扰 / 账户 / 显示 | 0x6d DND、0xa7 账户票据 |
| 0x04 | 运动器械 | 0x0e 气压、0x3e 强度、0x01 记录列表 |
| 0x05 | 健康数据（主域） | 0x20 睡眠、0x19 血氧、0x35 心率统计、0x8b 日常活动 |
| 0x0d | 表盘 | 0x06–0x0c |
| 0x19 | 定位 | AGPS、location client |
| 0x1a | 文件传输 | 基于 FileRequest |

## 5. 可调用级命令

六条命令具备完整请求/响应负载映射（见 `protobuf.md`）：

| svc | cmd | 请求 | 响应 |
|-----|-----|---------|----------|
| 0x05 | 0x35 | `TimeRangeRequest` | `HeartRateStatData` |
| 0x05 | 0x36 | `TimeRangeRequest`/`TypeRequest` | `SportStatData`/`SportStatList` |
| 0x04 | 0x01 | `TimeRangeRequest` | `FileNameData` |
| 0x04 | 0x02 | `StringRequest` | 文件通道 |
| 0x04 | 0x09 | `TimeRangeRequest` | 文件通道 |
| 0x05 | 0x8b | `SportStatList`（双向） | `SportStatList` |

## 6. 证据标准

- 本文档中的每个 id 都通过 smali 寄存器回溯还原，并在同时存在发送点常量与接收端注册时做了双向交叉验证。
- 未达到该标准的条目一律标记 `UNKNOWN`，且不在本仓库任何位置断言。
- 通道对第三方 App 已签名锁死；协议文档用于互操作性研究，不构成"实际可达"的主张。见 `ipc.md`。
