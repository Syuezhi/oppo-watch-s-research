# OPPO Watch S Reverse Engineering & Integration Research

Research notes on the communication architecture between **OPPO Watch S (OWWE262)** and **OHealth (com.heytap.health)** — protocol recovery, IPC permission analysis, and integration-path evaluation for third-party apps.

---

## Status

| Area | Result |
|------|--------|
| Protocol layer (serviceId/commandId, MessageEvent, protobuf) | **Largely recovered** (smali register-backtracking + cross-verified) |
| Transport/channel layer for third-party apps | **Blocked** — signature-level permission (`com.heytap.wearable.linkservice.permission.WEARABLE: signature`) |
| Health Connect route | **Not applicable on tested device** — OHealth writes no Watch data to HC |
| OHealth broadcast triggers | **Local refresh only** — no Watch sync triggered |
| **Overall grade for OHealth→HealthConnect→3rd-party route** | **C — not viable on tested configuration** |

---

## Tested Device

- **Watch**: OPPO Watch S, model **OWWE262** (paired, connected, classic BT)
- **Phone**: OPPO / ColorOS device (Android with the HealthFitness APEX module present)
- **OHealth**: `com.heytap.health` v1.0.3 (APK analyzed: 17 dex, ~124 MB)
- **Tools**: baksmali (Ubuntu/proot), smali register-backtracking scripts, Shizuku shell for dynamic tests

---

## Executive Summary

1. **The Watch speaks a proprietary message protocol over OHealth's `linkservice` stack**, not through any standard GATT profile. Messages are `MessageEvent(serviceId, commandId, byte[])` carrying protobuf payloads, with a callback-based request/response mechanism (`waitResponseMsgMap`, key = serviceId+commandId concat).
2. **~40+ business commands were recovered** with full send/receive chains (heart-rate stats, sport stats, sport records, sleep, SpO2, weight, wearable record, menstrual cycle, snore/OSA, wrist temperature, watchface, AGPS, file transfer, etc.). See `docs/message-codes.md`.
3. **The channel is sealed for third-party apps.** OHealth's external entry (`WearableServer`, action `com.heytap.wearable.linkservice.action.WEARABLE`) is protected by a **signature-level permission**. All other candidates (`IpcBtService` ×2) are not exported. `IWearableService` exists and is rich, but *interface existence ≠ third-party callability*.
4. **Broadcast experiments failed to trigger Watch sync.** `com.heytap.health.action_data_refresh` reaches a receiver and causes a local data refresh (verified: `SportHealthDataAPI.readSportHealthData`), but no BLE/transport activity follows. Four other Health/Sleep broadcasts produced no observable Watch-side action.
5. **Health Connect is empty of Watch data on this device.** OHealth declares **zero** `android.permission.health.*` permissions; `com.oplus.healthservice` (v16.2.10) contains **zero** Health Connect references; all 17 OHealth dex files contain **zero** `HealthConnect` references; dynamic monitoring showed no writes. HC mechanism exists but has no Watch data source. **Grade C.**

---

## Architecture

```
OPPO Watch S (OWWE262)
        │  proprietary link (classic BT observed: BR/EDR connected)
        ▼
OHealth (com.heytap.health) — 3 processes: main / :transport / :once
        │
        ├── BTClient.sendMsg(MessageEvent)          [com.heytap.health.devicemanager.btclient]
        │        ▼
        ├── MessageTransferManager.sendMessage(ModuleInfo, mac, MessageEvent, IWearableCallback, int)
        │        ▼
        └── com.oplus.wearable.linkservice transport (WearableServer / IWearableService / IpcBtService)
                 ▼
            Watch S
```

Receive direction:

```
transport → BTClient.onMessageReceived → dispatchMsgResponse → dispatchToListener
→ ReceiveMsgListener → MsgProcessor.onMessageReceived(mac, MessageEvent)
→ event.getData()[B → XxxProto.parseFrom([B) → business layer
```

Details: `docs/architecture.md`, `docs/transport.md`.

---

## Protocol Findings

- `MessageEvent(II[B)`: `(serviceId, commandId, data)`; defaults: `encryptOption=0 (device default)`, `cacheOption=1`, `transport=2 (BT)`, `priority=middle`.
- Only **OTA** and **eSIM** payloads explicitly set `encryptOption`; encryption (when used) is applied in the transport layer (`TransportLayerV1Wrapper` → `SecurityManager.encrypt/decrypt` → `com.oplus.wearable.crypto.Cipher`). **No keys were extracted; no crypto was broken.**
- Full command dictionary: `docs/message-codes.md`.

