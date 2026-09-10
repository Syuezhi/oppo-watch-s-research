# Protocol Overview

The Watch S message protocol as carried by OHealth's linkservice stack. This page is the index; details live in the linked documents.

---

## 1. Message Unit

Every business message is a `MessageEvent`:

```
MessageEvent(serviceId: int, commandId: int, data: byte[])
```

| field | type | default | meaning |
|-------|------|---------|---------|
| `mServiceId` | int | — | business domain (see `message-codes.md`) |
| `mCommandId` | int | — | command within domain |
| `mData` | byte[] | — | protobuf payload (`XxxProto.toByteArray()`), or file-channel related data |
| `mEncryptOption` | int | 0 = device default | 1=no, 2=yes, 3=yes-uncompressed |
| `mCacheOption` | int | 1 = default | 0x10 = not cached |
| `mPriority` | int | middle | |
| `mSeq` | int | 0 | sequence |
| `transport` | int | 2 = BT | 1 = WiFi |

Source: `com.oplus.wearable.linkservice.sdk.common.MessageEvent` (classes5.dex).

## 2. Two Pipelines

**Phone → Watch**
```
business → protobuf.toByteArray() → new MessageEvent(sid, cid, data)
        → BTClient.sendMsg() → send queue → MessageTransferManager.sendMessage()
        → transport (optional encryption) → watch
```
**Watch → Phone**
```
transport → BTClient.onMessageReceived() → dispatchMsgResponse (callback match)
        → dispatchToListener → ReceiveMsgListener / MsgProcessor.onMessageReceived()
        → event.getData()[B → XxxProto.parseFrom()
```
Full detail: `architecture.md`, `transport.md`.

## 3. Registration & Routing

- Handlers implement `MsgProcessor` (`onMessageReceived(mac, MessageEvent)`), declaring their ids via `acceptMsgTypes() → MsgType.create(sid, cid)`.
- `ReceiveMsgManager.registerMsgListener()` (classes12, L450) registers all processors through `BTClient.addMsgListener(...)`.
- Response correlation: `waitResponseMsgMap` keyed by `String.valueOf(serviceId) + commandId`; FIFO; timeout via `postDelayed`.

## 4. Command Families

See `message-codes.md` for the complete dictionary. Domains:

| svc | domain | sample commands |
|-----|--------|-----------------|
| 0x01 | DND / account / display | 0x6d DND, 0xa7 account ticket |
| 0x04 | workout devices | 0x0e pressure, 0x3e intensity, 0x01 record list |
| 0x05 | health data (main) | 0x20 sleep, 0x19 SpO2, 0x35 HR stats, 0x8b daily activity |
| 0x0d | watchface | 0x06–0x0c |
| 0x19 | location | AGPS, location client |
| 0x1a | file transfer | FileRequest-based |

## 5. Callable-Level Commands

Six commands with full request/response payload maps (see `protobuf.md`):

| svc | cmd | request | response |
|-----|-----|---------|----------|
| 0x05 | 0x35 | `TimeRangeRequest` | `HeartRateStatData` |
| 0x05 | 0x36 | `TimeRangeRequest`/`TypeRequest` | `SportStatData`/`SportStatList` |
| 0x04 | 0x01 | `TimeRangeRequest` | `FileNameData` |
| 0x04 | 0x02 | `StringRequest` | file channel |
| 0x04 | 0x09 | `TimeRangeRequest` | file channel |
| 0x05 | 0x8b | `SportStatList` (bidi) | `SportStatList` |

## 6. Evidence Standard

- Every id in these documents was recovered via smali register-backtracking and cross-validated between send-site constants and receiver registrations where both exist.
- Items that could not reach that bar are marked `UNKNOWN` and are not asserted anywhere in this repository.
- The channel is signature-locked for third-party apps; the protocol documentation is provided for interoperability research, not as a claim of practical reachability. See `ipc.md`.