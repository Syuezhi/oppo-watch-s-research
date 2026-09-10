# Transport Layer & Security Model

Scope: the `com.oplus.wearable.linkservice` stack inside OHealth (classes5.dex), plus the app-side message bus (`BTClient`, classes13). Status: `CONFIRMED` static unless marked.

---

## 1. Layers

```
BTClient                     (classes13, app-side bus: queue + response map + listener dispatch)
    │
MessageTransferManager       (classes5, com.oplus.wearable.linkservice.message)
    │   sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)  [L274]
    │   handleMessage(ModuleInfo, [B)                                       [L96]
    │
TransportLayerV1Wrapper      (classes5, dataprocessor/wrap) — encryption wrap point
ClientConsultHelper          (classes5, transport/consult) — negotiation-time crypto
    │
SecurityManager              (classes5, security)
    │   encrypt([B,[B,String) / encrypt([B,[B,[B,String)  [L312/L337]
    │   decrypt([B,[B,String) / decrypt([B,[B,[B,String)  [L151/L176]
    │   generatorKey(J,J,String) [L473] · queryKey/saveKey/clearKey [L677/775/108]
    │
com.oplus.wearable.crypto.Cipher.getInstance(String) → ICipher.encrypt/decrypt([B,[B,[B)
    │
WearableServer / IWearableService (AIDL) / IpcBtService   (see ipc.md)
```

## 2. Encryption Model

- `MessageEvent.mEncryptOption` (int), constants:
  - `ENCRYPT_DEVICE_DEFAULT = 0` (constructor default)
  - `ENCRYPT_NO = 1`
  - `ENCRYPT_YES = 2`
  - `ENCRYPT_UNCOMPRESS = 3`
- **In the entire recovered codebase, only two business areas explicitly call `setEncryptOption`:**
  - `OTAUpdateManager` (classes13; 5 call sites)
  - `EsimSubManager` (classes13; 1 call site)
- All other businesses leave the default (`0`), i.e. encryption is decided by transport-side policy — **not** by the business layer.
- Actual `encrypt`/`decrypt` invocation sites found:
  - `TransportLayerV1Wrapper`: L477 (encrypt), L1168/L1431 (decrypt)
  - `ClientConsultHelper`: L2359/L2848 (encrypt), L738/L983/L1499 (decrypt)
- **No key material was extracted, and no cipher was broken.** Key APIs (`saveKey`, `queryKey`, `generatorKey`) manage per-device key material; their storage medium was not pursued.

## 3. Request / Response Bookkeeping (app side)

- Send: `BTClient.sendMsg(MessageEvent[, MsgCallback])` → wraps into `CallbackMsg` → `sendMsgQueue` (LinkedBlockingQueue) → `checkSendMsgThread()` → `sendMsgInner()` (logs `"Send Msg:"`).
- Wait: `addToWaitResponse(CallbackMsg)` → `waitResponseMsgMap` keyed by `messageKey()` (string concat `serviceId + commandId`), value = FIFO `LinkedList<TimeoutMsg>`; a timeout `Runnable` is posted via `mainThreadHandler.postDelayed(waitResponseTimeout)`.
- Respond: `dispatchMsgResponse(mac, MessageEvent)` looks up the same key, `removeFirst()`, cancels the timeout, posts the callback.
- Timeout / failure hooks: `onSendMsgFail`, `TimeoutMsg.run`.

## 4. Physical Link (observed, not decoded)

- On the test device the watch was connected over **classic Bluetooth (BR/EDR)** (via `dumpsys bluetooth_manager`: `ACL BR/EDR:Y LE:N`).
- The BT stack recorded ongoing but non-continuous traffic to the watch; message cadence was not captured at the HCI layer.
- **No BLE GATT capture was performed.** Any statement about GATT characteristics, UUID-based protocols, or frame formats would be speculative — see `rejected-approaches.md`.

## 5. File Transfer Path (LIKELY)

Classes observed in classes5.dex: `file/FileTransferManager`, `file/transfer/FTCommandSender`, `FTTaskQueueManager`, `FileTransferReadSender`, `readwriter/DataSenderImpl`, `FDDataSenderImpl`, plus `FTSend` protobuf (`FTSendRequestResponse`). This matches the two file-level flows used by sport records and fitness data (see `protobuf.md`). Internal framing is `UNKNOWN`.

## 6. Limits

- Transport frame headers/TLV layout: `UNKNOWN`.
- Whether any given business payload is encrypted on the wire depends on transport policy that was not dynamically instrumented: `UNKNOWN` (business default = device default).
- The `WearableServer` ↔ system-level transport relationship (who ultimately drives the BT socket) was not fully traced beyond the classes listed above: `LIKELY` within `com.heytap.health:transport` process.