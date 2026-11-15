# AI 智能通知助手（Android / Kotlin / Compose）

按《AI 智能通知助手：技术实现方案》实现的**可安装版本**：纯客户端、无后端，AI 使用用户自带的 OpenAI 兼容 / Anthropic 兼容 API。

- 包名：`com.example.ainotification`
- 最低系统：Android 10（API 29），目标 API 35
- 版本：1.4.0（versionCode 5）
- 安装包：`AI智能通知助手-v1.4.0.apk`（debug 签名，可直接安装自测）

---

## 1. 它做什么

1. 通过 `NotificationListenerService` 监听系统通知，解析、过滤、去重后写入本地 Room 数据库。
2. 点「立即生成摘要」（或累积 30 条自动触发），把待摘要通知交给 AI：
   一次调用同时完成**事件合并 + 摘要 + 重要度分档**，返回固定 JSON。
3. 本地完成**校验、排序、兜底**，再发布一条独立的摘要通知；原始通知不被取消、替换或隐藏。
4. 点开摘要通知 → 摘要详情，每项可展开查看对应原始通知，实现来源追溯。
5. 还支持：通知统计、对通知提问、按重要度或按应用排序、重要通知单独提醒。

## 2. 功能对照（技术方案 → 实现）

| 方案要求 | 实现位置 |
|---|---|
| 单模块工程、按包分层 | `notification/`、`ai/`、`digest/`、`data/`、`worker/`、`ui/` |
| UI：Material 3 + 可切换主题色（7 套 + 动态取色）+ 液态玻璃质感 | `ui/theme/`、`ui/components/LiquidGlass.kt`；半透明卡片、高光描边、导航栏玻璃、扫光动画 |
| 首页模块化：摘要、重要通知、统计、通知类型、问 AI | `ui/home/HomeScreen.kt`（重要项卡、7 天柱状图、类型环形图、快捷提问） |
| 统计页：类别占比 + 重要消息 | `ui/stats/StatsScreen.kt`（环形图占比、重要项标题与内容，可点进摘要） |
| 摘要通知内容 | `digest/DigestPublisher.kt`：类型占比一行 + 重要消息「标题：内容」+ 另有 N 项 |
| 底部导航：首页 / 通知 / 设置 | 统计与问 AI 从首页卡片进入，页面带返回 |
| Room 五张表 | `data/Entities.kt`：notifications / digests / digest_items / digest_item_notification_ref / app_filters |
| 去重：`(notificationKey, contentHash)` 唯一索引 | `NotificationEntity` 的 unique index + `OnConflictStrategy.IGNORE` |
| 内容可用性 `FULL/TITLE_ONLY/EMPTY/HIDDEN` | `data/Enums.kt`、`NotificationParser` |
| 摘要状态 `PENDING/DONE/SKIPPED` | `DigestStatus`，敏感/黑名单入库即 `SKIPPED` |
| DataStore 保存 AI 配置与开关 | `data/SettingsRepository.kt` |
| API Key 用 Android Keystore 加密 | `data/SecureKeyStore.kt`（AES-256-GCM） |
| 通知监听、回调不联网 | `NotificationCaptureService`（解析与入库都在后台协程） |
| 重绑与连接状态 | `onListenerDisconnected` → `requestRebind`；`ListenerState` 供首页展示 |
| 过滤顺序：自身→常驻/媒体/下载→组汇总→黑名单→NO_AI/验证码→去重 | `NotificationFilter`、`NotificationRepository.store` |
| 默认不发送：短信/银行/支付/密码管理 + 验证码 | `NotificationPrivacy.isSensitivePackage / looksLikeVerificationCode` |
| 发送前脱敏：手机号/银行卡/邮箱 | `NotificationPrivacy.redact` |
| AI 接口单一 `AIProvider` | `ai/AiModels.kt`；`FakeAIProvider`、`OpenAICompatibleProvider`、`AnthropicProvider` |
| 多平台支持 | `ai/AiProviderType.kt`：OpenAI / DeepSeek / MiMo（小米）/ Claude / 自定义 OpenAI 兼容；`AiProviderFactory` 选择 |
| 429/5xx 指数退避，401/404 不重试 | `ai/HttpTransport.kt`（最多 2 次重试） |
| 7 步流水线 | `digest/DigestEngine` + `DigestInputBuilder`/`PromptBuilder`/`DigestValidator`/`DigestSorter` |
| 单次 ≤40 条、每条 ≈200 字 | `DigestInputBuilder.MAX_BATCH/MAX_BODY_CHARS` |
| 同一事件只概括成一条 | Prompt 规则 + `FakeAIProvider` 按「应用 + 会话标题」合并 |
| 固定 JSON + 容错解析（去 ```json 围栏） | `ai/DigestParser` |
| 校验：重要度归一、剔除无效编号、未覆盖编号→「其他 N 条」 | `digest/DigestValidator` |
| 抽查：needsAction 但无硬信息 →「建议查看原文」 | `DigestValidator` + 详情页提示 |
| 排序：重要度 → 需行动 → 最新时间；可按应用分组 | `digest/DigestSorter`（`DigestSortMode`） |
| 两次失败 → 本地简版兜底（FALLBACK，保持 PENDING） | `DigestEngine.persistFallback` |
| 事务写入摘要 + 关联表 + 标 DONE | `data/DigestRepository.persist`（`withTransaction`） |
| WorkManager：唯一任务 + 网络约束 + 失败重试 3 次 | `worker/WorkScheduler`、`DigestWorker` |
| 每日清理任务，默认保留 30 天 | `CleanupWorker` |
| 暂停 AI 时任何触发直接跳过 | `DigestEngine` |
| 摘要通知：专用渠道、固定 id、InboxStyle、无 high 时静默 | `digest/DigestPublisher` |
| 重要通知单独提醒 | `DigestPublisher.alertImportant`（独立高优先级渠道 `ai_digest_important`） |
| 摘要是否发通知可开关 | 通知设置「摘要通过通知发送」 |
| 点击通知带 digestId 跳转详情 | `PendingIntent` → `MainActivity` → `Routes.digestDetail` |
| 逐项展开来源，过期显示「原始通知已过期」 | `ui/digest/DigestDetailScreen.kt` |
| 首页：监听状态、待处理数、最新摘要、立即生成、失败提示 | `ui/home/HomeScreen.kt` |
| 引导：通知使用权 / 通知权限（onResume 重新检测；电池优化提示已隐藏） | 首页两步授权 + `LifecycleEventEffect(ON_RESUME)` |
| 首次开启 AI 前说明并征得同意 | 首页 `AlertDialog` + `consentGiven` |
| 新手教程 | `ui/onboarding/OnboardingScreen.kt`：5 页引导，内含权限申请、AI Key 填写、摘要呈现设置，全程可跳过；未完成时不显示「准备就绪」 |
| 配置状态 | `ui/SetupStatus.kt` + 设置页「配置状态」卡片：可在设置里补完引导没做的项 |
| 通知列表：按应用筛选 + 关键词 LIKE 搜索 | `ui/notifications/NotificationListScreen.kt` |
| 通知详情：完整内容 + 可用性 + 所属摘要 | `ui/notifications/NotificationDetailScreen.kt` |
| 通知统计：按天 / 应用 / 重要度 | `ui/stats/StatsScreen.kt` + `data/StatsRepository.kt`（全部本地 SQL 聚合） |
| 通知问答 | `ui/ask/AskScreen.kt` + `ai/AskEngine.kt`（取最近 60 条做上下文） |
| 设置：AI / 通知（白名单、黑名单、仅保存、保留期）/ 隐私 | `ui/settings/*` |
| 清空数据二次确认 | 隐私设置 `AlertDialog` |
| 导出诊断信息（不含通知内容） | 隐私设置「导出诊断信息」 |
| 日志不输出正文与 Key | 全工程无正文日志；诊断信息仅数量与状态 |

## 3. 构建

环境：JDK 17 + Android SDK（platform 35、build-tools 34.0.0）。用 Android Studio 直接打开工程，或命令行：

```bash
./gradlew :app:assembleDebug
```

产物：`app/build/outputs/apk/debug/app-debug.apk`。

单元测试：

```bash
./gradlew :app:testDebugUnitTest
```

> `debug.keystore` 仅用于本地自测（口令 `android`，alias `androiddebugkey`）。正式分发请换成自己的签名。

## 4. 安装与首次使用

1. 安装 APK，首次打开进入 5 页新手引导：权限申请 → AI Key（可跳过）→ 摘要呈现设置 → 完成。每页都可以「跳过」，之后在设置里补。
2. 若跳过了授权，首页会提示补授权：
   - **通知使用权** → 系统「通知使用权」列表里勾选「AI 智能通知助手」；
   - **发送通知权限**（Android 13+）。
   国产系统建议在手机管家的「自启动管理」里手动允许（界面不再提示，应用也检测不到这项）。
3. 让手机收到几条普通通知，首页会显示「待摘要 N 条」。
4. 点「立即生成摘要」：
   - 默认是 **Fake AI**，不联网，用来验证「解析 → 校验 → 排序 → 发布」全链路；
   - 验证通过后到「设置 → AI」选择服务商（OpenAI / DeepSeek / MiMo / Claude / 自定义），填 Base URL、Model、API Key，点「测试连接」，再生成。
5. 通知栏出现摘要（有重要项时还会单独提醒），点开进入摘要详情，可逐项展开来源通知。
6. 「统计」页看 7 天趋势、应用分布、重要度分布；「问 AI」页可以直接问「今天有什么重要的事？」。

## 5. 验收路径（对应方案第 13 节）

| 步骤 | 验证方法 |
|---|---|
| 2 采集 | 真实通知入库；同一通知反复更新不产生重复行；本应用摘要通知不会出现在通知列表 |
| 3 稳定性 | 首页显示监听状态；重启后能恢复；断开可点「重新绑定」 |
| 4 FakeAI 链路 | 点按钮 → 出现摘要通知 → 点开看到来源通知 |
| 5 真实 AI | 四家服务商任选其一配置后能生成；断网时不崩溃、不丢数据（任务 retry） |
| 6 隐私 | 短信/验证码类不会出现在请求里；暂停 AI 后无任何请求 |
| 7 自测 | 日常使用一周，观察摘要质量 |

## 6. 隐私与安全说明

- 数据只存在应用私有目录，`allowBackup=false`，`data_extraction_rules` 排除数据库与文件参与云备份。
- API Key 由 Android Keystore 中的 AES 密钥加密，密钥不可导出；界面只显示末 4 位。
- 发送给 AI 前先本地过滤（敏感应用、验证码、空正文）+ 脱敏（手机号、银行卡号、邮箱）。
- 通知正文放在 `<notifications>` 标签，并在 system prompt 中声明为数据而非指令；重要度只依据内容本身。
- 客户端无法防住设备被攻破后的数据提取，本应用不宣称绝对安全。

## 7. 已知限制

- 未在真机/模拟器上运行验证（构建环境无设备）；通知可用性、厂商后台策略需要实机确认。
- 各服务商模型名会更新，默认值只是起点，请在设置里改成官方当前可用的模型名。
- Claude 走 Anthropic Messages 协议，不支持 `response_format`，只靠 Prompt 约束输出，解析层已做容错。
- MiMo 默认使用 `https://api.xiaomimimo.com/v1`（OpenAI 兼容），模型名以官方文档为准。
- 通知问答的上下文取最近 60 条通知，暂不做向量检索；数据量很大时可后续换成 FTS/检索。
- 定时摘要（PeriodicWorkRequest）、摘要详细程度分档属于 V1.0 之后的方向。

## 8. 目录结构

```
app/src/main/java/com/example/ainotification/
├── AssistantApplication.kt      # 手写 AppContainer（依赖装配）
├── MainActivity.kt              # 单 Activity + 引导页 + 通知跳转
├── ai/                          # 多平台 Provider、Prompt、解析、问答
├── data/                        # Room 实体/DAO/数据库、DataStore、Keystore、统计
├── digest/                      # 流水线：输入、校验、排序、发布、引擎
├── notification/                # 监听服务、解析、过滤、隐私、仓库
├── ui/                          # 首页、通知、统计、问 AI、设置、引导、详情
└── worker/                      # 摘要 Worker、清理 Worker、调度
```
