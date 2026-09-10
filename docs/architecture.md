# Architecture — Watch S ↔ OHealth Communication

**Evidence levels**: `CONFIRMED` = static smali analysis with cross-dex verification and/or dynamic confirmation; `LIKELY` = strong static evidence, not fully verified; `UNKNOWN` = inconclusive.

---

## 1. System Overview

```
OPPO Watch S (OWWE262)
        │  proprietary link (classic BT / BR-EDR observed on test device)
        ▼
OHealth  com.heytap.health
        │
        ├─ BTClient (com.heytap.health.devicemanager.btclient)      [classes13]
        │
        ├─ MessageTransferManager (com.oplus.wearable.linkservice.message)  [classes5]
        │
        ├─ linkservice stack (com.oplus.wearable.linkservice.*)     [classes5]
        │     WearableServer / WearableApiManager / IWearableService
        │     transport wrappers (TransportLayerV1Wrapper, ClientConsultHelper)
        │     security (SecurityManager → com.oplus.wearable.crypto.Cipher)
        │
        ▼
Watch S
```

Processes observed on test device: `com.heytap.health` (main), `com.heytap.health:transport`, `com.heytap.health:once`. `CONFIRMED`

---

## 2. Send Pipeline (phone → watch)

| # | Step | Class · evidence |
|---|------|------------------|
| 1 | Business layer builds protobuf, `XxxProto.newBuilder()…build().toByteArray()` | e.g. `SleepStateMsgProcessor` c12 |
| 2 | Wrap in `new MessageEvent(serviceId, commandId, data)` | `MessageEvent.smali` (classes5) L123; constructor `(II[B)` defaults: encryptOption=0, cacheOption=1, transport=2 (BT) |
| 3 | `BTClient.sendMsg(MessageEvent)` (also overloads with `MsgCallback`) | classes13 `BTClient.smali` L1477 / L1505 → L1436 (queue) → L1520 `sendMsgInner` |
| 4 | Queue: `sendMsgQueue` (LinkedBlockingQueue) + `checkSendMsgThread` dispatcher | classes13 `BTClient.smali` L1455–1461 |
| 5 | `MessageTransferManager.sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)` | classes5 `MessageTransferManager.smali` L274 |
| 6 | Transport layer applies optional encryption (`TransportLayerV1Wrapper.encrypt` → `SecurityManager.encrypt`) | classes5 `TransportLayerV1Wrapper.smali` L477; `SecurityManager.smali` L312/L337 |
| 7 | Physical link | classic BT (BR/EDR connected observed via `dumpsys bluetooth_manager` on test device) |

**Request/response correlation** (`CONFIRMED`): responses are matched in `BTClient.dispatchMsgResponse` using `waitResponseMsgMap`, key = `String.valueOf(serviceId) + commandId` (string concat, no separator), FIFO per key, with a timeout posted via `postDelayed` (`addToWaitResponse` L538; `messageKey` L942; `dispatchMsgResponse` L715).

---

## 3. Receive Pipeline (watch → phone)

| # | Step | Class · evidence |
|---|------|------------------|
| 1 | Transport delivers message | → `BTClient.onMessageReceived(mac, MessageEvent)` classes13 L981 |
| 2 | Response matching (if a caller waits) | `dispatchMsgResponse` L715 → callback via `mainThreadHandler` |
| 3 | Broadcast to registered listeners | `dispatchToListener` L810 → `ReceiveMsgListener` list |
| 4 | Business dispatch | `MsgProcessor.onMessageReceived(mac, MessageEvent)` (classes12 `com.heytap.device.data.sporthealth.receive.*`) |
| 5 | Payload decode | `event.getData()` → `[B` → `XxxProto.parseFrom([B)` |

**Processor registration** (`CONFIRMED`): `ReceiveMsgManager.registerMsgListener()` (classes12, L450) iterates processors, reads `acceptMsgTypes()` → `MsgType.create(sid, cid)` → registers via `BTClient.addMsgListener`.

Observed processor examples: `DNDMsgProcessor` (1,0x6d), `SleepStateMsgProcessor` (5,0x20), `Spo2ManualMsgProcessor` (5,0x19), `WeightMsgProcessor` (5,0x42), `WearRecordMsgProcessor` (5,0x40), `DailyActivityStatisticsProcessor` (5,0x8b), `MenstrualCycleMsgProcessor` (5,0xf8), `SnoreActiveStateProcessor` (5,0xcb), `OsaResultDaysProcessor` (5,0xc8) … (full list in `message-codes.md`).

---

## 4. Key Classes

| Class | Dex | Role |
|-------|-----|------|
| `com.heytap.health.devicemanager.btclient.BTClient` | classes13 | App-side message bus: send queue, response map, listener dispatch |
| `com.oplus.wearable.linkservice.sdk.common.MessageEvent` | classes5 | Wire message: (serviceId, commandId, data) + options |
| `com.oplus.wearable.linkservice.message.MessageTransferManager` | classes5 | Transport-side send/receive manager |
| `com.oplus.wearable.linkservice.WearableServer` | classes5 | Exposed service (filter `…action.WEARABLE`; signature permission) |
| `com.oplus.wearable.linkservice.WearableApiManager` | classes5 | Client-side API manager; `getIWearableServiceBnInterface()` returns the local `IWearableService.Stub` |
| `com.oplus.wearable.linkservice.sdk.IWearableService` | classes5 | AIDL surface (sendMessage, connect, bond ops, file ops…) |
| `TransportLayerV1Wrapper` / `ClientConsultHelper` | classes5 | Encryption wrap points (`SecurityManager.encrypt/decrypt`) |
| `com.oplus.wearable.linkservice.security.SecurityManager` | classes5 | `encrypt/decrypt/generatorKey/queryKey/saveKey` — keys are per-device |
| `MsgProcessor` family | classes12 | Business message handlers (receive side) |
| `BTClient$CallbackMsg`, `$TimeoutMsg` | classes13 | Request/response bookkeeping |

---

## 5. What is NOT confirmed

- Exact byte layout after transport wrapping (frame headers, TLV) — `UNKNOWN`
- Physical transport selection logic (RFCOMM vs other) — `UNKNOWN` (only BR/EDR presence observed)
- Watch-side implementation — out of scope, `UNKNOWN`
- Encryption applicability per business type beyond OTA/eSIM — `LIKELY` unencrypted by default (encryptOption=0 default), not wire-verified
