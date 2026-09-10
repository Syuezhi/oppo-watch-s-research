# Health Connect 调查

**简体中文 | [English](health-connect.md)**

目标：验证测试机上是否存在 **Watch S → OHealth → Health Connect → 第三方 App** 这条链路。

**结果：不存在。评级 C。**

---

## 1. 存在的组件

| 组件 | 路径 / 版本 | 备注 |
|-----------|----------------|------|
| `com.android.healthconnect.controller` | 系统模块（HealthFitness APEX 家族） | 运行中（观察到 pid）；providers：`HealthConnectSearchIndexablesProvider`、`InitializationProvider` |
| `com.android.health.connect.backuprestore` | `/apex/com.android.healthfitness/app/...` | 备份/恢复模块 |
| HC Activity | `TrampolineActivity`、`PermissionsActivity`、`ExportSetupActivity`、`ImportFlowActivity`、`MigrationActivity`、`RouteRequestActivity` | 可解析；`PermissionsActivity` 与 `TrampolineActivity` 已成功从 shell 拉起 |

## 2. 权限审计（决定性部分）

```
com.heytap.health           android.permission.health.*  →  0 条
com.oplus.healthservice     android.permission.health.*  →  0 条
com.ai.assistance.operit    android.permission.health.*  →  0 条
```

- 对三个包执行 `dumpsys package`（精确 `android\.permission\.health\.` 过滤）：**全部为空**。
- OHealth 完全没有声明任何 Health Connect 读写权限。

## 3. 代码审计

- 全部 17 个 OHealth dex 文件（`classes.dex` … `classes17.dex`）：`HealthConnect` 字符串计数 = **0**。
- `com.oplus.healthservice`（v16.2.10，`/system_ext/app/HealthService/HealthService.apk`）的 classes.dex：`health.connect` = **0**，`HealthConnect` = **0**。
- 两侧都不含任何 Health Connect 客户端库引用。

## 4. 动态证据

- 现场 HC 界面快照（设备截取）：标题"健康数据共享"。
  - **"0 个应用拥有访问权限（共 6 个应用）"**
  - **"最近没有任何应用访问'健康数据共享'"**
- 实验窗口内的 logcat 监控：OHealth 无 HC 写入活动；无可归因于 OHealth 的 `insertRecords`/HC API 痕迹。
- HC 进程存活但状态为空；未挂载任何数据源。

## 5. 尝试过的捷径（供复现）

| 尝试 | 结果 |
|---------|--------|
| `am start -a android.health.connect.action.START_EXPORT_SETUP` | 无法解析（shell） |
| `am start -n .../.exportimport.ExportSetupActivity` | 启动异常 |
| `am start -n .../.permissions.request.PermissionsActivity` | ✅ 已拉起（界面可见） |
| `am start -n .../.navigation.TrampolineActivity` | ✅ 已拉起（界面可见） |
| `dumpsys healthconnect` | 无 dump 实现 |
| 监视器：测试期间过滤 HealthConnect 的 `logcat` | 没有可捕获的内容 |

## 6. 正确的表述

- 这**不是**在说第三方 App 无法使用 Health Connect。HC 是标准平台 API：任何声明了相应 `android.permission.health.*` 权限的 App 都可以经用户授权获得访问。
- 在这台设备上，**Health Connect 里没有任何 Watch S 数据可读** —— OHealth（以及 OPPO 健康服务）从未向 HC 写入数据。

## 7. 结论

```
Watch S → OHealth → Health Connect   ❌  （所测配置上无写入路径）
Watch S → OHealth → OPPO/欢太 云     ✅  （观察到：OHealth 以账号/云为中心）
```

评级：**C** —— 在此设备上 Health Connect 不是 Watch S 的可行数据路线。