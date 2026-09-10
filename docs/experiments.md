# Dynamic Experiments Log

All dynamic tests used **observation-only** tooling (shell/adb-equivalent via Shizuku on an own device). No system modification, no injection, no data destruction.

---

## Experiment A — Broadcast Trigger Tests

**Question**: can a third-party app trigger Watch sync through OHealth's exported broadcasts?

### A1. `com.heytap.health.action_data_refresh` (receiver: `HealthSeedCardReceiver`)

| Step | Observation |
|------|-------------|
| send | `am broadcast --include-stopped-packages -a com.heytap.health.action_data_refresh` → `Broadcast completed: result=0` (no permission denial) |
| +1s | `ActivityManager: Broadcasting: Intent {…}` — delivered |
| +1s | OHealth main process: `DFJ.MedalSecret … medal_check_DailyStepMedal` (step-medal check) |
| +1s | `SportHealthDataAPI: readSportHealthData table:1002` (startTime=endTime=now; 1 row read; errorCode:0) — **local provider read** |
| +0–60s | **No** BTClient / sendMsg / Data-Sync / transport activity. Watch BT counters unchanged in character. |

**Verdict: local refresh only.** Receivers reachable and functional; no Watch-side effect.

### A2–A5. Other broadcasts

| Action | Result |
|--------|--------|
| `com.heytap.health.sleep.ACTION_SLEEP_STAT_REFRESH` | delivered; code inspection: receiver branch calls `SeedCardSendDataToMetisHelper.sendSleepDataTOSmartBrain` + `SleepDataAdapter.updateSleepSportData` — local/card/assistant logic. No Watch action observed. |
| `com.heytap.health.action_STEP_GOAL` | delivered; no observable action |
| `SLEEP_REMIND_ACTION` | delivered; no observable action |
| `com.heytap.health.PhoneSleepMeasure.notify` | delivered; no observable action |

**Method note**: first attempts were silently swallowed by log noise; reliable capture required `logcat -c` → send → immediate full-buffer dump. Some OHealth log lines are release-gated (e.g. `LogUtils` formatting), so absence of logs alone was never treated as proof — verdicts rely on downstream side effects (provider reads, BT activity).

---

## Experiment B — Health Connect Audit

See `health-connect.md` for the full report. Summary:

1. HC components present; controller alive; providers instantiated.
2. `android.permission.health.*` declared by OHealth / OPPO health service / Operit: **0**.
3. HC client library references in all OHealth dex + healthservice dex: **0**.
4. Live UI: “0 apps have access; no recent data access”.
5. Attempts to launch export/permission UIs from shell: permission UI ✅, export ❌.
6. Logcat watcher (rotating) during experiment window: no HC writes.

**Verdict: no ingestion path. Grade C.**

---

## Experiment C — Watch Communication Observations

| Item | Observation |
|------|-------------|
| Connection state | `OPPO Watch S` bonded; `ACL BR/EDR:Y LE:N` (classic BT), `STATE_CONNECTED` |
| Processes | `com.heytap.health`, `:transport`, `:once` all alive |
| BT address counter sample | tx/rx bytes non-zero and accumulating over time (normal background sync) |
| Correlation with broadcasts | none observed |

---

## Experiment D — Static-vs-Dynamic Cross Checks

- `MessageEvent` construction sites (63 in classes12) all supply `toByteArray()` payloads — no empty/raw-string variants found in the scanned dex set (scan limited to classes2/3/5–7/9/11–14).
- `MsgType.create(sid, cid)` registrations (42 entries) cross-checked against `MessageEvent(sid, cid)` constructor constants — matched where both sides exist (`SleepState`, `DailyActivity`, `PhoneDisplay`, `SpO2`, `Weight`, `WearRecord`, `UserInfo`, `SyncAccountBodyInfo`, `AGPSTransferTask`…).

---

## Reproduction Notes

- Device tooling: Shizuku shell (uid 2000) for `am broadcast` / `dumpsys` / `logcat`; Ubuntu/proot + baksmali for dex analysis.
- Commands are quoted inline above and in `research/timeline.md`.
- All outputs were filtered for the project keywords; raw captures remain private and are not published.