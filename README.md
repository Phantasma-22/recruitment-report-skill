# 招聘周报生成器

**自用。** 这是一个 OpenClaw 单文件技能，用招聘汇总人数生成可离线打开的 HTML 周报。

- 将各岗位的阶段人数汇总到同一份报告，支持招聘四阶段、飞书六阶段和自定义阶段。
- 对比本周与紧邻的上一周，显示各阶段的本周人数、上周人数、增减人数和环比。
- 按 `recruitment-YYYY-MM-DD.json` 查找两周的数据，也支持手动录入和批量粘贴。多岗位报告可导出为 HTML、CSV，或使用浏览器打印保存为 PDF。

## 安装

将 [SKILL.md](SKILL.md) 放到 OpenClaw 工作区的 `skills/recruitment-report/`，在目标会话调用 `/recruitment-report`。技能文件内附数据样例、模板和生成命令，使用 Node.js 内置模块。
