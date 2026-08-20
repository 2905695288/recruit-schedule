# recruit-schedule

Qoder Skill：求职日程总结（面试/笔试/测评）+ 苹果日历自动同步。

## 功能

- 通过 `qq-email` MCP 批量读取最近 1 天、3 天或 1 周的邮件
- 识别校招/求职相关的面试、笔试、测评通知，提取时间与凭证（链接/账号/会议号）
- 按固定模板输出 Markdown 总结文档
- 通过 `apple-calendar` MCP 对比现有日历，将缺失或变更的日程写入苹果日历（含默认提醒：面试类提前 1 天 + 30 分钟，截止类提前 1 天 + 2 小时）

## 依赖

- [Qoder](https://qoder.com) IDE
- MCP：`qq-email`（[@xingyuchen/email-mcp](https://www.npmjs.com/package/@xingyuchen/email-mcp)）、`apple-calendar`（[apple-calendar-mcp](https://github.com/s-morgan-jeffries/apple-calendar-mcp)，macOS）

## 安装

将本目录放入 Qoder 全局 Skill 目录：

```bash
cp -r recruit-schedule ~/.qoder/skills/
```

重载 Skill 列表后，直接说「看看最近三天的面试笔试安排」「总结最近一周的招聘邮件」即可触发。

## 文件

- `SKILL.md`：Skill 主文件（工作流程、输出模板、日历同步规范）
