# NanoClaw 教学大纲

> 本目录收录了 NanoClaw 项目的两周教学大纲，帮助开发者系统性地掌握这个多渠道个人 Claude 助手项目。

## 产物清单

| 版本 | 格式 | 文件 | 访问地址 |
|------|------|------|----------|
| V1 | 段落版 | `syllabus-v1.md` | [查看](https://github.com/sevenyearsonelife/nanoclaw/blob/main/linus/syllabus-v1.md) |
| V1 | 表格版 | `syllabus-v1-table.md` | [查看](https://github.com/sevenyearsonelife/nanoclaw/blob/main/linus/syllabus-v1-table.md) |
| V2 | 段落版 | `syllabus-v2.md` | [查看](https://github.com/sevenyearsonelife/nanoclaw/blob/main/linus/syllabus-v2.md) |
| V2 | 表格版 | `syllabus-v2-table.md` | [查看](https://github.com/sevenyearsonelife/nanoclaw/blob/main/linus/syllabus-v2-table.md) |

## 版本说明

- **V1**：由模型 A 生成的原始版本，内容详实，以段落形式展开
- **V2**：由模型 B 生成的重构版本，结构重新组织，内容保持一致

## 格式说明

- **段落版**：以自然段落形式呈现，适合沉浸式阅读
- **表格版**：以表格形式呈现，适合快速查阅和对比

## 项目简介

NanoClaw 是一个多渠道个人 Claude 助手，支持：

- 多渠道接入（WhatsApp / Telegram / Slack / Discord / Gmail 等）
- 持久化记忆（基于 CLAUDE.md 的分层记忆系统）
- 定时任务（Cron / Interval / Once 三种调度类型）
- 容器隔离执行（Docker / Apple Container 隔离环境）

技术栈：Host 端 Node.js 20+ / pnpm，Container 端 Bun，SQLite（better-sqlite3 / bun:sqlite），Claude Agent SDK。

## 使用建议

1. 先通读 V1 段落版建立整体认知
2. 再读 V2 段落版进行结构化的知识梳理
3. 日常查阅时使用表格版快速定位知识点
