# AI 通知助手 · 第一阶段 MVP

目标：在自己的手机上，稳定抓到真实通知并存进本地数据库。本阶段**不联网、不碰 AI**。

- 包名 `com.notificationassistant.ai`，minSdk 29，Kotlin + Jetpack Compose + Room，单模块，Gradle Kotlin DSL + Version Catalog
- 依赖注入用手写的 `AppContainer`（没有引入 Hilt，少一个注解处理器，手机上构建更快）

## 在 Code On The Go 里构建

1. 把整个文件夹解压到手机，用 Code On The Go 打开。
2. 执行 `assembleDebug`，APK 在 `app/build/outputs/apk/debug/`。**记下首次构建和增量构建的耗时。**
3. 如果提示缺少 `android-34` 平台：在 Code On The Go 里安装 SDK 34；或把 `app/build.gradle.kts` 里的 `compileSdk`/`targetSdk` 改成你已有的版本（依赖是按 compileSdk 34 选的，改成更高没问题，改低会报错）。
4. 如果构建时 Gradle 内存不足，调大 `gradle.properties` 里的 `-Xmx`。

## 首次使用

1. 打开应用，首页点“通知使用权 → 去开启”，在系统页里打开本应用开关。
2. **开关是灰色点不动**（Android 13+ 侧载 APK 常见）：首页点“应用信息” → 右上角 ⋮ → “允许受限制的设置”，再回去开。
3. Android 13+ 在首页点“发送通知权限 → 授权”。
4. 首页“监听服务”显示“运行中”即成功。
5. 设置页里把本应用的电池优化设为“不受限制”，减少被系统杀掉的概率；国产系统还需在自家的“自启动 / 后台运行”里放行。

## 验收清单

**第 1 批**：APK 能装、能启动、四个底部页面能切换。

**第 2 批**：让微信、短信、邮件各来一条真实通知 → 时间线里出现；同一条通知被刷新、或关掉再重新开启通知使用权 → 不会出现重复。

**第 3 批**：
- 连续用一天，对比时间线条数和你实际收到的通知。
- 重启手机后不打开应用，等收到一条通知，看是否入库（验证系统自动恢复）。
- 首页“内容不完整的应用”列表里记录了哪些应用抓不到内容。

## 设计说明

- **过滤**：排除自身包名；忽略常驻通知（`isOngoing` / 前台服务）和群组汇总通知；按内容哈希去重。
- **去重窗口**：同一 `contentHash`（SHA-256 of 包名+标题+正文）在 10 分钟内只存一条。系统在服务重新连接时会把通知栏里还在的通知再投递一遍，它们的发布时间不变，会被挡住；隔很久再出现的相同内容（如“好的”）仍正常入库。要改成永久去重，把 `NotificationRepository.DEDUP_WINDOW_MS` 改成 `Long.MAX_VALUE`。
- **contentAvailability**：`FULL` / `TITLE_ONLY` / `TEXT_ONLY` / `EMPTY`，抓不到完整内容的通知照样入库并标记。目前无法区分“应用主动隐藏内容”和“本来就没内容”，统一归为 `EMPTY` / `TITLE_ONLY`。
- **服务断开**：`onListenerDisconnected` 里调用 `requestRebind`；首页有“重新连接”按钮（requestRebind + 组件禁用再启用）；`MainActivity.onStart` 发现已授权但未连接时也会请求重绑。
- **隐私**：日志只记录异常类型，不记录通知标题/正文；`allowBackup=false`，通知不会进云备份。
- `QUERY_ALL_PACKAGES` 仅用于读取其他应用的名称（个人自用，不上架）；读不到时回退显示包名。

## 目录结构

```
app/src/main/java/com/notificationassistant/ai/
├─ NotificationAssistantApp.kt   Application
├─ AppContainer.kt               手写依赖容器
├─ data/
│  ├─ NotificationRepository.kt  入库 + 去重 + 查询
│  └─ local/                     NotificationEntity / Dao / AppDatabase
├─ capture/
│  ├─ NotificationCaptureService.kt  监听服务
│  ├─ NotificationParser.kt          解析（标题/正文/应用名/哈希/完整度）
│  ├─ NotificationFilter.kt          过滤
│  └─ ListenerState.kt               服务状态 + 重绑
├─ permission/PermissionHelper.kt    权限检测与跳转
└─ ui/                           MainActivity、导航、ViewModels、5 个页面
```

## 本阶段不做

AI 接口、摘要、统计、模板、规则、图表、Prompt 自定义。
