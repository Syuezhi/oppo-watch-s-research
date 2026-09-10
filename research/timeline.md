# Research Timeline (Phase 1–10)

Condensed log of the investigation that produced this repository. Dates relative to the study window (September 2026).

---

## Phase 1–2 — Orientation & false starts
- Initial target: find "watch" related data paths in OHealth.
- Early lead on `GM_CHARACTERISTIC` → **corrected**: belongs to `com.omron.ar` (glucose meter domain). Lesson: don't trust names.
- Toolchain established: APK pull → `unzip classes*.dex` → **baksmali** (Ubuntu/proot; deps: dexlib2/util/guava/jcommander). jadx abandoned on-device (OOM).

## Phase 3–4 — Real BLE stack vs. watch stack
- Located the generic BLE framework: `com.heytap.health.ble.{GattChannel, BleClient, DispatchCenter, ValueChangedCallback}` — full CCCD/notify/write chains recovered.
- Confirmed its business consumers were **third-party fitness devices** (treadmill etc.), *not* the watch.
- Watch path diverges into `com.oplus.wearable.linkservice` + `com.heytap.health.devicemanager`.

## Phase 5 — SDK/transport localization
- `WearableServer` + `IpcBtService` found declared in OHealth manifest.
- `IWearableService` (AIDL) surface enumerated.
- `ConnectManager → linkservice.sdk.Node` = the watch-facing edge of OHealth.

## Phase 6 — Message protocol recovery
- `MessageEvent(II[B)` structure + constants (`ENCRYPT_*`, `TRANSPORT_BT`, …).
- `BTClient.sendMsg` / `MessageTransferManager.sendMessage` / receive chain via `MsgProcessor`.
- Cross-dex sweeps: 55 `sendMsg` call sites, 191 receive-side files in `sporthealth/receive`.

## Phase 7 — Callable-level commands
- Six commands fully reverse-engineered with protobuf field maps (HR stats, Sport stats, Sport list, Sport record, Fitness, Daily activity).
- `FitnessProto`/`FileProto` located in `classes14.dex`; `*_FIELD_NUMBER` constants extracted.

## Phase 8 — IPC boundary
- `WearableServer`: filter `…action.WEARABLE`, permission `…permission.WEARABLE` → **signature** (dumpsys).
- `IpcBtService` ×2: not exported. `com.heytap.accessory`: `/product/priv-app/AccessoryFramework`.
- Direct IPC / bridge APK IPC: **rejected**.

## Phase 9 — Dynamic trigger hunt (broadcasts)
- Five OHealth broadcasts tested under a clean logcat protocol.
- `action_data_refresh`: delivered → **local refresh only** (medal check + provider read). Others: no observable action.
- Verdict: no exported broadcast triggers watch sync.

## Phase 10 — Health Connect audit
- HC components present; **zero** HC permissions declared anywhere; **zero** HC refs in OHealth & healthservice dex.
- Live HC UI: 0 apps with access; no recent access.
- **Grade C** for the HC route.

---

## Final State

| Layer | Result |
|-------|--------|
| Protocol | ⭐ Recovered (documented here) |
| Channel | 🔒 Signature-locked |
| Data egress (HC) | 🚫 Not present |
| Remaining live routes | UI automation / user-driven export / cloud (unresearched) |