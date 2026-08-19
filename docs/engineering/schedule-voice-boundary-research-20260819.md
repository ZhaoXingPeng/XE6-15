# 语音日程边界复核（2026-08-19）

结论：之前列出的“每月 31 日、2 月 29 日、最后一个工作日、从下月开始修改”等例子只是边界的一部分。行业产品真正稳定的边界由六个维度组成：事件形态、重复模式、重复范围、单次例外、时区/本地时间和提醒/对话抢占。本报告把行业事实与 VoiceLife 当前取舍分开，避免把“当前没做”误写成“行业不支持”。

动作：日程联调必须按本报告的矩阵补齐测试；本阶段不因测试用例临时扩展产品能力。所有未开放能力都要得到清晰澄清且不写库；所有已经承诺的能力都要验证日程事实、语音播报、状态机和屏幕显示四者一致。

## 1. 调查范围与证据

本次直接读取了 8 个官方来源，并以当前仓库代码交叉核对。环境没有可用的 WebSearch/WebFetch MCP，因此使用 HTTPS 直读官方页面；没有把搜索摘要或二手文章当作事实依据。

| 来源 | 直接证据 | 对本项目的意义 |
| --- | --- | --- |
| [RFC 5545 §3.3.10](https://www.rfc-editor.org/rfc/rfc5545.txt) | 无效日期（如 2 月 30 日）和不存在的本地时间必须忽略，且不能计入 occurrence；`RECURRENCE-ID` 定位原始实例，`RANGE=THISANDFUTURE` 表示从该实例及以后生效 | 不能把短月 31 日自动改成月末；实例修改必须保留原始发生时间 |
| [Google Calendar recurring events](https://developers.google.com/calendar/api/guides/recurringevents) | 单次实例、整条系列、`this and following` 是不同操作；实例用 `originalStartTime` 唯一定位；把整条系列拆成大量单次例外会造成性能和通知问题 | “从下个月开始都改”是合法的系列操作，不是语病；当前 VoiceLife 应澄清并拒绝执行，而不是猜成“改一次” |
| [Google Calendar `events.instances`](https://developers.google.com/workspace/calendar/api/v3/reference/events/instances) | 系列实例查询有 `timeMin/timeMax` 窗口；取消的 recurring occurrence 是否返回由 `showDeleted` 等参数控制，不能和“规则不存在”混为一谈 | 查询、取消和历史审计要分别定义；窗口外的实例不能靠“最近几条”猜测 |
| [Google Calendar event resource](https://developers.google.com/workspace/calendar/api/v3/reference/events) | 事件资源把 `start/end`、`timeZone`、`transparency`（是否占用忙闲）、`visibility`、`attendees`、`reminders` 和 `recurrence` 分开建模；“有标题”不等于“可执行的定时提醒” | VoiceLife 当前只有时间、地点、备注和单一到点任务；全天、忙闲、参与者、权限和提前提醒偏移必须显式列为未开放能力 |
| [Microsoft Graph recurrencePattern](https://learn.microsoft.com/en-us/graph/api/resources/recurrencepattern) | 除 daily/weekly/absolute monthly/yearly 外，还支持 relative monthly/yearly；`index` 可为 first/second/third/fourth/last | “每月第二个周二”“每年最后一个周一”是行业标准能力，不应被描述为不合理输入；它们只是当前未开放 |
| [Microsoft Graph recurrenceRange](https://learn.microsoft.com/en-us/graph/api/resources/recurrencerange) | 重复范围可为 `endDate`、`noEnd` 或 `numbered`；`recurrenceTimeZone` 单独定义范围时区 | `occurrence_count` 是常见产品能力；VoiceLife 当前拒绝它属于产品范围决定，不是行业事实 |
| [Microsoft Graph event resource](https://learn.microsoft.com/en-us/graph/api/resources/event) | 系列主事件、普通 occurrence、被修改的 exception、取消 occurrence 是不同 `type`；exception 保留 `originalStartTime`，还可能改变标题、时间、地点和参与者 | 日程事实层必须保留原始 occurrence 身份，修改后的展示对象不能覆盖定位键 |
| [Microsoft Graph event](https://learn.microsoft.com/en-us/graph/api/resources/event) / [reminder](https://learn.microsoft.com/en-us/graph/api/resources/reminder) | 协作事件包含组织者、参与者、响应状态、可用性和提醒设置；提醒是事件属性而非文本备注 | VoiceLife 不应把“提醒我”误写成协作邀请或忙闲占用；无这些字段时要在语音中说明能力范围 |
| [Apple EventKit EKRecurrenceRule](https://developer.apple.com/documentation/eventkit/ekrecurrencerule.md) | 规则可按日期或最大次数结束；支持复杂的星期/月/年过滤和 set position；`EKSpan` 区分单次修改与未来全部实例 | 单次、未来实例、整条规则必须有不同确认语义；复杂相对周期应进入后续能力，而不是语音层硬编码 |
| [Apple EventKit EKRecurrenceEnd](https://developer.apple.com/documentation/eventkit/ekrecurrenceend.md) | 重复结束条件是结束日期或 occurrence count 二选一的规则对象 | 语音确认必须复述“到哪天结束”或“共几次”，不能只播报“已设置周期” |
| [Apple EventKit EKEvent](https://developer.apple.com/documentation/eventkit/ekevent) / [EKAlarm](https://developer.apple.com/documentation/eventkit/ekalarm) | 日历事件、全天标记、地点、参与者和一个或多个 alarm 是独立字段；全天事件不应被当作当天 00:00 的普通会议 | VoiceLife 当前没有显式全天/参与者/多 alarm 数据模型；“某天有件事”只能按无时刻记录处理，不能承诺 00:00 播报 |
| [OpenAI Realtime turn handling](https://platform.openai.com/docs/guides/realtime-model-capabilities) | 用户打断时要取消进行中的响应、停止播放，并按实际播放位置截断未播放的内容 | 提醒抢占和用户“停一下”不能只改 UI 状态，必须同时清理生成、音频队列和上下文 |
| [LiveKit turn handling](https://docs.livekit.io/agents/build/turns/) | VAD 负责说话开始/打断，turn detector 判断一句话是否结束；对误打断可恢复原播报 | 长句、停顿、背景声和“嗯”不能只用一个固定静音阈值判断；压测要覆盖误打断恢复 |
| [Home Assistant voice pipeline](https://developers.home-assistant.io/docs/voice/overview) | Assist Pipeline 将语音转文字、会话/意图处理和文字转语音拆成独立集成 | 屏幕状态应对应阶段事件，工具失败不能短暂显示成功 |

审计记录：查询 8 次，收到 8 个官方页面，引用 8 个来源；没有连续失败。Apple 页面采用官方 Markdown 版本，其他页面采用官方 HTML/RFC 原文。

## 2. 重新划定的真实边界

### 2.1 事件不是只有“有时间/没时间”

行业产品至少区分三种对象：

- **定时事件**：有开始时刻，可选结束时刻；结束必须晚于开始，同一天跨午夜要明确表达。
- **全天事件**：只有日期，没有具体时刻；提醒语义通常是当天或提前若干天，而不是在 00:00 播报。
- **待办/提醒**：重点是“到点通知”，不一定占用时间段，也不应参与会议冲突判断。

VoiceLife 当前 `schedule` 同时允许缺少 `start_time`，但提醒服务只对有可执行时间的日程注册任务。这个行为必须在语音层说清楚：缺日期/时间时追问；用户明确说“某天有件事”时可以保存为无时刻记录，但不能承诺到点播报。全天事件和待办是否要成为独立类型，暂列产品决策，不用句子特判掩盖。

### 2.2 周期模式与周期范围是两个问题

行业产品把“怎么重复”和“重复到什么时候”分开。模式包括固定周几、固定月日、相对月内位置（第二个周二、最后一个周一）和固定月日；范围包括结束日期、无限重复、次数上限。

因此：

- “每月 31 日”在二月没有发生点；不得改成二月最后一天。
- “每年 2 月 29 日”在平年跳过；跳过不消耗次数。
- “每月最后一个工作日”不是“每月最后一天”，需要工作日/节假日规则，当前不能降级。
- “重复 10 次”是行业正常语义；当前实现拒绝 `occurrence_count`，必须返回明确不支持且不写入。
- “每两周周一”是 `interval=2` 的 weekly 规则，不是每月两次。

### 2.3 修改范围必须先确认

同一句“把它改到十点”可能指：

1. 只改某一个 occurrence；
2. 从该 occurrence 起修改后续；
3. 修改整条规则（包括已经发生的历史实例是否保留）。

Google、RFC 和 Apple 都把这些范围分开。VoiceLife 当前只有“单次例外”和“整条规则”接口，没有安全的 `THISANDFUTURE` 系列拆分，因此用户说“从下个月开始都改”必须追问并告知当前不支持；不能静默改一次，也不能直接重建整条规则导致历史事实变化。

### 2.4 查询必须按用户窗口展开，而不是返回“最近几条”

“这一天有什么”是日历的基本查询，不是特殊用例。查询结果必须：

1. 在请求的本地日期窗口内展开所有周期 occurrence；
2. 应用 skip/modify 例外后再返回；
3. 合并已物化实例、未物化 occurrence 和提醒状态；
4. 对跨午夜事件明确归属（按开始日期还是覆盖日期）。

基线代码存在可复现的结构性风险：`ScheduleRuleService::query_schedule_rules` 每条规则固定只展开最近 3 个 occurrence；`schedule.query` 虽然解析了 `start_date/end_date`，却没有把日期窗口传入规则服务，只能对这 3 个结果做过滤。这会让“半年后的某天有什么”错误返回空，不应归因给 STT。本分支已将日期窗口贯穿到规则服务，并在有窗口时按最多 128 条的有界上限展开；无窗口调用仍保持默认 3 条兼容行为。

### 2.5 本地时间、时区和 DST 是语义的一部分

RFC 规定不存在的本地时间 occurrence 要忽略；Microsoft 还把 recurrence timezone 作为独立字段。真实产品还要处理：

- “今天/明天/下周一”以设备当前时区解释；
- 12/24 小时制和“下午三点半”不能丢失半小时；
- 跨午夜事件不能被截成负时长；
- 用户说纽约、东京或夏令时，不能静默换算成北京时间；
- 设备时钟未同步时，创建和提醒都必须可见地标记不确定性。

VoiceLife 当前周期计算固定 UTC+8、无 DST。这个限制可以保留，但必须在澄清和播报中明确，不得让模型假装支持 IANA 时区。

### 2.6 提醒是主动事件，不是普通对话回复

提醒触发时可能正处于待机、聆听、思考、TTS 播放、网络重连或错误恢复。需要预先确定：

- 当前对话优先还是提醒优先；
- 提醒能否被“停一下”打断；
- 被打断后是从剩余文本继续、重新播报，还是标记为已读；
- 同一提醒重启/重连后是否去重；
- 修改或取消日程是否撤销已排队的提醒。

OpenAI Realtime 的可验证做法是同时取消响应、停止音频并截断未播放上下文；LiveKit 还区分真实打断和误打断。VoiceLife 的测试必须记录 generation、音频队列、屏幕和提醒任务 ID，不能只看最终一句 TTS。

### 2.7 产品 API 已经把“边界对象”拆开

这次补查 Google、Microsoft 和 Apple 的原始 API 后，可以确定边界不只在日期计算，还在对象身份和查询语义：

- 查询必须带窗口和状态语义。Google 的 `events.instances` 明确提供 `timeMin/timeMax`，并允许调用方决定是否返回已取消 occurrence；“没有返回”不能直接解释为“规则没有这一天”。
- Microsoft 用 `seriesMaster`、`occurrence`、`exception` 和取消 occurrence 区分生命周期。被改期的实例仍保留 `originalStartTime`，因此 VoiceLife 的例外输出保留原始时间是契约要求，不是实现细节。
- Apple 把规则终止条件建模成独立的 recurrence end（结束日期或次数），而不是把次数塞进自然语言备注。

因此，后续测试不再把所有失败归因于 STT：同一句“把下周二那场改到十点”，必须先验证模型是否识别了目标系列和原始 occurrence，再验证日程层是否正确写入 exception，最后验证提醒任务、语音播报和屏幕是否同步。

### 2.8 协作、忙闲和提醒偏移也是产品边界

Google Calendar、Microsoft Graph 和 EventKit 都把协作与提醒建模为独立对象，而不是把它们塞进标题或备注。至少要分开验证以下语义：

- **全天事件**：只有日期，不占用一个从 00:00 开始的时间段；“周五全天出差”不能被注册成周五零点的语音提醒。
- **忙闲与冲突**：有开始/结束时间的事件才参与本地冲突检查；地点、备注或“提醒我”本身不产生占用。
- **参与者与权限**：邀请、组织者、响应状态属于协作系统；当前 VoiceLife 没有账户/权限/发送邀请能力，必须明确拒绝或转人工确认，不能假装创建成功。
- **提醒偏移**：行业产品允许事件前若干分钟/小时的 alarm；VoiceLife 当前只有一次“到开始时刻”的提醒任务，用户说“提前十分钟提醒”属于未开放能力，不能静默改成到点提醒。
- **删除与撤销**：取消事件必须撤销尚未触发的提醒任务；已经播报的提醒不能被事后删除伪造成从未发生。

## 3. 现有假设的修正表

| 旧说法 | 调查后的准确说法 | 对当前版本的动作 |
| --- | --- | --- |
| “次数上限不是行业能力” | 行业常见；Graph、Apple、RFC 都有对应概念 | 保持当前拒绝，但新增清晰拒绝和无写入测试；后续单独做产品提案 |
| “每月第二个周二不合理” | 是标准 relative monthly 能力 | 当前拒绝，不得降级成固定日期；列入后续能力候选 |
| “从下个月开始修改是用户说错了” | 是合法的系列拆分语义 | 当前没有接口，必须澄清并拒绝执行；未来设计 `THISANDFUTURE` |
| “查询远期日期是特殊边界” | 是日历查询基本能力 | 修复规则查询窗口和例外合并，不能在语音层重试掩盖 |
| “每月 31 日可自动顺延月末” | RFC 明确要求无效 occurrence 忽略 | 保持跳过规则，增加跨年回归 |
| “提醒只是 TTS 的另一种来源” | 提醒是独立主动事件，有去重、抢占、恢复和撤销 | 按提醒生命周期做状态机压测 |

## 4. 新增测试矩阵

每条用例都要用百炼 TTS 生成真实音频，经串口注入 SparkBot，记录 STT 明文、MCP 参数/结果、状态机事件、屏幕两行文本和提醒任务状态。失败按“语音交互/显示/状态机”或“日程事实/持久化/提醒排程”归类，不为单句话增加分支。

| 类别 | 语音输入或环境 | 预期 |
| --- | --- | --- |
| 事件形态 | “下周三有个发布会”；“下周三 14:00 到 15:30 发布会”；“那天提醒我带资料” | 缺时刻时保存/提醒语义必须明确；有时间时参与冲突检查；不会把待办当占时事件 |
| 全天与待办 | “周五全天出差”；“周五提醒我带护照”；“周五 9 点到 10 点开会” | 全天不转换成 00:00 定时事件；待办不参与冲突；当前无提前提醒偏移时要明确拒绝或降级为无时刻记录 |
| 自然语言时间 | “下午三点半”“今晚十二点”“下下周一”“月底” | 复述完整日期和时刻；“月底”必须澄清是最后一天还是最后工作日 |
| 不完整/歧义 | “明天开会”“把那个改到三点”“取消刚才的” | 只追问最小缺失字段；候选多时不得误改/误删；补答后只执行一次 |
| 事件边界 | 结束早于开始；跨午夜；无结束时刻；重复创建同名同刻事件 | 拒绝非法时段；跨午夜保持正时长；重复创建要提示而非静默重复 |
| 冲突与忙闲 | 同时创建两个有时间事件；同地点但不重叠；“提醒我参加”但无时间 | 只对真正重叠的有时刻事件报冲突；地点相同不等于时间冲突；无时刻记录不伪造忙闲 |
| 周期模式 | 每两周周一；每月第二个周二；每月最后一个工作日；每年最后一个周一 | 当前支持的规则正确创建；未开放规则明确拒绝且不写库 |
| 周期范围 | 直到某日；重复 10 次；无限重复；结束日早于开始日 | 合法范围复述完整；`occurrence_count` 当前拒绝；非法范围不创建 |
| 无效日期 | 每月 31 日跨二月；每年 2 月 29 日跨平年；DST 跳过的本地时刻 | 无效 occurrence 跳过且不计数；不自动改成月末或邻近时刻 |
| 例外范围 | 只改本次；从下月起都改；整条规则改时间；改后再取消本次 | 单次例外保留原始时间；未支持的后续拆分必须澄清；取消后不残留提醒 |
| 远期查询 | 创建半年后的周期规则，查询指定日期；查询窗口含跨午夜事件 | 返回窗口内完整 occurrence，已应用例外；不受最近 3 条上限影响 |
| 状态过滤 | 查询 active/cancelled/completed/all；规则已取消但历史实例存在 | 结果与状态过滤一致；播报不把历史实例说成未来提醒 |
| 对象身份 | 系列主事件、普通 occurrence、改期 exception、取消 occurrence | 查询和操作都保留 rule/series 与 original_start_time；取消实例不会被误报为规则不存在 |
| 提醒抢占 | 到点时待机、聆听、思考、播报、重连；“停一下”“继续”“取消提醒” | 只播报一次；generation 单调；音频与屏幕恢复到可解释状态；取消撤销后续任务 |
| 提醒偏移 | “提前十分钟提醒我”；“提前一天提醒”；修改/取消已排提醒 | 未开放的提前偏移明确拒绝且不写入错误任务；修改/取消后旧 task id 不再触发 |
| 协作与权限 | “邀请小王参加”；“把会议设为忙碌”；无权限日历 | 当前能力明确拒绝，不创建本地假事件；错误不进入成功屏幕或成功 TTS |
| 重启/重连 | 提醒前重启；提醒中断网后恢复；同一事件重复收到回调 | 不重复播报、不漏播；任务 ID 去重；恢复后状态从事实重新计算 |
| 显示与语音 | 长标题、数字日期、英文地点、中文标点、emoji、两行边界 | 固定一或两行滚动；不覆盖表情；断句不切 UTF-8；屏幕与播报对象一致 |

## 5. 对 VoiceLife 的明确结论

本阶段继续承诺：UTC+8 下的一次性事件、daily/weekly/固定月日/月末/yearly、结束日期、单次 skip/modify、窗口查询和到点提醒。`occurrence_count`、relative monthly/yearly、IANA 时区、DST、节假日工作日和 `THISANDFUTURE` 都是后续产品决策，不在语音层偷偷模拟。

优先级最高的日程根因不是“模型没理解特殊说法”，而是查询窗口没有贯穿到规则展开，且例外结果没有在返回前合并。本分支已修复这两个契约问题，并用 Host 测试覆盖远期窗口、skip 和移动后的 modify；仍需在百炼多轮实板压测中验证序列化、语音播报和屏幕呈现没有新的跨层问题。

## 6. 复查触发条件

只要新增全天事件、次数上限、relative monthly/yearly、IANA 时区、节假日规则、系列后续修改或多提醒，就必须新增产品决策和架构契约，并同步更新：语音澄清、MCP schema、数据库、提醒排程、状态机、屏幕显示和实板回归矩阵。
