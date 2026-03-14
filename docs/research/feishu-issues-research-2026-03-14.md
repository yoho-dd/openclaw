# 飞书 (Feishu/Lark) 问题反馈深度调研

> 调研日期：2026-03-14
> 数据来源：GitHub Issues, PRs, 社区反馈, Web 搜索

## 背景

OpenClaw 已内置飞书插件 `@openclaw/feishu`（93 个源码文件，36 个测试文件），由社区贡献者 @m1heng 维护。飞书是 OpenClaw 社区中反馈最活跃的渠道之一，GitHub 上相关 issue 超过 **170 个**（含 PR），去重后飞书专属 issue 约 **100+ 个**，open PR 超过 **50 个**。

## 一、核心问题汇总表

### 1. 插件重复注册/冲突（最高频，10+ issues）

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 升级后出现 "duplicate plugin id detected" 警告，内置插件与旧安装的社区插件冲突 | #45951, #45881, #45805, #45776, #45603, #45154, #44848, #40854, #37548, #35884, #34394, #33239, #30172, #29632 | **未解决** | 最高频问题，每个版本升级都有人报告。根因：内置 bundled 插件与用户手动安装的社区插件共存时 ID 冲突 |
| `openclaw doctor --fix` 错误启用内置 feishu 插件，与 `openclaw-lark` 社区插件冲突 | #44722 | **未解决** | doctor 修复逻辑不感知第三方 lark 插件 |

### 2. 消息收发问题

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 升级后飞书通道停止收消息，长连接未建立 | #40854, #44197, #40802 | **未解决** | 与插件重复注册关联 |
| WebSocket 长连接失败 "system busy" | #42354 | **未解决** | 飞书服务端限流 |
| 消息重复送达（一条消息多次回复） | #43969, #37477(已关闭) | **部分解决** | #37477 已修复去重逻辑，但 #43969 仍 open |
| 流式输出开启后重复发送两条消息 | #44085, #43612, #41195, #38872, #38824, #34108 | **未解决** | 长回复场景下卡片重复下发，WebSocket 重连也导致重复处理 |
| 流式卡片合并无关回复 | #43704, #38938 | **未解决** | streaming + tool calls 场景 |
| 群聊中 slash commands 不被拦截 | #42228 | **未解决** | |
| /new 启动问候语未送达 | #39408 | **未解决** | |
| 消息以冒号结尾时被截断 | #44776 | **未解决** | |
| 命令有效但普通消息无响应 | #33943 | **未解决** | |
| DM (p2p) 消息被路由到 webchat | #44866, #33307 | **未解决** | DM 自动回复路由也受影响 |
| Subagent announce 超时，未送达用户 DM | #44649 | **未解决** | @mention 转发场景 |
| Interactive Card 内容解析缺失 | #41607 | **未解决** | |
| @all 不应识别为 bot mention | #37706 | **未解决** | 群聊 @所有人 误触发 |
| Cron 投递到飞书失败 "requires a target" | #40531 | **未解决** | |
| 原始 provider error 泄露给用户 | #45050, #41435 | **未解决** | 内部 tool-call payload 也会泄露 |
| DM 收到消息但 agent 超时无响应 | #41046 | **未解决** | |
| sendMessage 无 HTTP 超时导致队列死锁 | #36412 | **已解决** | |

### 3. 图片/文件/媒体问题

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 发送图片只能作为附件，无法内联预览 | #22608, #35575(已修) | **部分解决** | |
| 图片导致 gateway 崩溃 | #44778 | **未解决** | |
| 图片下载后文件名与 Agent 路径不匹配 | #44792 | **未解决** | |
| 图片分析不工作 | #42257 | **未解决** | |
| read image tool 结果在 payload 中丢失 | #41744 | **未解决** | |
| message tool filePath 发送成功但对方未收到 | #25200, #42902 | **未解决** | |
| 中文文件名被 URL 编码乱码 | #44988, #43840, #40770, #33912(已修) | **部分解决** | 修了又复发 |
| 发送语音消息报错 | #41996 | **未解决** | |
| 链接中下划线导致超链接显示不全 | #41860 | **未解决** | |

