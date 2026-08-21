---
name: recruit-schedule
description: 通过 qq-email MCP 批量读取最近 1 天、3 天或 1 周的邮件，识别校招/求职相关的面试、笔试、测评通知，提取时间安排并按固定格式输出总结文档，再通过 apple-calendar MCP 对比现有日历，将缺失或已变更的日程自动写入苹果日历。当用户提到"面试安排"、"笔试安排"、"测评安排"、"最近有什么考试/面试"、"总结一下求职日程"、"看看最近一天/三天/一周的邮件安排"时使用。
---

# 求职日程总结（面试/笔试/测评）

通过 `qq-email` MCP 扫描收件箱，提取招聘相关的面试、笔试、测评时间安排；所有事项持久化到 `references/recruit-data/` 的 JSON 状态文件（按天分文件，跟踪凭证、日历同步与完成状态）；日历同步以 JSON 状态为前置过滤，只处理需要同步的事项；全部处理完成后一次性输出总结文档。

## 前置条件

- 需要 `qq-email` MCP 可用（工具：`search_emails`、`read_emails`）
- MCP 不可用时提示用户先启动 email-mcp-server：
  `nohup node "$(npm root -g)/@xingyuchen/email-mcp/dist/index.js" > /tmp/email-mcp-server.log 2>&1 &`
- 日历同步需要 `apple-calendar` MCP 可用（工具：`calendar_list_events`、`calendar_list_calendars`）；不可用时仅输出总结文档，跳过日历同步并告知用户

## 工作流程

```
Step 1 确定时间范围 → Step 2 拉取邮件 → Step 3 筛选 → Step 4 去重
→ Step 5 提取字段 → Step 6 更新 JSON 状态 → Step 7 日历同步（JSON 预筛）
→ Step 8 完成情况确认 → Step 9 输出总结文档（最后一步）
```

### Step 1: 确定时间范围

根据用户请求确定统计窗口（默认最近 3 天）：

| 用户说法 | 范围 |
|---|---|
| 最近一天 / 今天 | 当天 0 点起 |
| 最近三天 | 今天往前 3 天 |
| 最近一周 / 近 7 天 | 今天往前 7 天 |

### Step 2: 拉取邮件

调用 `search_emails`，query 使用 `since:YYYY-MM-DD`（收件日期过滤），limit 100：

```
search_emails(query="since:2026-08-20", limit=100)
```

若结果达到 limit 上限，缩小时间范围分段再查。邮件日期字段为 UTC，展示时统一换算为北京时间（UTC+8）。

### Step 3: 筛选招聘相关邮件

保留主题/正文含以下信号且**面向本人邀约**的邮件：
- 面试：面试邀请、面试提醒、面试信息更新、视频面试、AI面试、预约面试
- 笔试：笔试通知、笔试邀请、在线笔试、考试邀请、技术笔试
- 测评：在线测评、人才测评、测评通知、登记表及测评

剔除：推广营销、验证码、回绝/感谢信、投递成功确认、宣讲会预告、猎头群发、简历更新提醒（除非内含测评）。

### Step 4: 去重与冲突处理（重要）

- 同一公司同一事项有多封邮件（如"邀请"→"提醒"→"信息更新"），**以时间最新的一封为准**
- 面试时间发生变更时，旧时间作废，备注"已由 HH:MM 调整为 HH:MM"
- 已过期/已结束的条目仍列出，标注 `⚠️ 已过期` 或 `✅ 已完成`，帮助用户确认是否错过

### Step 5: 提取字段

**面试** → 面试时间（日期+时刻）、面试方（公司名）、岗位、面试方式（视频/电话/现场）

**笔试** → 区分两类：
- **固定时间**：邮件给出具体开考时刻（如 `2026-08-22 19:00-21:00`）→ 记录具体时间 + 时长
- **弹性窗口**：给出时间段内任选（如 `08:00-21:00 任选 2 小时`）或"收到邀请后 N 天内完成" → 记录窗口起止/截止日期 + 作答时长

**测评** → 生效/失效时间（即截止时间）、预计用时；只说"建议 48 小时内完成"的按邮件日期推算截止日

