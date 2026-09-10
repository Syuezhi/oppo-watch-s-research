# Rejected Approaches — With Evidence

Every item below was tried or evaluated and then **rejected**. Listed so nobody re-walks the same dead ends.

---

## 1. Health Connect route

**Result: ❌ Not viable on tested configuration.**

Evidence:
- OHealth declares **0** `android.permission.health.*` permissions (dumpsys).
- `com.oplus.healthservice` (v16.2.10) contains **0** Health Connect references (dex scan).
- All 17 OHealth dex files contain **0** `HealthConnect` references.
- Live HC UI: 0 apps with access, no recent access.
- No writes observed during monitoring.

→ `docs/health-connect.md`

---

## 2. OHealth broadcast `com.heytap.health.action_data_refresh`

**Result: ❌ Local refresh only.**

Evidence: receiver delivered; downstream = step-medal check + `SportHealthDataAPI.readSportHealthData` (local DB); no BT/transport/sendMsg activity within observation window.

---

## 3. Broadcast `com.heytap.health.sleep.ACTION_SLEEP_STAT_REFRESH`

**Result: ❌ Local / smart-brain logic.**

Evidence: receiver branch (`HealthSeedCardReceiver`) → `sendSleepDataTOSmartBrain` + `updateSleepSportData`. Requires `hasCurDaySleepData` extra to do anything; nothing Watch-related.

---

## 4. Broadcast `com.heytap.health.action_STEP_GOAL`

**Result: ❌ No Watch sync observed.**

---

## 5. Broadcast `SLEEP_REMIND_ACTION`

**Result: ❌ No Watch sync observed.**

---

## 6. Broadcast `com.heytap.health.PhoneSleepMeasure.notify`

**Result: ❌ No Watch sync observed.**

---

## 7. Direct `WearableServer` binding from third-party app

**Result: ❌ Signature permission.**

Evidence: `com.heytap.wearable.linkservice.permission.WEARABLE` → `prot=signature` (dumpsys). Non-OPPO-signed apps cannot bind.

---

## 8. Direct linkservice / AIDL calls

**Result: ❌ No signature available.**

`IWearableService` is reachable only through the signature-locked server (or in-process); a bridge APK hits the same wall.

---

## 9. UUID-based guessing of the Watch BLE protocol

**Result: ❌ Unreliable; not Watch S protocol.**

Explicitly flagged as **not** Watch S evidence:
- `FTMS` (`00001826` …) — represents Fitness Machine Service (treadmill context), not the watch.
- treadmill-related UUIDs (e.g. `b1e73412-…`, `8a998074-…` — FTMS extended chars, treadmill domain).
- NUS-like group `6E4000xx` — no GATT call-path evidence on this device.
- `A5xx` / `A6xx` / `A7xx` clusters — string presence only; no invocation chain.
- `AA15` / `BB15` — accessory-framework constants (`com.heytap.accessory.connectivity.params`), not shown to carry Watch messages.
- `a49eaa15-…` / `a49ebb15-…` — accessory pairing constants; not proven to be Watch S business channels.

Rule adopted: **UUID in string pool ≠ protocol. Only an actual GATT/Binder call chain counts, and only dynamic verification confirms.** None of the above survived that bar for Watch S.

---

## Cross-cutting: what was NOT attempted (by design)

- No crypto breaking, no key extraction.
- No signature bypass, no root injection, no APK patching.
- No BLE-layer attack on the watch.
- No account/cloud credential misuse.

These remain outside scope regardless of difficulty.