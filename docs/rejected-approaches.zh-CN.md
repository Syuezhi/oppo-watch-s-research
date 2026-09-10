# 已排除方案 — 附证据

**简体中文 | [English](rejected-approaches.md)**

以下各项均已尝试或评估后**排除**。列出它们，是为了不再有人重走同样的死路。

---

## 1. Health Connect 路线

**结果：❌ 在所测配置上不可行。**

证据：
- OHealth 声明 **0 条** `android.permission.health.*` 权限（dumpsys）。
- `com.oplus.healthservice`（v16.2.10）含 **0 处** Health Connect 引用（dex 扫描）。
- 全部 17 个 OHealth dex 文件含 **0 处** `HealthConnect` 引用。
- 现场 HC UI：0 个应用拥有访问权限，无最近访问。
- 监控期间未观察到写入。

→ `docs/health-connect.md`

---

## 2. OHealth 广播 `com.heytap.health.action_data_refresh`

**结果：❌ 仅本地刷新。**

证据：接收者已投递；下游 = 步数勋章检查 + `SportHealthDataAPI.readSportHealthData`（本地数据库）；观察窗口内无 BT/传输层/sendMsg 活动。

---

## 3. 广播 `com.heytap.health.sleep.ACTION_SLEEP_STAT_REFRESH`

**结果：❌ 本地 / 智能大脑逻辑。**

证据：接收者分支（`HealthSeedCardReceiver`）→ `sendSleepDataTOSmartBrain` + `updateSleepSportData`。需要 `hasCurDaySleepData` extra 才会做任何事；与手表无关。

---

## 4. 广播 `com.heytap.health.action_STEP_GOAL`

**结果：❌ 未观察到手表同步。**

---

## 5. 广播 `SLEEP_REMIND_ACTION`

**结果：❌ 未观察到手表同步。**

---

## 6. 广播 `com.heytap.health.PhoneSleepMeasure.notify`

**结果：❌ 未观察到手表同步。**

---

## 7. 第三方 App 直接绑定 `WearableServer`

**结果：❌ signature 权限。**

证据：`com.heytap.wearable.linkservice.permission.WEARABLE` → `prot=signature`（dumpsys）。非 OPPO 签名的 App 无法绑定。

---

## 8. 直接调用 linkservice / AIDL

**结果：❌ 无可用签名。**

`IWearableService` 只能通过签名锁定的服务端（或进程内）到达；桥接 APK 撞同一堵墙。

---

## 9. 基于 UUID 猜测手表 BLE 协议

**结果：❌ 不可靠；不是 Watch S 协议。**

已明确标注为**不构成** Watch S 证据：
- `FTMS`（`00001826` …）—— 代表 Fitness Machine Service（跑步机语境），不是手表。
- 跑步机相关 UUID（如 `b1e73412-…`、`8a998074-…` —— FTMS 扩展 chars、跑步机域）。
- NUS-like 组 `6E4000xx` —— 本设备上无 GATT 调用路径证据。
- `A5xx` / `A6xx` / `A7xx` 集群 —— 仅字符串存在；无调用链。
- `AA15` / `BB15` —— 配件框架常量（`com.heytap.accessory.connectivity.params`），未证明承载 Watch 消息。
- `a49eaa15-…` / `a49ebb15-…` —— 配件配对常量；未被证明是 Watch S 业务通道。

采用的规则：**字符串池里的 UUID ≠ 协议。只有实际的 GATT/Binder 调用链才算数，且只有动态验证才能确认。** 上述各项对 Watch S 都过不了这条线。

---

## 横切面：按设计未尝试的事

- 不破解加密、不提取密钥。
- 不绕过签名、不做 root 注入、不修改 APK。
- 不对手表做 BLE 层攻击。
- 不滥用账号/云凭证。

这些无论难度如何都在范围之外。