**链接与凭证** → 邮件中含的考试/测评/面试链接、登录账号、密码/通行证、会议号一并提取，直接填入对应章节表格的「链接」「账号/密码」列（不单设章节）；日历事件的 `description` 中仍不写这些信息

### Step 6: 更新 JSON 状态文件（数据持久化）

提取出事项后（Step 5 完成）立即与 JSON 状态合并，作为后续日历同步与总结输出的数据源。写入 `~/.qoder/skills/recruit-schedule/references/recruit-data/`（运行时 skill 目录，勿写入工作区副本，该目录已 gitignore，数据仅本地保存）。

**6.1 文件组织（按天分文件）**

- 文件名：`YYYY-MM-DD.json`，一天一个文件
- 日期归属规则：
  - **固定时间事项**（面试、固定笔试）→ 按**事项发生日**（start 日期）归文件
  - **持续多天的弹性笔试/测评**（窗口任选、N 天内完成、生效~失效）→ 按**开始时间**（窗口起始日/生效日）归文件；无开始时间的按**邮件接收日**归文件
  - **完全未注明时间的事项** → 按**邮件接收日**归文件

**6.2 JSON 结构（三大类：exams 笔试 / assessments 测评 / interviews 面试）**

```json
{
  "date": "2026-08-23",
  "interviews": [
    {
      "company": "示例公司A",
      "position": "XX工程师（城市）",
      "time_type": "fixed",
      "start": "2026-08-26 14:00",
      "end": null,
      "duration_minutes": 60,
      "format": "飞书视频面试（含代码考核）",
      "link": "https://...",
      "account": null,
      "password": null,
      "meeting_id": null,
      "calendar_synced": true,
      "time_changed": false,
      "completed": null,
      "received_at": "2026-08-21",
      "notes": ""
    }
  ],
  "exams": [
    {
      "company": "示例公司B",
      "position": "XX研发工程师",
      "title": "示例公司B校招笔试-XX方向",
      "time_type": "fixed",
      "start": "2026-08-23 10:00",
      "end": "2026-08-23 11:40",
      "duration_minutes": 100,
      "link": "https://...",
      "account": "exam_account",
      "password": "******",
      "calendar_synced": true,
      "time_changed": false,
      "completed": false,
      "received_at": "2026-08-21",
      "notes": "在线考试平台"
    }
  ],
  "assessments": [
    {
      "company": "示例公司C",
      "title": "人才特质在线测评",
      "time_type": "window",
      "valid_from": "2026-08-19 19:37",
      "deadline": "2026-09-03 19:37",
      "duration_minutes": 40,
      "link": "https://...",
      "account": "user@example.com",
      "password": "通行证 12345678901234",
      "calendar_synced": true,
      "time_changed": false,
      "completed": false,
      "received_at": "2026-08-19",
      "notes": ""
    }
  ]
}
```

> 注：以上为结构示例，实际数据中的真实链接、账号、密码写入本地 JSON 文件，完整保存不截断。

字段说明：
- `time_type`：`fixed`（固定时间）/ `window`（窗口任选或生效~失效）/ `open`（未注明时间）
- `calendar_synced`：是否已在苹果日历创建事件（日历同步成功后置 `true`；未注明时间不入日历的保持 `false` 并在 notes 说明）
- `time_changed`：最新邮件时间与 JSON 记录不一致时置 `true`，日历更新成功后清除
- `completed`：`true`（已完成/已参加）/ `false`（未完成）/ `null`（未知，待确认）；已过窗口仍未完成的改标 `"expired"`
- 凭证字段（link/account/password/meeting_id）完整保存，与总结文档一致，不截断

**6.3 更新规则（每次扫描邮件后执行）**

1. 先读取目标日期已存在的 JSON 文件；不存在则新建
2. **新增事项** → 追加到对应类别数组，`calendar_synced` 初始为 `false`
3. **已有事项**（同 company + 同类型 + 同 position/title 视为同一事项）→ 以最新邮件为准合并更新时间/链接/凭证，并在 notes 记录变更（如"时间由 17:30 调整为 19:30"）；时间发生变更的置 `time_changed: true`，供 Step 7 决定是否需更新日历事件
4. 事项已过截止时间且 `completed` 仍为 `false` 的，改为 `"expired"`
5. `calendar_synced` 的回写在 Step 7 日历同步完成后执行