---

## Message Protocol (callable-level commands)

| svc | cmd | Business | Request protobuf | Response protobuf |
|-----|-----|----------|------------------|-------------------|
| 0x05 | 0x35 | HR stats | `FitnessProto.TimeRangeRequest` | `HeartRateStatData` (+V2) |
| 0x05 | 0x36 | Sport stats | `TimeRangeRequest` / `TypeRequest` | `SportStatData` / `SportStatList` |
| 0x04 | 0x01 | Sport record list | `TimeRangeRequest` (supportTlv=2) | `FileNameData` |
| 0x04 | 0x02 | Sport record | `StringRequest` (sportId) | file transfer channel |
| 0x04 | 0x09 | Fitness | `TimeRangeRequest` | file transfer channel |
| 0x05 | 0x8b | Daily activity | `SportStatList` (bidirectional) | `SportStatList` |

Field-level detail: `docs/protobuf.md`.

---

## IPC Boundary (summary)

| Component | Exposure | Protection |
|-----------|----------|------------|
| `WearableServer` | filter `...action.WEARABLE` | `com.heytap.wearable.linkservice.permission.WEARABLE` → **signature** |
| `IpcBtService` (linkservice.transport) | not exported | — |
| `IpcBtService` (accessory.connectivity) | not exported | — |
| `IWearableService` (AIDL) | reachable only in-process / same-signature | signature |

**Interface existence ≠ third-party callability.** Details: `docs/ipc.md`.

---

## Health Connect Investigation

See dedicated report: `docs/health-connect.md`.

**Result: Watch S → OHealth → Health Connect does not exist on the tested device.** Third-party apps *can* use Health Connect in general (with user-granted permissions) — but there is no Watch data in HC to read here.

---

## Rejected Approaches

See `docs/rejected-approaches.md` for the full list with evidence:

1. Health Connect route — ❌ (no source data)
2. `action_data_refresh` broadcast — ❌ local refresh only
3. `ACTION_SLEEP_STAT_REFRESH` — ❌ local/card logic
4. `action_STEP_GOAL` / `SLEEP_REMIND_ACTION` / `PhoneSleepMeasure.notify` — ❌ no Watch action observed
5. Direct `WearableServer` binding — ❌ signature permission
6. UUID-guessing (FTMS / NUS-like / A5xx / A6xx / A7xx / AA15 / BB15) — ❌ unreliable; not Watch S protocol

---

## Possible Integration Paths

| Path | Viability | Notes |
|------|-----------|-------|
| Direct OHealth IPC | ❌ | signature-locked |
| Bridge APK → OHealth IPC | ❌ | same signature lock applies to any non-OPPO-signed app |
| Custom Watch transport | ⚠️ impractical | would require re-implementing accessory/link negotiation + key material; not attempted |
| **UI automation** (accessibility / automated taps over OHealth UI) | ✅ workable | no API attacks; operates the app's own UI |
| **OHealth data export** (user-driven) + file ingestion | ✅ workable | offline snapshot |
| OHealth cloud (web endpoints) | ❓ unknown | not researched |

---

## Reproduction

See `docs/experiments.md` and `research/timeline.md` for the full experiment log, commands used (baksmali pipeline, Shizuku/adb shell observations), and raw evidence pointers. No proprietary APKs, decompiled sources, or key material are included in this repository.

---

## Limitations

- Findings are **configuration-specific**: OHealth v1.0.3 on the tested ColorOS build. Other versions may differ.
- Protocol recovery is **static-first**, with dynamic verification where possible (logcat). Some payload semantics are marked LIKELY/UNKNOWN.
- No BLE-layer capture was performed; the physical link was observed only at the OS Bluetooth level.
- No crypto was attacked; no signatures bypassed.

## Disclaimer

This project is for **interoperability research and educational purposes only**. It contains no proprietary code, no APK files, no decompiled sources, no credentials, and no personal data. All device addresses, account identifiers, and private paths are redacted. Do not use this to violate any terms of service or local law.

## License

MIT — see `LICENSE`.
