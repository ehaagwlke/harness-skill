# Harness 技能

> [**English**](./README.md) | **中文**

一个始终开启的门控执行闭环，确保任何代码、配置、内容或基础设施变更都经过捕获、计划、测试和用户明确验收后才发布。

## 它能做什么

Harness 是一个**始终开启**的结构化执行框架，强制执行纪律严明的门控闭环：

```
CAPTURE → PLAN → [CLARIFY?] → EXECUTE → [SMOKE COMMIT →] SMOKE → CONFIRM → [REDO?] → CLOSE
```

适用于任何会改变代码、配置、内容或基础设施的任务。

## 门控（Gates）

| 门控 | 名称 | 类型 | 说明 |
|------|------|------|------|
| 0 | CAPTURE | 自动 | 将请求记录到会话日志（`docs/SESSION.md`） |
| 1 | PLAN | 软门控 | 拆解为最小可独立执行的动作；在触碰任何文件前记录计划 |
| 2 | CLARIFY | 条件触发 | 范围模糊或涉及破坏性操作时，提出聚焦的问题 |
| 3 | EXECUTE | 自动 | Claude 计划并将文件编辑委托给 DeepSeek；Claude 验证输出 |
| 4 | SMOKE | 条件触发 | 需要部署时 smoke commit + 自动推送；运行 smoke 测试；UI 变更时做前端交付验证 |
| 5 | CONFIRM | **硬门控** | 展示交付物；等待用户明确接受后才继续 |
| 6 | CLOSE | 自动 | 在会话日志记录结果；更新变更日志；执行最终 VCS 操作 |

## 核心规则

- **DeepSeek 编辑，Claude 计划/审查** — Gate 3 的所有文件编辑都通过 DeepSeek 委托完成。
- **先记后做** — 计划在任何文件被触碰前写出。
- **一次一个问题** — 不问复合问题。
- **先 smoke 再 confirm** — 所有代码变更必须先通过 smoke 测试才能进入 Gate 5。
- **两种提交** — smoke commit（预验收，自动推送）vs final commit（验收后，需显式要求才推送）。
- **会话日志是真理** — 所有状态都保存在 `docs/SESSION.md`（或项目指定的位置）。

## 不可协商的设计原则

- 易用、易懂、视觉吸引人、简单直接。
- 从第一天起支持 i18n：`zh-CN`、`en-US`、`ar-AR`。
- 从第一天起支持光暗主题，默认跟随系统设置。
- 默认移动端友好。
- 任何可撤销的操作都必须有撤销路径。
- 任何超过 300 ms 的操作都要显示进度/状态。

## 安装

此技能位于 `.agents/skills/harness/`。确保以下文件存在：

```
harness/
├── SKILL.md      # 主技能说明（带 YAML frontmatter）
├── README.md     # 英文 + 中文双语说明
├── README.zh.md  # 本文件（简体中文）
├── GATES.md      # 门控规则快速参考
└── WORKFLOW.md   # 详细工作流参考
```

## 许可证

MIT
