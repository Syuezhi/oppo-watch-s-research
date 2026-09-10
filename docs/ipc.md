# IPC Boundary — What Third-Party Apps Can and Cannot Reach

Status: `CONFIRMED` via manifest parsing + `dumpsys package` on the test device + smali analysis.

---

## 1. The One Exposed Entry Point

| Property | Value |
|----------|-------|
| Service | `com.heytap.health/com.oplus.wearable.linkservice.WearableServer` |
| Filter action | `com.heytap.wearable.linkservice.action.WEARABLE` |
| Required permission | `com.heytap.wearable.linkservice.permission.WEARABLE` |
| Permission protection level | **signature** (`dumpsys package` verified) |
| `onBind` returns | `WearableApiManager.getIWearableServiceBnInterface().asBinder()` |

**Consequence**: only apps signed with the same certificate (OPPO’s own suite) can bind this service. _Interface existence ≠ third-party callability._

## 2. Services That Are NOT Exported

| Service | Where declared | Note |
|---------|----------------|------|
| `com.oplus.wearable.linkservice.transport.connect.ipc.server.IpcBtService` | OHealth manifest | self-closing tag, no intent-filter → not exported |
| `com.heytap.accessory.connectivity.bt.ipc.server.IpcBtService` | OHealth manifest | self-closing tag, no intent-filter → not exported |
| `com.heytap.accessory` (separate package) | `/product/priv-app/AccessoryFramework` v16.35.0 | no public component surface observed in resolver tables |

## 3. Permission Inventory (all OHealth-defined, all `signature`)

```
com.heytap.wearable.linkservice.permission.WEARABLE   signature
com.heytap.health.permission.ACCESS                   signature
com.heytap.health.apiprovider.PERMISSION              signature
com.heytap.health.DEFAULT_PERMISSION                  signature
com.heytap.health.permission.MIPUSH_RECEIVE           signature
com.heytap.health.permission.PROCESS_PUSH_MSG         signature
com.heytap.health.permission.PUSH_PROVIDER            signature
```

## 4. `IWearableService` — AIDL Surface

Methods recovered from `com/oplus/wearable/linkservice/sdk/IWearableService.smali` + `$Stub`/`$Proxy`:

```
addListener(String, IWearableListener)
removeListener(String, IWearableListener)
connect(String, Node, boolean)
disconnect(String, Node)
createBond(String, Node, byte[])
removeBond(String, Node, IRemoveBoundCallback)
getBondNodes(String) / getConnectedNodes(String)
getBondNodesOfWearOS(...) / getConnectedNodesOfWearOS(...)
getWearOSNodeIdByMac(String, String)
sendMessage(String, String, MessageEvent, IWearableCallback): boolean
olinkSendFile(String, FileTransferTask): FileTransferTask
receiveFile(String, int, String, String, String): boolean
rejectFile(String, String) / cancelFile(String, String)
```

**Do not read this as "callable by third-party apps."** It is reachable only via the signature-locked `WearableServer` binder (or in-process).

## 5. Bind Relationships Observed in Code

| Caller | Target | Note |
|--------|--------|------|
| `WearableServer$WearableServerManager.acquireL` | `WearableServer` (self-bind, ComponentName) | keeps its own server alive |
| `WearableListenerService$MyHandler.acquireL` | from `mIntent` (SDK pattern for partner apps) | listener service pattern |
| `com.heytap.accessory.discovery.CentralManager` | action `com.heytap.accessory.ScanService`, pkg `com.heytap.accessory` | accessory framework (priv-app) |

## 6. Other External Surfaces (for completeness)

- **Exported activities** (deeplinks): `.router.RouterActivity`, `.linkage.ui.DeviceDetailsActivity`, `.router` variants, wallet/pay SDK entries, feedback, mini-app entries… — UI entry points only; no protocol invocation.
- **Exported receivers without permission** (e.g. `com.heytap.health.action_data_refresh` → `HealthSeedCardReceiver`): tested; cause local refresh only (see `experiments.md`).
- **Broadcast `opkg` note**: OHealth receivers were reachable with `am broadcast --include-stopped-packages` from shell; no permission denial observed.

## 7. Conclusion

```
Direct IPC from non-OPPO-signed apps:  ❌  (signature permission)
Bridge APK IPC path:                   ❌  (same wall — bridge is also an outside app)
Root injection (out of scope):         ⚠️  possible in principle; not attempted, not recommended
```