### Step 7: 日历同步（JSON 状态预筛 + osascript 写入）

**7.1 用 JSON 状态预筛（精简步骤，避免重复查询日历）**

逐条检查 Step 6 合并后的 JSON：

| JSON 状态 | 动作 |
|---|---|
| `calendar_synced: true` 且时间无变更 | 直接标记 `已在日历`，跳过，不查日历 |
| `calendar_synced: true` 但 `time_changed: true` | 进入更新流程（7.3） |
| `calendar_synced: false` 且有明确时间 | 进入日历查询比对（7.2） |
| 已过期 / `time_type: open`（未注明时间） | 不写入日历 |

例外处理：若用户反馈日历中找不到某事件（可能被手动删除），将该条目 `calendar_synced` 置 `false` 重新同步。

**7.2 查询日历并比对**（仅针对预筛后需要同步的事项，只处理未来未发生的）

- 调用 `calendar_list_calendars` 获取 `calendar_id`（用户主日历通常是名为「日历」的那个）
- 调用 `calendar_list_events` 拉取 **今天 ~ 待同步事项的最晚日期** 窗口内的已有事件；注意 MCP 的 `start_iso`/`end_iso` 均为 **UTC 时间**，北京时间需减 8 小时（如北京 `08-21 00:00` → `2026-08-20T16:00:00`）；单次查询不超过 8 天，必要时分段
- 按「公司名 + 事项类型」匹配事件标题：

| 比对结果 | 动作 |
|---|---|
| 日历中不存在 | 新建事件（见 7.3） |
| 已存在且时间一致 | 跳过，标记 `已在日历`，回写 `calendar_synced: true` |
| 已存在但时间不一致（以邮件最新通知为准） | 更新时间（见 7.3），在结果中标注 |
| 日历中时间与该事项冲突（占用了同一时段） | 不覆盖，仅在结果中提示用户手动处理 |

**7.3 写入规范（默认使用 AppleScript）**

`apple-calendar` MCP 的写入接口已知异常（报 `CALENDAR_NOT_FOUND`/`EVENT_NOT_FOUND`），**不要尝试 `calendar_create_event`/`calendar_update_event`**；新建与更新一律通过 Bash 执行 `osascript` 写入「日历」：

```applescript
tell application "Calendar"
  tell calendar "日历"
    set s to current date
    set year of s to 2026 -- 依次设置 month/day/hours/minutes/seconds
    set ev to make new event with properties {summary:"示例公司面试", start date:s, end date:s + (1 * hours), description:"岗位｜面试方式｜邮件发件日期"}
    tell ev -- 新建事件必须附带提醒
      make new display alarm at end of display alarms with properties {trigger interval:-1440}
    end tell
  end tell
end tell
```

- `summary`：`公司名+类型`，如「XX面试」「XX笔试」「XX测评截止」
- 时间：面试/固定笔试按邮件起止时间；只给了开始时间的按邮件注明时长或默认 1 小时
- **弹性笔试/测评（有截止日期）**：在截止时刻创建提醒事件，标题带「截止」，如 `XX笔试截止（收到后3天内完成）`；截止日无法确定具体时刻的用全天事件
- `description`：写岗位名、面试方式、原始邮件发件日期，**不写**考试链接/账号密码/会议号
- **默认提醒（新建事件必须附带 `display alarm`）**：
  - 面试类：提前 1 天（`-1440`）+ 提前 30 分钟（`-30`）
  - 截止类（笔试/测评截止）：提前 1 天（`-1440`）+ 提前 2 小时（`-120`）；`trigger interval` 以**事件开始时间**为基准，若事件开始时刻早于实际截止时刻，按差值换算分钟数（如截止 23:59、事件 23:30 开始，提前 2 小时 = `-91`）
  - 用户另有偏好时按用户要求调整
- **更新已有事件**：`every event whose summary is "…"` 定位后修改 `start date`/`end date`，同样遵循 7.4 确认策略
- 写入成功后立即回写 JSON：`calendar_synced: true`，清除 `time_changed`
- 逐条记录同步结果（新建 / 已更新 / 已存在跳过 / 需手动处理），供 Step 9 生成「五、日历同步结果」章节

**7.4 确认策略**