### 4. 多账号/多 Agent 路由

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 多 Agent 路由：所有消息路由到同一 agent | #45158 | **未解决** | |
| 多账号路由失败，总路由到默认账号 | #16354 | **未解决** | |
| 多账号 Token 隔离问题 | #13916 | **已解决** | |
| 多机器人无法收发信息 | #41163 | **未解决** | |
| 群聊多机器人身份混乱 | #40194, #38778 | **未解决** | 机器人间 @mention 也不响应 |
| accountId binding 不生效 | #43633, #26066, #33418 | **未解决** | open_id 跨 app 错误 |
| 回复目标变成 "heartbeat" | #39756 | **未解决** | 路由 ID 异常 |
| Doc tools 应使用当前 agent 凭证 | #44975 | **未解决** | Critical |
| 多账号 session account 优先级讨论 | #45659 | **讨论中** | 设计决策 |
| dmPolicy pairing 模式配对未持久化 | #43289 | **未解决** | |
| dmPolicy 默认值静默改变 | #17741 | **未解决** | 需迁移方案 |
| 跨渠道重复消息投递 | #45514 | **未解决** | |

### 5. 连接/稳定性

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| Session lock 文件导致请求超时 | #45726, #43322 | **未解决** | Card API 失败可导致 13+ 小时 session 锁死 |
| Embedded agent LLM 请求超时 | #42079 | **未解决** | |
| AxiosError 升级 3.8 后超时 | #42255 | **已解决** | |
| 301 重定向循环 (HTTP/1.1) | #44572 | **已解决** | |
| WebSocket 断连 1006 no reason | #19248 | **未解决** | 长期存在 |
| 长连接配置失败 | #35202, #37415 | **未解决** | Windows WSL2 场景也受影响 |
| Gateway 重启静默丢失 in-flight session | #38836 | **未解决** | |
| 群消息 bot-info probe 超时导致消息丢弃 | (PR #43788 待合并) | **未解决** | |
| tenant_access_token 返回 undefined | #44677 | **未解决** | |
| 升级缺少 @larksuiteoapi/node-sdk 依赖 | #39815, #39800 | **未解决** | 依赖未正确打包 |
| probe 健康检查无缓存，月耗 43k API | #43373 | **未解决** | 资源浪费 |
| 缺少 bot_p2p_chat_entered_v1 事件处理 | #42351 | **未解决** | |

### 6. 功能请求

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 支持 user_access_token | #29895 | **未实现** | 解锁更多个人 API |
| ACP Thread Bindings for Feishu | #44823, #40097 | **未实现** | |
| 日历和邮件 API 集成 | #32618, #43696 | **未实现** | |
| Doc comment 读取 API | #45051, #44741 | **未实现** | |
| 消息附件下载工具 | #44149, #41951 | **未实现** | |
| 原生语音消息发送 | #44246 | **未实现** | |
| DM 话题/线程 session 隔离 | #35478 | **未实现** | |
| outbound adapter 忽略 threadId | #35598 | **未解决** | |
| Bitable tools 可配置开关 | #45195 | **未实现** | |
| sendMedia 能力暴露 | #43948 | **未实现** | |
| Wiki 分页不生效 | #37626 | **未解决** | |
| Lark 知识库 space_id 被篡改 | #45301 | **未解决** | |
| /models 只显示 provider 不显示 model | #40722 | **未解决** | |
| 飞书配置在 Web UI 中不显示 | #39911 | **未解决** | |
| 群聊错误消息抑制 | #44598 | **未实现** | |
| 插件 tools 未暴露给 embedded agent | #39920 | **未解决** | |
| 文档创建是否为一等能力 | #43172 | **讨论中** | |
| Bitable 附件上传 + 记录删除 | #43729 | **未实现** | |
| 读取飞书 Sheets 电子表格 | #39805 | **未实现** | PR #43868 待合并 |
| Drive 上传功能 | #31372 | **未实现** | |
| 用户身份创建文档 | #30578 | **未实现** | 需 user_access_token |
| 思维导图 (MindNote) 支持 | #29362 | **未实现** | |
| 飞书用户搜索工具 | #28242 | **未实现** | |
| 群聊自动轮询 | #44314 | **未实现** | |
| 群聊多 Agent 协作 | #37374 | **未实现** | |
| 用户繁忙时发送排队反馈 | #39669 | **未实现** | |
| 健康监控 stale-socket 阈值可配 | #35532 | **未实现** | |

### 7. 流式卡片 (Streaming Card) 专项

| 问题/反馈 | Issues 链接 | 是否解决 | 备注 |
|---|---|---|---|
| 聊天后续调用变得分块且无法编辑（3.12 回退到 3.2 才解决） | #45299 | **未解决** | 严重回归 |
| 卡片 "table number over limit" 无 fallback | #43690 | **未解决** | |
| 常规消息发送失败，发卡片后才恢复 | #35556 | **未解决** | |
| 流式卡片格式损坏 | #40028 | **已解决** | |

## 二、重要待合并 PR（社区贡献）

| PR | 标题 | 解决的问题 |
|---|---|---|
| #45900 | 抑制误报 duplicate id 警告 | 插件冲突 |
| #45904 | 处理 WebSocket unhandled rejections 防僵尸连接 | 连接稳定性 |
| #45948 | 视频文件用 msg_type "file" 而非无效 "media" | 媒体发送 |
| #45936 | 改进 interactive card 文本提取 | 消息解析 |
| #45674 | WSClient 增加 PingInterval/PingTimeout 配置 | 连接稳定性 |
| #45673 | tools 优先使用 session account 而非 defaultAccount | 多账号路由 |
| #45278 | 线程/话题模式启用流式卡片回复 | 功能增强 |
| #45246 | 按 session accountId 解析 tool account | 多账号路由 |
| #45209 | Bitable tools 可配置开关 | 功能增强 |
| #44829 | 修复冒号结尾消息截断 | 消息截断 |
| #44735 | doctor 跳过 feishu auto-enable（存在替代插件时） | 插件冲突 |
| #44567 | 尊重 channels.* 配置防重复 plugin id | 插件冲突 |
| #44256 | @all 不作为 bot mention | 群聊误触发 |
| #44178 | 飞书文档评论功能 | 功能增强 |
| #44118 | 群聊 slash commands 绕过 mention gate | 命令不生效 |
| #43916 | 解码 URL 编码的文件名 | 中文乱码 |
| #43868 | 飞书 Sheets 读取工具 | 新功能 |
| #43862 | 防止流式卡片合并无关回复 | 消息重复 |
| #43713 | 飞书日历工具 | 新功能 |
| #43170 | ACP persistent bindings 扩展到飞书 | 新功能 |
| #42940 | 防止 multi-final 回复卡片重复 | 消息重复 |
| #42436 | WSClient 启用 autoReconnect | 连接稳定性 |
| #41849 | HTTP client 支持 HTTPS proxy | 代理支持 |
| #41678 | 入站消息包含 interactive card 内容 | 消息解析 |
| #40936 | 飞书完整 ACP agent 支持 | 新功能 |

## 三、问题分类统计

| 类别 | Open 数量 | 已解决 | 待合并 PR |
|---|---|---|---|
| 插件重复注册/冲突 | 15 | 0 | 3 |
| 消息收发/重复 | 18 | 3 | 5 |
| 图片/文件/媒体 | 11 | 3 | 2 |
| 多账号/多Agent路由 | 13 | 1 | 3 |
| 连接/稳定性 | 11 | 2 | 3 |
| 流式卡片 | 4 | 1 | 2 |
| 功能请求 | 28 | 0 | 7 |

## 三、竞品对比

| 产品 | 飞书支持 | 特点 | 与 OpenClaw 差异 |
|---|---|---|---|
| 飞书官方插件 (2026.3.6) | 原生 | 飞书团队自研，用户身份操作文档/日历/任务 | 仅飞书生态，非通用 AI gateway |
| LangBot (原 QChatGPT) | 支持 | 10+ IM 平台，集成 Dify/n8n/Coze | 偏中国 IM 生态 |
| AstrBot | 支持 | QQ/微信/Telegram/飞书，配置简单 | 功能较轻量 |
| ZeroClaw | 支持 | Rust 实现，双域名支持 | 性能好但生态小 |
| Moltbot | 支持 | AWS CloudFormation 企业级部署 | 闭源，云绑定 |
| xzq-xu/openclaw-plugin-feishu | 社区 | 生产级社区飞书插件 | OpenClaw 生态内替代 |
| 飞书苗搭一键部署 | 原生 | 2 分钟低代码部署 | 降低部署门槛 |

## 五、社区生态

### 第三方插件/工具

| 项目 | 描述 |
|---|---|
| [xzq-xu/openclaw-plugin-feishu](https://github.com/xzq-xu/openclaw-plugin-feishu) | 生产级社区飞书插件（npm: @xzq-xu/feishu） |
| [m1heng/clawdbot-feishu](https://github.com/m1heng/clawdbot-feishu) | 早期社区插件 |
| [shubhankargokhale/feishu-openclaw](https://github.com/shubhankargokhale/feishu-openclaw) | 独立桥接，无需公网服务器 |
| [abca12a/lark-openclaw](https://github.com/abca12a/lark-openclaw) | Lark 国际版 webhook 模式插件 |

### 中文教程/指南

- [飞书官方公告](https://www.feishu.cn/content/article/7613711414611463386) — 飞书官方 OpenClaw 插件发布
- [菜鸟教程](https://www.runoob.com/ai-agent/openclaw-feishu.html) — 手把手配置指南
- [腾讯云指南](https://cloud.tencent.com/developer/article/2626160) — 详细安装教程
- [阿里云指南](https://help.aliyun.com/zh/simple-application-server/use-cases/openclaw-integrated-fly-book) — 云部署方案
- [飞书苗搭一键部署](https://www.feishu.cn/content/article/7615218249831058381) — 2 分钟低代码部署
- GitHub #45746 — 42.3 万字完整中文教程

### 注意事项

- 约 30 个 spam issue（飞书交流群推广）已被 `r: spam` 标签关闭
- 飞书用户社区已通过飞书群活跃运营

## 六、关键结论

1. **最紧急**：插件重复注册（15 open），每次升级都有报告，严重影响新用户体验和升级流程。目前有 3 个 PR 尝试修复但尚未合并
2. **第二优先**：消息重复/流式卡片问题（18 open），streaming card 场景尤其严重，有用户被迫回退版本
3. **第三优先**：图片/文件处理（中文文件名乱码反复修了又复发，图片上传/预览链路多处断裂）
4. **第四优先**：多账号/多 Agent 路由（13 open），企业用户刚需但路由逻辑有多处 bug
5. **功能缺口**：user_access_token、日历/邮件 API、附件下载、Sheets 读取、线程绑定等竞品已有而 OpenClaw 缺失。但已有对应 PR 待合并（日历、Sheets、ACP bindings 等）
6. **社区非常活跃**：大量中文教程、第三方插件、飞书官方也已发布自研插件，显示飞书渠道对 OpenClaw 的重要性
7. **积压严重**：超过 25 个社区贡献的 PR 待审核/合并，贡献者热情高但合并速度不匹配
