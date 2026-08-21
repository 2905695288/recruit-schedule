# recruit-schedule

Qoder Skill：求职日程总结（面试/笔试/测评）+ 日历自动同步 + JSON 状态跟踪。

## 功能

- 通过 `qq-email` MCP 批量读取最近 1 天、3 天或 1 周的邮件
- 识别校招/求职相关的面试、笔试、测评通知，提取时间与凭证（链接/账号/会议号）
- 事项持久化到按天分文件的 JSON 状态文件（`references/recruit-data/`），跟踪日历同步与完成状态，支持跨会话查询"哪些还没做"
- 日历同步以 JSON 状态预筛，跳过已同步且无变化的事项，避免重复查询
- 按固定模板输出 Markdown 总结文档
- 对比现有日历，将缺失或变更的日程写入日历（含默认提醒：面试类提前 1 天 + 30 分钟，截止类提前 1 天 + 2 小时）

## 依赖

- [Qoder](https://qoder.com) IDE
- 邮件 MCP：`qq-email`（[@xingyuchen/email-mcp](https://www.npmjs.com/package/@xingyuchen/email-mcp)）
- 日历 MCP（按系统选择）：
  - **macOS**：[apple-calendar-mcp](https://github.com/s-morgan-jeffries/apple-calendar-mcp)，仅用于查询；事件写入走系统 `osascript`（MCP 写入接口存在已知异常）
  - **Android / Windows / Linux**：见下方说明

> 💡 **Android 用户提示**：`apple-calendar` MCP 与 `osascript` 写入方案仅适用于 macOS。Android 用户可将日历 MCP 替换为 Google Calendar 类实现（如 [google-calendar-mcp](https://github.com/nspady/google-calendar-mcp) 或其他支持 Google Calendar 的 MCP）完成日历写入。替换后 SKILL.md 中 Step 7 的 JSON 预筛与查询比对逻辑可直接复用，仅需将「7.3 写入规范」中的 `osascript` 部分改为对应 MCP 的事件创建工具调用（注意保留提醒字段：面试类提前 1 天 + 30 分钟，截止类提前 1 天 + 2 小时）。

## 安装

将本目录放入 Qoder 全局 Skill 目录：

```bash
cp -r recruit-schedule ~/.qoder/skills/
```

重载 Skill 列表后，直接说「看看最近三天的面试笔试安排」「总结最近一周的招聘邮件」即可触发。

## 文件

- `SKILL.md`：Skill 主文件（9 步工作流程、JSON 状态文件规范、日历同步规范、输出模板）
- `.gitignore`：排除本地个人数据目录
- `references/recruit-data/`：按天分文件的 JSON 状态数据（含考试链接、账号密码等个人凭证），已 gitignore，仅本地保存，不随仓库分发