新建事件可直接执行；**修改已有事件**（update）前需先在对话中列出变更内容，征得用户确认后再执行。

**7.5 MCP 职责边界**

`apple-calendar` MCP 仅用于**查询**（`calendar_list_calendars`/`calendar_list_events`）；写入一律走 `osascript`。若未来 MCP 写入接口修复（不再报 NOT_FOUND），可恢复为 MCP 优先，但仍需附带提醒字段（若其 schema 不支持 alarm，则提醒部分仍用 osascript 补加）。

### Step 8: 完成情况确认

- 若存在**时间已过但 `completed` 为 `null`/`false`** 的事项，用 `AskUserQuestion` 逐条（或打包）询问用户是否已完成/已参加，将答复写回 JSON 的 `completed` 字段
- 邮件中出现完成信号（如面试满意度调查→面试已完成）可直接置 `true`，并在对话中说明依据
- 用户主动告知"某某笔试做完了"时，同样更新对应 JSON 条目

### Step 9: 按模板输出总结文档（最后一步，一次性生成）

全部处理（JSON 更新、日历同步、完成确认）完成后，按输出模板一次性生成 Markdown 文档（保存到用户工作区的 `秋招/求职日程/` 目录，文件名格式 `求职日程总结-YYYYMMDD.md`），含「五、日历同步结果」章节，并在对话中同步展示。若当天文档已存在（同一天多次运行），以最新结果覆盖。

## 输出模板

```markdown
# 求职日程总结（最近 N 天）

> 统计窗口：YYYY-MM-DD ~ YYYY-MM-DD ｜ 扫描邮件 X 封 ｜ 招聘相关 Y 封

## 一、面试安排（时间 + 面试方）

| 日期 | 时间 | 面试方 | 岗位 | 方式 | 链接 | 账号/会议号 | 状态/备注 |
|---|---|---|---|---|---|---|---|
| MM-DD 周X | HH:MM | 公司名 | 岗位名 | 视频/现场 | 完整链接 | 会议号/账号 | 已确认/待确认 |

## 二、笔试安排

### 固定时间
| 公司 | 具体时间 | 时长 | 链接 | 账号 | 状态/备注 |
|---|---|---|---|---|---|

### 弹性窗口（任选时间完成 / N 天内完成）
| 公司 | 作答时长 | 窗口/截止日期 | 链接 | 账号/密码 | 状态/备注 |
|---|---|---|---|---|---|

## 三、测评安排（具体时间 + 截止日期）

| 公司 | 测评类型 | 生效时间 | 截止时间 | 链接 | 账号/通行证 | 状态 |
|---|---|---|---|---|---|---|

## 四、已过期/已结束（防遗漏核对）

| 公司 | 事项 | 原时间 | 状态 |
|---|---|---|---|

## 五、日历同步结果

| 公司 | 事项 | 日历时间（北京） | 操作 |
|---|---|---|---|
| 公司名 | 面试/笔试/测评截止 | MM-DD HH:MM | ✅ 新建 / 🔄 已更新 / ⏭️ 已存在 / ⚠️ 冲突需手动处理 |
```

排序规则：面试按时间升序；笔试/测评按截止时间升序（最紧急的在前）。

## 注意事项

- 总结文档各表格的「链接」「账号/密码」列须完整填入考试/测评/面试链接及邮件中提到的登录账号、密码/通行证、会议号（文档保存在用户本地供本人查阅）；链接注意从邮件原文完整拷贝，不要截断或加省略号，邮件未提及的填「—」
- 日历事件不写链接/账号/密码（日历会同步到多设备），凭证只进总结文档
- 时间一律使用北京时间；邮件中写 GMT+08:00/UTC+08:00 的无需换算
- 遇到无法确定截止日期的（如"请尽快完成"），标注"未注明，建议尽快"，此类事项不写入日历
- 若用户只要某一类（如只看面试），省略其他章节
- 用户明确说"只看文档不加日历"时跳过 Step 7（但 Step 6 的 JSON 更新仍要执行，`calendar_synced` 保持 `false`）
- JSON 状态文件是跨会话的单一数据源：回答"最近有什么安排""哪些还没做"类问题时，优先读取 `references/recruit-data/` 下相关日期的文件，再结合最新邮件增量更新
