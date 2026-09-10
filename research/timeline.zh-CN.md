# 研究时间线（第 1–10 阶段）

**简体中文 | [English](timeline.md)**

产出本仓库的调查过程浓缩日志。日期相对研究窗口（2026 年 9 月）。

---

## 阶段 1–2 — 定向与试错
- 初始目标：在 OHealth 中寻找与"手表"相关的数据路径。
- 早期线索 `GM_CHARACTERISTIC` → **修正**：属于 `com.omron.ar`（血糖仪域）。教训：不要相信命名。
- 工具链建立：APK 拉取 → `unzip classes*.dex` → **baksmali**（Ubuntu/proot；依赖：dexlib2/util/guava/jcommander）。jadx 因设备内存不足放弃。

## 阶段 3–4 — 真正的 BLE 栈 vs 手表栈
- 定位通用 BLE 框架：`com.heytap.health.ble.{GattChannel, BleClient, DispatchCenter, ValueChangedCallback}` —— 完整 CCCD/notify/write 链还原。
- 确认其业务消费方是**第三方健身设备**（跑步机等），*不是*手表。
- 手表路径分流至 `com.oplus.wearable.linkservice` + `com.heytap.health.devicemanager`。

## 阶段 5 — SDK/传输层定位
- 在 OHealth manifest 中发现声明的 `WearableServer` + `IpcBtService`。
- 枚举 `IWearableService`（AIDL）接口面。
- `ConnectManager → linkservice.sdk.Node` = OHealth 面向手表的那条边。

## 阶段 6 — 消息协议还原
- `MessageEvent(II[B)` 结构 + 常量（`ENCRYPT_*`、`TRANSPORT_BT`…）。
- `BTClient.sendMsg` / `MessageTransferManager.sendMessage` / 经 `MsgProcessor` 的接收链。
- 跨 dex 扫描：55 处 `sendMsg` 调用点，`sporthealth/receive` 下 191 个接收侧文件。

## 阶段 7 — 可调用级命令
- 六条命令完成逆向并附 protobuf 字段映射（心率统计、运动统计、运动列表、运动记录、健身、日常活动）。
- `FitnessProto`/`FileProto` 定位在 `classes14.dex`；`*_FIELD_NUMBER` 常量提取完成。

## 阶段 8 — IPC 边界
- `WearableServer`：filter `…action.WEARABLE`，权限 `…permission.WEARABLE` → **signature**（dumpsys）。
- `IpcBtService` ×2：未导出。`com.heytap.accessory`：`/product/priv-app/AccessoryFramework`。
- 直连 IPC / 桥接 APK IPC：**排除**。

## 阶段 9 — 动态触发搜索（广播）
- 在干净 logcat 协议下测试五个 OHealth 广播。
- `action_data_refresh`：已投递 → **仅本地刷新**（勋章检查 + provider 读取）。其余：无可观测动作。
- 判定：不存在能触发手表同步的导出广播。

## 阶段 10 — Health Connect 审计
- HC 组件存在；**零**条 HC 权限被任何包声明；OHealth 与 healthservice dex 中 **零**处 HC 引用。
- 现场 HC UI：0 个应用拥有访问权限；无最近访问。
- HC 路线 **评级 C**。

---

## 最终状态

| 层 | 结果 |
|-------|--------|
| 协议 | ⭐ 已还原（记录于此） |
| 通道 | 🔒 signature 锁死 |
| 数据出口（HC） | 🚫 不存在 |
| 剩余活路 | UI 自动化 / 用户手动导出 / 云端（未研究） |