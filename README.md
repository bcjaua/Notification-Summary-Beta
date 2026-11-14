# Notification-Summary-Beta
通知摘要项目的测试仓库
# AI 智能通知助手（AI Notification Assistant）

一个面向 Android 的本地通知整理工具。应用通过 Android 通知使用权读取设备通知，将通知保存在本地，并提供基于重要度的 AI 摘要能力。项目依据《AI 智能通知助手：技术实现方案》搭建。

> **当前版本说明：** 本仓库是 MVP 开发工程，不代表 PDF 中列出的所有功能都已完成，也不是已经完成真机验收的正式发布版。请先构建并在自己的设备上测试，再用于日常通知管理。

## 目录

- [项目目标](#项目目标)
- [核心功能与当前状态](#核心功能与当前状态)
- [技术栈](#技术栈)
- [项目结构](#项目结构)
- [运行环境](#运行环境)
- [构建与运行](#构建与运行)
- [首次使用](#首次使用)
- [配置真实 AI 服务](#配置真实-ai-服务)
- [通知处理与隐私](#通知处理与隐私)
- [数据与安全说明](#数据与安全说明)
- [已知限制与待完成事项](#已知限制与待完成事项)
- [GitHub Actions 构建 APK](#github-actions-构建-apk)
- [问题反馈与开发建议](#问题反馈与开发建议)

## 项目目标

MVP 的核心使用流程：

1. 用户授予通知使用权。
2. 应用监听新通知，解析并保存到本地数据库。
3. 用户点击“立即生成摘要”。
4. 摘要模块将符合条件的通知交给 Fake AI 或用户配置的 OpenAI 兼容服务。
5. 应用解析 AI 返回结果，校验来源编号并按重要度排序。
6. 应用发布一条独立的 AI 摘要通知；原始通知不会被取消、替换或隐藏。

AI 摘要的目标分档为 `high`、`medium`、`low`。同一事件可以合并多条通知，并通过来源关联保留回看原始通知的基础数据结构。

## 核心功能与当前状态

| 模块 | 当前工程状态 |
|---|---|
| Android 原生单模块工程 | 已建立 |
| Jetpack Compose / Material 3 界面 | 已建立基础页面 |
| 通知监听服务 | 已有基础实现 |
| 通知正文解析与内容哈希去重 | 已有基础实现 |
| 常驻通知、组汇总通知过滤 | 已实现基础过滤 |
| Room 数据模型 | 已建立通知、摘要、摘要项、来源关联、应用过滤等数据模型 |
| Fake AI 摘要 | 已实现基础链路 |
| OpenAI 兼容 Chat Completions | 已有基础请求实现，需配置并实测 |
| 摘要 JSON 解析、来源编号校验与排序 | 已有基础实现 |
| 固定 ID 摘要通知 | 已实现基础发布逻辑 |
| 通知历史页面 | 已有基础页面 |
| API Key 加密存储 | 已有 Android Keystore 相关实现 |
| 每日清理任务 | 已有基础 Worker |
| DataStore 配置持久化 | 尚未完整接入 |
| 摘要详情及逐项原始通知追溯 UI | 尚未完成 |
| 应用 ALLOW / BLOCK / NO_AI 管理界面 | 尚未完成 |
| 通知累计 30 条自动触发 | 尚未完成 |
| 每日 AI 调用次数上限 | 尚未完成 |
| 网络失败重试及完整本地简版兜底 | 尚未完成 |
| 完整诊断导出、隐私管理页面 | 尚未完成 |
| 真机兼容性及 PDF 全部验收项 | 尚未完成 |

具体以仓库中的 [`PROJECT_STATUS.md`](PROJECT_STATUS.md) 为准。

## 技术栈

- **语言：** Kotlin
- **界面：** Jetpack Compose、Material 3
- **本地数据：** Room
- **偏好设置：** DataStore（依赖已配置，持久化尚未完整接入）
- **网络：** OkHttp
- **JSON：** kotlinx.serialization
- **异步：** Kotlin Coroutines
- **后台任务：** WorkManager
- **密钥保护：** Android Keystore 相关加密存储
- **构建：** Gradle、Android Gradle Plugin、KSP

## 项目结构

```text
ai-notification-assistant/
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── res/
│       └── java/com/example/ainotification/
│           ├── AssistantApplication.kt
│           ├── MainActivity.kt
│           ├── ai/
│           │   ├── AIRepository.kt
│           │   └── DigestPublisher.kt
│           ├── data/
│           │   ├── Models.kt
│           │   └── SecureKeyStore.kt
│           ├── notification/
│           │   └── NotificationCaptureService.kt
│           ├── ui/
│           └── worker/
│               └── Workers.kt
├── .github/workflows/android.yml
├── build.gradle.kts
├── gradle.properties
├── settings.gradle.kts
├── PROJECT_STATUS.md
└── README.md
```

## 运行环境

- Android Studio（建议使用当前稳定版）。
- JDK 17。
- Android SDK：项目配置 `compileSdk = 35`、`targetSdk = 35`。
- 最低系统版本：Android 10（API 29）。
- 首次构建需要网络下载 Gradle 插件和依赖。

## 构建与运行

### 使用 Android Studio

1. 解压项目 ZIP。
2. 在 Android Studio 中选择 **Open**，打开 `ai-notification-assistant` 项目根目录。
3. 等待 Gradle Sync 完成；如果提示安装 Android SDK 35，请按提示安装。
4. 连接 Android 设备并启用开发者选项与 USB 调试，或启动模拟器。
5. 选择 `app` 运行配置并运行。

也可以在项目根目录执行：

```bash
./gradlew :app:assembleDebug
```

Windows 命令行使用：

```powershell
.\gradlew.bat :app:assembleDebug
```

成功后，Debug APK 通常位于：

```text
app/build/outputs/apk/debug/app-debug.apk
```

**注意：** 当前交付包未包含 Gradle Wrapper 可执行脚本及 Wrapper JAR。如果项目目录中没有 `gradlew` / `gradlew.bat`，请先在 Android Studio 中使用其 Gradle 集成构建，或补齐 Gradle Wrapper 后再执行上述命令。构建是否成功需以实际 Android Studio / CI 结果为准。

## 首次使用

1. 安装并打开应用。
2. 在系统设置中进入“通知使用权”或“通知访问权限”，允许本应用读取通知。
3. Android 13（API 33）及以上系统还需要授予通知发送权限，否则应用无法发布摘要通知。
4. 返回应用，确认通知监听授权状态。
5. 先使用 Fake AI 流程验证基本链路。
6. 收到若干普通通知后，点击“立即生成摘要”。
7. 检查应用是否发布 AI 摘要通知，并确认原始通知仍保留在系统通知栏中。

不同 Android 厂商对后台运行的管理不同。若监听服务容易断开，请检查系统的电池优化、自启动和后台运行限制设置。

## 配置真实 AI 服务

当前工程包含 OpenAI 兼容 Chat Completions 的基础请求实现。使用真实服务前，请确认服务商提供兼容接口，并准备：

- **Base URL：** 服务商 API 根地址。代码会在该地址基础上请求 `/chat/completions`；请按当前实现和服务商要求填写，避免重复拼接路径。
- **Model：** 服务商支持的模型名称。
- **API Key：** 对应服务商的密钥。
- **网络：** 设备能够访问该 API 服务。

使用前请检查应用中的 Fake AI 开关及配置项。真实 API 的请求会将符合筛选条件的通知内容发送到你配置的第三方服务。不要将 API Key 写入源码、提交到 Git 仓库或发送给他人。

> 当前版本的配置持久化、连接测试、错误处理和重试能力尚未达到完整方案要求。请勿将当前实现视为对任意 OpenAI 兼容服务商均已验证。

## 通知处理与隐私

通知可能包含私人聊天、验证码、金融信息或其他敏感内容。当前工程包含基础的本地过滤和脱敏逻辑，设计目标包括：

- 过滤本应用自身通知，避免摘要通知再次进入采集。
- 过滤常驻通知和群组汇总通知。
- 对部分短信、银行、支付、密码管理类应用及验证码特征进行默认跳过处理。
- 对发送给 AI 的内容尝试替换手机号、银行卡号和邮箱等信息。
- 不在日志中主动输出通知正文或 API Key。

这些规则属于基础实现，不能保证识别所有敏感应用或所有敏感信息。用户应在真实 AI 模式开启前检查过滤逻辑，并避免将不希望离开设备的通知发送给第三方服务。

## 数据与安全说明

- 通知和摘要数据设计为保存在应用私有存储中。
- API Key 使用 Android Keystore 相关加密机制保护。
- 加密存储不等于绝对安全；设备被攻破、调试环境不安全或系统存在漏洞时，仍可能发生数据泄露。
- 卸载应用或清除应用数据可能导致本地通知历史和配置丢失。
- 请勿把真实通知内容、API Key 或个人数据加入公开 issue、日志或代码仓库。

## 已知限制与待完成事项

在将项目用于长期日常使用或发布前，建议至少完成：

1. 完整接入 DataStore，持久化 Base URL、Model、超时、输出长度、Temperature 和相关开关。
2. 完成摘要详情页，支持摘要项到原始通知的逐条来源追溯。
3. 完成应用过滤管理，支持 `ALLOW`、`BLOCK`、`NO_AI`。
4. 实现通知累计达到阈值后的自动摘要任务。
5. 实现每日调用上限和调用用量记录。
6. 补齐网络错误分类、重试、失败状态及本地简版摘要兜底。
7. 完成首次启用 AI 的第三方数据发送告知与用户确认。
8. 完成通知清理、摘要清理、清除全部数据等隐私管理操作。
9. 在 Android 10 至当前目标版本及不同厂商设备上测试通知监听、权限撤销、后台回收和重启恢复。
10. 按 PDF 第 13–14 页的验收步骤逐项测试并记录结果。

## 问题反馈与开发建议

提交问题时请提供：

- Android 版本及设备型号（无需提供个人设备标识）。
- 复现步骤。
- 预期行为与实际行为。
- 不包含通知正文、API Key 或其他私人内容的错误日志。

建议按“通知采集 → Fake AI 摘要 → 摘要通知与来源追溯 → 真实 AI → 隐私与稳定性”的顺序开发和验收，先确保本地链路可靠，再扩展功能。
