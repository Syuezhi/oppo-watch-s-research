# 动态实验日志

**简体中文 | [English](experiments.md)**

所有动态测试均使用**仅观察**工具（自有设备上经 Shizuku 的 shell/adb 等效环境）。不改系统、不注入、不破坏数据。

---

## 实验 A — 广播触发测试

**问题**：第三方 App 能否通过 OHealth 的导出广播触发手表同步？

### A1. `com.heytap.health.action_data_refresh`（接收者：`HealthSeedCardReceiver`）

| 步骤 | 观察 |
|------|-------------|
| 发送 | `am broadcast --include-stopped-packages -a com.heytap.health.action_data_refresh` → `Broadcast completed: result=0`（无权限拒绝） |
| +1s | `ActivityManager: Broadcasting: Intent {…}` —— 已投递 |
| +1s | OHealth 主进程：`DFJ.MedalSecret … medal_check_DailyStepMedal`（步数勋章检查） |
| +1s | `SportHealthDataAPI: readSportHealthData table:1002`（startTime=endTime=now；读 1 行；errorCode:0）—— **本地 provider 读取** |
| +0–60s | **无** BTClient / sendMsg / 数据同步 / 传输层活动。手表蓝牙计数特征不变。 |

**判定：仅本地刷新。** 接收者可达且功能正常；无手表侧效应。

### A2–A5. 其他广播

| Action | 结果 |
|--------|--------|
| `com.heytap.health.sleep.ACTION_SLEEP_STAT_REFRESH` | 已投递；代码检查：接收者分支调用 `SeedCardSendDataToMetisHelper.sendSleepDataTOSmartBrain` + `SleepDataAdapter.updateSleepSportData` —— 本地/卡片/助手逻辑。无可观测手表动作。 |
| `com.heytap.health.action_STEP_GOAL` | 已投递；无可观测动作 |
| `SLEEP_REMIND_ACTION` | 已投递；无可观测动作 |
| `com.heytap.health.PhoneSleepMeasure.notify` | 已投递；无可观测动作 |

**方法备注**：最初的尝试被日志噪声吞没；可靠捕获需要 `logcat -c` → 发送 → 立刻全缓冲 dump。部分 OHealth 日志行受 release 开关控制（如 `LogUtils` 格式化），因此"日志为空"从不单独作为证据——判定依赖下游副作用（provider 读取、BT 活动）。

---

## 实验 B — Health Connect 审计

完整报告见 `health-connect.md`。摘要：

1. HC 组件存在；controller 存活；providers 已实例化。
2. OHealth / OPPO 健康服务 / Operit 声明的 `android.permission.health.*`：**0**。
3. 全部 OHealth dex + healthservice dex 中的 HC 客户端库引用：**0**。
4. 现场 UI："0 个应用拥有访问权限；最近没有数据访问"。
5. 从 shell 尝试拉起导出/权限界面：权限界面 ✅，导出 ❌。
6. 实验窗口内的 logcat 监视器（轮转）：无 HC 写入。

**判定：无写入路径。评级 C。**

---

## 实验 C — 手表通信观察

| 项目 | 观察 |
|------|-------------|
| 连接状态 | `OPPO Watch S` 已配对；`ACL BR/EDR:Y LE:N`（经典蓝牙），`STATE_CONNECTED` |
| 进程 | `com.heytap.health`、`:transport`、`:once` 均存活 |
| 蓝牙地址计数样本 | tx/rx 字节非零且随时间累积（正常后台同步） |
| 与广播的相关性 | 未观察到 |

---

## 实验 D — 静态 vs 动态交叉核对

- `MessageEvent` 构造点（classes12 中 63 处）全部提供 `toByteArray()` 负载——扫描的 dex 集（classes2/3/5–7/9/11–14）中未发现空负载或原始字符串变体。
- `MsgType.create(sid, cid)` 注册（42 条）与 `MessageEvent(sid, cid)` 构造函数常量交叉核对——两侧同时存在处全部吻合（`SleepState`、`DailyActivity`、`PhoneDisplay`、`SpO2`、`Weight`、`WearRecord`、`UserInfo`、`SyncAccountBodyInfo`、`AGPSTransferTask`…）。

---

## 复现备注

- 设备工具：Shizuku shell（uid 2000）用于 `am broadcast` / `dumpsys` / `logcat`；Ubuntu/proot + baksmali 用于 dex 分析。
- 命令已在上文与 `research/timeline.md` 中内联引用。
- 所有输出均以项目关键词过滤；原始捕获保持私有，不对外发布。