# Health Connect Investigation

Goal: verify whether the chain **Watch S → OHealth → Health Connect → third-party app** exists on the test device.

**Result: it does not. Grade C.**

---

## 1. Components Present

| Component | Path / Version | Note |
|-----------|----------------|------|
| `com.android.healthconnect.controller` | system module (HealthFitness APEX family) | running (pid observed); providers: `HealthConnectSearchIndexablesProvider`, `InitializationProvider` |
| `com.android.health.connect.backuprestore` | `/apex/com.android.healthfitness/app/...` | backup/restore module |
| HC activities | `TrampolineActivity`, `PermissionsActivity`, `ExportSetupActivity`, `ImportFlowActivity`, `MigrationActivity`, `RouteRequestActivity` | resolvable; `PermissionsActivity` and `TrampolineActivity` were launched successfully from shell |

## 2. Permission Audit (the decisive part)

```
com.heytap.health           android.permission.health.*  →  0 entries
com.oplus.healthservice     android.permission.health.*  →  0 entries
com.ai.assistance.operit    android.permission.health.*  →  0 entries
```

- `dumpsys package` (precise `android\.permission\.health\.` filter) on all three packages: **empty**.
- OHealth declares **no** Health Connect read/write permissions at all.

## 3. Code Audit

- All 17 OHealth dex files (`classes.dex` … `classes17.dex`): `HealthConnect` string count = **0**.
- `com.oplus.healthservice` (v16.2.10, `/system_ext/app/HealthService/HealthService.apk`), classes.dex: `health.connect` = **0**, `HealthConnect` = **0**.
- Neither side contains any Health Connect client library reference.

## 4. Dynamic Evidence

- Live HC UI snapshot (captured from device): title “健康数据共享”.
  - **“0 个应用拥有访问权限（共 6 个应用）”**
  - **“最近没有任何应用访问‘健康数据共享’”**
- Logcat monitoring during the experiment window: no HC write activity from OHealth; no `insertRecords`/HC API traces attributable to OHealth.
- HC process is alive with empty state; no data sources attached.

## 5. Attempted Shortcuts (for reproducibility)

| Attempt | Result |
|---------|--------|
| `am start -a android.health.connect.action.START_EXPORT_SETUP` | not resolvable (shell) |
| `am start -n .../.exportimport.ExportSetupActivity` | exception on launch |
| `am start -n .../.permissions.request.PermissionsActivity` | ✅ launched (UI-visible) |
| `am start -n .../.navigation.TrampolineActivity` | ✅ launched (UI-visible) |
| `dumpsys healthconnect` | no dump implementation |
| Watcher: `logcat` filtered for HealthConnect during tests | nothing to catch |

## 6. Correct Framing

- This is **not** a statement that Health Connect cannot be used by third-party apps. HC is a standard platform API: any app that declares the relevant `android.permission.health.*` permissions can be granted access by the user.
- On this device, **there is no Watch S data in Health Connect to read** — OHealth (and the OPPO health service) never ingest data into HC.

## 7. Conclusion

```
Watch S → OHealth → Health Connect   ❌  (no ingestion path on tested configuration)
Watch S → OHealth → OPPO/欢太 cloud  ✅  (observed: OHealth is account/cloud-centric)
```

Grade: **C** — Health Connect is not a viable data route for Watch S on this device.