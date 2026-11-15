# 构建记录（1.4.0）

## 产物

| 项目 | 值 |
|---|---|
| 文件名 | `AI智能通知助手-v1.4.0.apk`（打包内位于 `apk/` 目录） |
| 大小 | 28.62 MB |
| SHA-256 | `E05CDDAC9C3D702E3524AD0AE823B1B0FFEB8353D9A048A7771DBD8CD5D6FE7D` |
| 包名 | `com.example.ainotification` |
| 版本 | versionCode 5 / versionName 1.4.0 |
| 兼容 | minSdk 29（Android 10+），targetSdk 35 |
| 入口 | `com.example.ainotification.MainActivity` |
| 监听服务 | `com.example.ainotification.notification.NotificationCaptureService` |
| 通知渠道 | `ai_digest`（默认）、`ai_digest_important`（高优先级） |
| 签名 | v1 + v2，debug 证书（`debug.keystore`，口令 `android`，alias `androiddebugkey`） |
| 备份 | `allowBackup=false`；`enableOnBackInvokedCallback=true`（预测式返回） |

校验命令（均已通过）：

```bash
apksigner verify AI智能通知助手-v1.4.0.apk
aapt2 dump badging AI智能通知助手-v1.4.0.apk
```

## 构建环境

- OpenJDK 17.0.2 / Gradle 8.11.1
- AGP 8.7.3 / Kotlin 2.0.21 / KSP 2.0.21-1.0.28 / Compose BOM 2024.12.01
- Android SDK：platform 35、build-tools 34.0.0

## 构建命令

```bash
export JAVA_HOME=/path/to/jdk17
./gradlew :app:assembleDebug
# 产物：app/build/outputs/apk/debug/app-debug.apk
```

单元测试：`./gradlew :app:testDebugUnitTest`

本机依赖走阿里云 / 腾讯云 / 华为云 Maven 镜像（`--init-script` 注入），工程内仓库声明仍是标准的
`google()/mavenCentral()`；`local.properties` 打包时已排除。

## 1.4.0 相对 1.3.0 的变更

### 引导不再「假装完成」

- 完成页根据真实状态显示：权限未开或 Key 未配置时标题为「还没配置完成」，并列出缺失项，
  不再显示「准备就绪」。
- 完成页提供「去补全设置」，直接跳回权限页或 AI 页。
- 底部主按钮在未完成时显示「稍后再设置」，仍可结束引导。

### 设置里补完配置

- 设置页顶部新增「配置状态」卡片：显示还剩几项未完成，并可直接
  - 开关通知使用权 / 请求通知权限；
  - 跳转 AI 设置配置 Key；
  - 跳转通知设置调整摘要通知。
- 状态在返回设置页时自动刷新（`LifecycleEventSource.ON_RESUME` + `SetupStatus`）。

### 液态玻璃质感

- 新增 `ui/components/LiquidGlass.kt`：半透明底 + 顶部高光 + 渐变高光描边 + 柔和投影。
- 全局卡片（`GlassCard`）改为玻璃质感，首页摘要主卡叠加缓慢扫过的反光（5.2s 循环）。
- 底部导航栏使用玻璃背景与圆角顶部，内容在其下透出。
- 说明：Compose 无法真正模糊背后系统内容，这里用多层半透明与高光模拟折射。

### 统计页新增两张卡

- 「消息类别」：按 androidCategory 归并成中文类别，环形占比图 + 图例（带生长动画）。
- 「重要消息」：最近摘要里的 high 项，展示概括标题 + 具体内容，点击进入摘要详情。

### 通知内容按统计模式呈现

- 主摘要通知与重要提醒通知统一改为：
  第一行「类型：聊天 5 · 邮件 2 · 其他 3」，其后是重要消息的「标题：内容」，超出显示「另有 N 项」。
- 为此 `DigestInput` 增加 `androidCategory`，由 `NotificationCategories` 统一归一与计数。

## 验证记录

| 步骤 | 结果 |
|---|---|
| `:app:compileDebugKotlin` | 成功 |
| `:app:assembleDebug` | 成功 |
| 单元测试（JUnit4，4 个测试类） | 17/17 通过 |
| `apksigner verify` | 通过 |
| `aapt2 dump badging` | 包名/版本 1.4.0/SDK/权限 均符合预期 |

未做：真机安装、通知监听实测、玻璃质感与扫光观感、引导状态流转（本机无设备/模拟器）。
