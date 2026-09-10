# IPC 边界 — 第三方 App 能够到与不能够到的地方

**简体中文 | [English](ipc.md)**

状态：经 manifest 解析 + 测试机 `dumpsys package` + smali 分析确认为 `CONFIRMED`。

---

## 1. 唯一的对外入口

| 属性 | 值 |
|----------|-------|
| 服务 | `com.heytap.health/com.oplus.wearable.linkservice.WearableServer` |
| Filter action | `com.heytap.wearable.linkservice.action.WEARABLE` |
| 所需权限 | `com.heytap.wearable.linkservice.permission.WEARABLE` |
| 权限保护级别 | **signature**（`dumpsys package` 已验证） |
| `onBind` 返回 | `WearableApiManager.getIWearableServiceBnInterface().asBinder()` |

**后果**：只有使用同一证书签名的 App（OPPO 自家套件）才能绑定此服务。_接口存在 ≠ 第三方可调。_

## 2. 未导出的服务

| 服务 | 声明于 | 备注 |
|---------|----------------|------|
| `com.oplus.wearable.linkservice.transport.connect.ipc.server.IpcBtService` | OHealth manifest | 自闭合标签，无 intent-filter → 未导出 |
| `com.heytap.accessory.connectivity.bt.ipc.server.IpcBtService` | OHealth manifest | 自闭合标签，无 intent-filter → 未导出 |
| `com.heytap.accessory`（独立包） | `/product/priv-app/AccessoryFramework` v16.35.0 | 解析表中未观察到公开组件面 |

## 3. 权限清单（全部为 OHealth 自定义，全部 `signature`）

```
com.heytap.wearable.linkservice.permission.WEARABLE   signature
com.heytap.health.permission.ACCESS                   signature
com.heytap.health.apiprovider.PERMISSION              signature
com.heytap.health.DEFAULT_PERMISSION                  signature
com.heytap.health.permission.MIPUSH_RECEIVE           signature
com.heytap.health.permission.PROCESS_PUSH_MSG         signature
com.heytap.health.permission.PUSH_PROVIDER            signature
```

## 4. `IWearableService` — AIDL 接口面

从 `com/oplus/wearable/linkservice/sdk/IWearableService.smali` + `$Stub`/`$Proxy` 还原的方法：

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

**不要把这份清单读成"第三方 App 可调用"。** 它只能通过签名锁定的 `WearableServer` binder（或进程内）到达。

## 5. 代码中观察到的绑定关系

| 调用方 | 目标 | 备注 |
|--------|--------|------|
| `WearableServer$WearableServerManager.acquireL` | `WearableServer`（自绑定，ComponentName） | 保持自身服务存活 |
| `WearableListenerService$MyHandler.acquireL` | 来自 `mIntent`（面向合作 App 的 SDK 模式） | 监听服务模式 |
| `com.heytap.accessory.discovery.CentralManager` | action `com.heytap.accessory.ScanService`，包 `com.heytap.accessory` | 配件框架（priv-app） |

## 6. 其他对外面（完整性起见）

- **导出的 Activity**（deeplink）：`.router.RouterActivity`、`.linkage.ui.DeviceDetailsActivity`、`.router` 变体、钱包/支付 SDK 入口、反馈、小程序入口…… —— 仅为 UI 入口；不承载协议调用。
- **无权限导出的 receiver**（例如 `com.heytap.health.action_data_refresh` → `HealthSeedCardReceiver`）：已实测；只引发本地刷新（见 `experiments.md`）。
- **广播 `opkg` 备注**：OHealth 的 receiver 可从 shell 用 `am broadcast --include-stopped-packages` 到达；未观察到权限拒绝。

## 7. 结论

```
非 OPPO 签名 App 直连 IPC：  ❌  （signature 权限）
Bridge APK 走 IPC 的路径：   ❌  （同一堵墙——桥接器同样是外部 App）
Root 注入（超范围）：        ⚠️  原理上可能；未尝试，不推荐
```