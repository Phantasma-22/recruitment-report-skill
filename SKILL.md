---
name: recruitment-report
description: 把已授权的招聘汇总数据生成按岗位分组的离线 HTML 周报、漏斗或自定义阶段报表。适用于 BOSS 直聘和飞书招聘周报。
---

# 招聘图表生成器

用汇总人数生成一个可直接在浏览器打开的 HTML。多岗位周报可导出整份 HTML、打印保存为 PDF、下载 CSV。原有单图表模式可导出 PNG。只处理报表所需的岗位、周期、阶段和人数。

## 安装到 OpenClaw

把本文件保存到目标 agent 工作区的 `skills/recruitment-report/SKILL.md`，或从包含此技能目录的位置运行 `openclaw skills install ./recruitment-report`。在目标会话使用 `/recruitment-report`。 [OpenClaw 技能安装文档](https://docs.openclaw.ai/tools/skills)

## 生成报表

1. 使用用户提供的汇总，或在用户授权的招聘数据源中取数。按岗位记录本期人数和可用的上期人数。缺失值写为 `null`，确认人数为零时写为 `0`。
2. 选择正确的阶段。在四阶段模式中，BOSS 沟通、简历上传、筛选通过、进入二面按顺序填写。同一批候选人的四阶段人数才可作为逐阶段转化依据。在飞书六阶段模式中，依序填写投递简历、安排评估、安排面试、申请 Offer、接受 Offer、新增入职。将安排评估和安排面试保留在六阶段报表内。
3. 按下方结构把数据存为授权工作区内的 JSON 文件。取数依据存在歧义时保留空白，并在交付时说明具体缺项。
4. 用下方本地命令生成新的 HTML 文件。交付该文件，让用户在浏览器中打开后查看、编辑或导出。只有用户指定了发送目的地和授权数据时，才使用对应的发送或上传工具。

### 输入 JSON

```json
{
  "title": "招聘周报",
  "scope": "某部门",
  "start": "2026-01-12",
  "end": "2026-01-18",
  "previousStart": "2026-01-05",
  "previousEnd": "2026-01-11",
  "schema": "six",
  "jobs": [
    {"name": "岗位 A", "current": [36, 14, 8, 1, 0, null], "previous": [30, 10, 5, 0, null, null]},
    {"name": "岗位 B", "current": [24, 11, 4, 0, null, null]}
  ]
}
```

`start`、`end`、`jobs` 为必填，其余字段可省略。`schema` 为 `four`、`six` 或 `custom`，默认 `four`。`custom` 需要提供 `stages`，依序填写 2–8 个互异的非空阶段名称。填写上期起止日期时应提供完整日期范围。同名岗位合并成一条记录。`current` 和 `previous` 应按阶段顺序填写非负整数或 `null`，尾部缺项会补为 `null`。

### 按固定文件名自动导入

每周将汇总写入 `recruitment-inbox/recruitment-YYYY-MM-DD.json`。文件名日期是周起始日，与 JSON 的 `start` 一致。`end` 是起始日加 6 天。每个文件记录一周，岗位只填本周人数 `current`。例如 `recruitment-2026-01-12.json` 的内容可以是：

```json
{
  "start": "2026-01-12",
  "end": "2026-01-18",
  "scope": "某部门",
  "schema": "four",
  "jobs": [{"name": "岗位 A", "current": [36, 14, 8, 1]}]
}
```

脚本读取日期最新的周文件，只使用起始日早 7 天的周文件作为上期。两周采用相同的招聘范围和阶段定义，按岗位名称合并。上期文件缺失时生成本期报表，环比留空。报告保存在 `reports/recruitment-week-YYYY-MM-DD.html`。重复运行复用相同内容，数据更新时生成带版本标记的新文件。

可以把导入命令接到现有获准的 OpenClaw 周报流程。该流程仍需先由现有数据源或人工写出每周 JSON。

### 本地执行

在授权的 workspace 运行以下命令。按实际位置填写路径。手动生成单份报表时使用尚未存在的名称。Node.js 使用内置模块，浏览器端可离线打开。

先从本文件提取内嵌导出脚本。第二个路径用于暂存脚本，首次使用在现有目录写入即可。已经提取了同版本脚本时可直接使用。

```text
node -e "const f=require('node:fs');const a=f.readFileSync(process.argv[1],'utf8').split(/\r?\n/);const b=a.indexOf('<!-- OPENCLAW_EXPORT_NODE_BEGIN -->');const e=a.indexOf('<!-- OPENCLAW_EXPORT_NODE_END -->',b+1);if(b<0||e<=b+2)throw Error('Export script section missing');f.writeFileSync(process.argv[2],a.slice(b+2,e-1).join('\n')+'\n',{encoding:'utf8',flag:'wx'});" "./skills/recruitment-report/SKILL.md" "./export-recruitment-report.cjs"
```

生成已填数据的报表。省略最后一个 JSON 路径可生成空白的手动录入工具。

```text
node "./export-recruitment-report.cjs" "./skills/recruitment-report/SKILL.md" "./recruitment-report.html" "./recruitment-input.json"
```

从固定目录导入最新周报：

```text
node "./export-recruitment-report.cjs" "./skills/recruitment-report/SKILL.md" "./reports" --latest "./recruitment-inbox"
```

如果当前会话没有本地文件执行权限，由用户在自己的工作区运行上述命令。

## 内嵌导出脚本

<!-- OPENCLAW_EXPORT_NODE_BEGIN -->
```javascript
"use strict";
const fs = require("node:fs");
const path = require("node:path");
const { createHash } = require("node:crypto");

function requireValue(condition, message) {
  if (!condition) throw new Error(message);
}
function isRecord(value) {
  return value !== null && typeof value === "object" && !Array.isArray(value);
}
function optionalText(value, key) {
  if (value === undefined) return undefined;
  requireValue(typeof value === "string", key + " 应填写文字");
  return value.trim();
}
function dateValue(value, key, optional = false) {
  if (optional && value === undefined) return undefined;
  requireValue(typeof value === "string" && /^\d{4}-\d{2}-\d{2}$/.test(value) && !value.startsWith("0000"), key + " 应使用 YYYY-MM-DD");
  const day = new Date(value + "T00:00:00Z");
  requireValue(Number.isFinite(day.getTime()) && day.toISOString().slice(0, 10) === value, key + " 日期无效");
  return value;
}
function countArray(values, key, expectedSize) {
  requireValue(Array.isArray(values) && values.length <= expectedSize, key + " 最多包含 " + expectedSize + " 项");
  requireValue(values.every(value => value === null || (Number.isSafeInteger(value) && value >= 0)),
    key + " 每项应为非负整数或 null");
  return Array.from({ length: expectedSize }, (_, i) => values[i] ?? null);
}
function normalize(input) {
  requireValue(isRecord(input), "输入应为 JSON 对象");
  const schema = input.schema === undefined ? "four" : input.schema;
  requireValue(["four", "six", "custom"].includes(schema), "schema 应为 four、six 或 custom");
  const start = dateValue(input.start, "start");
  const end = dateValue(input.end, "end");
  const previousStart = dateValue(input.previousStart, "previousStart", true);
  const previousEnd = dateValue(input.previousEnd, "previousEnd", true);
  requireValue(start <= end, "本期结束日期须等于或晚于开始日期");
  requireValue(Boolean(previousStart) === Boolean(previousEnd), "请填写完整的上期开始和结束日期");
  requireValue(!previousStart || !previousEnd || previousStart <= previousEnd, "上期结束日期须等于或晚于开始日期");

  let stages;
  if (schema === "custom") {
    requireValue(Array.isArray(input.stages) && input.stages.length >= 2 && input.stages.length <= 8, "自定义报表应包含 2–8 个阶段");
    stages = input.stages.map((label, i) => optionalText(label, "stages[" + i + "]"));
    requireValue(stages.every(Boolean) && new Set(stages).size === stages.length, "阶段名称须非空且互异");
  }
  const size = stages?.length ?? (schema === "four" ? 4 : 6);
  requireValue(Array.isArray(input.jobs) && input.jobs.length > 0, "请提供至少一项岗位数据");
  const names = new Set();
  const jobs = input.jobs.map((job, i) => {
    const key = "jobs[" + i + "]";
    requireValue(isRecord(job), key + " 应包含岗位名称和人数");
    const name = optionalText(job.name, key + ".name");
    requireValue(name, key + ".name 应填写岗位名称");
    requireValue(!names.has(name), key + ".name 岗位名称重复");
    names.add(name);
    const current = countArray(job.current, key + ".current", size);
    const previous = job.previous === undefined ? undefined : countArray(job.previous, key + ".previous", size);
    return { name, current, previous };
  });
  return {
    title: optionalText(input.title, "title"), scope: optionalText(input.scope, "scope"),
    start, end, previousStart, previousEnd, schema, stages, jobs,
  };
}

function getHtml(markdown) {
  const lines = markdown.split(/\r?\n/);
  const begin = lines.indexOf("<!-- OPENCLAW_REPORT_HTML_BEGIN -->");
  const end = lines.indexOf("<!-- OPENCLAW_REPORT_HTML_END -->", begin + 1);
  requireValue(begin >= 0 && end > begin + 2 && lines[begin + 1] === "```html" && lines[end - 1] === "```", "未找到完整的 HTML 模板");
  return lines.slice(begin + 2, end - 1).join("\n") + "\n";
}

function fillHtml(html, report) {
  const placeholder = '<script id="initial-multi-report" type="application/json">null</script>';
  requireValue(html.split(placeholder).length === 2, "HTML 数据插槽应存在且唯一");
  if (!report) return html;
  const safeJson = JSON.stringify(report).replace(/[<>&\u2028\u2029]/g, char => "\\u" + char.charCodeAt(0).toString(16).padStart(4, "0"));
  return html.replace(placeholder, () => placeholder.replace(">null<", () => ">" + safeJson + "<"));
}

function readJson(source) {
  try {
    const raw = fs.readFileSync(source, "utf8").replace(/^\uFEFF/, "");
    return JSON.parse(raw);
  } catch (error) {
    throw new Error(path.basename(source) + " 读取失败。" + error.message);
  }
}

function shiftDays(isoDate, days) {
  const date = new Date(isoDate + "T00:00:00Z");
  date.setUTCDate(date.getUTCDate() + days);
  return date.toISOString().slice(0, 10);
}

function readWeek(inbox, start) {
  const filename = "recruitment-" + start + ".json";
  const data = readJson(path.join(inbox, filename));
  try {
    requireValue(isRecord(data), "应为 JSON 对象");
    requireValue(data.start === dateValue(start, "文件名日期"), "start 应与文件名日期一致");
    requireValue(data.end === shiftDays(start, 6), "end 应为 start 加 6 天");
    const { title, scope, end, schema, stages, jobs } = data;
    const currentJobs = Array.isArray(jobs) ? jobs.map(job => isRecord(job)
      ? { name: job.name, current: job.current } : job) : jobs;
    return normalize({ title, scope, start, end, schema, stages, jobs: currentJobs });
  } catch (error) {
    throw new Error(filename + " 校验失败。" + error.message);
  }
}

function latestReport(inbox) {
  const weeks = fs.readdirSync(inbox, { withFileTypes: true }).flatMap(entry => {
    const match = /^recruitment-(\d{4}-\d{2}-\d{2})\.json$/.exec(entry.name);
    return entry.isFile() && match ? [match[1]] : [];
  }).sort();
  requireValue(weeks.length, "收件目录没有名称为 recruitment-YYYY-MM-DD.json 的周文件");
  const start = weeks.at(-1);
  const current = readWeek(inbox, start);
  const earlier = shiftDays(start, -7);
  const empty = Array(current.jobs[0].current.length).fill(null);
  if (!weeks.includes(earlier)) {
    console.log("未找到上期文件 recruitment-" + earlier + ".json，仅包含本期人数。");
    return { ...current, jobs: current.jobs.map(job => ({ ...job, previous: empty })) };
  }
  const previous = readWeek(inbox, earlier);
  requireValue(current.schema === previous.schema, "本期与上期 schema 不一致");
  requireValue(JSON.stringify(current.stages) === JSON.stringify(previous.stages), "本期与上期自定义阶段不一致");
  requireValue((current.scope ?? "") === (previous.scope ?? ""), "本期与上期招聘范围不一致");

  const priorJobs = new Map(previous.jobs.map(job => [job.name, job.current]));
  const jobs = current.jobs.map(job => {
    const counts = priorJobs.get(job.name) ?? empty;
    priorJobs.delete(job.name);
    return { ...job, previous: counts };
  });
  for (const [name, counts] of priorJobs) jobs.push({ name, current: empty, previous: counts });
  return { ...current, previousStart: previous.start, previousEnd: previous.end, jobs };
}

function writeOrCompare(destination, html) {
  try {
    fs.writeFileSync(destination, html, { encoding: "utf8", flag: "wx" });
    return "已生成";
  } catch (error) {
    if (error.code !== "EEXIST") throw error;
    if (fs.lstatSync(destination).isFile() && fs.readFileSync(destination, "utf8") === html) return "已复用";
    return null;
  }
}

function saveWeekly(outputDir, start, html) {
  fs.mkdirSync(outputDir, { recursive: true });
  const base = path.join(outputDir, "recruitment-week-" + start);
  let filename = base + ".html";
  let status = writeOrCompare(filename, html);
  if (!status) {
    const hash = createHash("sha256").update(html).digest("hex");
    filename = base + "-" + hash + ".html";
    status = writeOrCompare(filename, html);
    requireValue(status, "版本文件已存在且内容不同，请使用另一个报告目录");
  }
  console.log(status + " " + filename);
}

function main() {
  const [skillPath, outputPath, inputOrMode, inboxPath, ...extra] = process.argv.slice(2);
  const useLatest = inputOrMode === "--latest";
  requireValue(skillPath && outputPath && !extra.length && (useLatest ? inboxPath : !inboxPath),
    "用法：node 导出脚本 SKILL.md 报表.html [输入.json]，或 SKILL.md 报表目录 --latest 周文件目录");
  const template = getHtml(fs.readFileSync(skillPath, "utf8"));
  if (useLatest) {
    const report = latestReport(inboxPath);
    saveWeekly(outputPath, report.start, fillHtml(template, report));
    return;
  }
  const report = inputOrMode ? normalize(readJson(inputOrMode)) : null;
  const html = fillHtml(template, report);
  fs.writeFileSync(outputPath, html, { encoding: "utf8", flag: "wx" });
  console.log("已生成 " + outputPath);
}

try {
  main();
} catch (error) {
  console.error("生成报表失败。 " + error.message);
  process.exitCode = 1;
}
```
<!-- OPENCLAW_EXPORT_NODE_END -->

## 内嵌离线 HTML

<!-- OPENCLAW_REPORT_HTML_BEGIN -->
```html
<!doctype html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="color-scheme" content="light">
<title>招聘图表生成器</title>
<style>
  :root {
    color: #182d3a; background: #f1f4f3; font: 16px/1.55 "Microsoft YaHei", "PingFang SC", "Noto Sans CJK SC", system-ui, sans-serif;
    --ink: #182d3a; --muted: #506472; --line: #d5e0dd; --paper: #fff;
    --green: #096451; --green-dark: #074c3e; --pale: #e5f2ed; --warning: #854719;
  }
  *, *::before, *::after { box-sizing: border-box; }
  body { margin: 0; }
  button, input, select { font: inherit; }
  button { cursor: pointer; }
  button:disabled { opacity: .48; cursor: default; }
  button:focus-visible, input:focus-visible, select:focus-visible { outline: 3px solid #5ebea7; outline-offset: 2px; }
  .shell { max-width: 1260px; margin: auto; padding: 0 28px; }
  header { background: #142f2d; color: #f6fcf9; }
  .mast { display: flex; align-items: center; gap: 14px; min-height: 84px; padding-block: 16px; }
  .mark { width: 40px; height: 40px; border-radius: 11px; background: #b8e1cd; display: grid; align-content: center; gap: 3px; padding: 9px; }
  .mark i { display: block; height: 4px; background: #175449; border-radius: 3px; }
  .mark i:nth-child(2) { width: 72%; } .mark i:nth-child(3) { width: 48%; }
  h1, h2, p { margin: 0; }
  h1 { font-size: clamp(1.25rem, 3.6vw, 1.62rem); line-height: 1.2; font-weight: 750; }
  .mast small { color: #c0d9d0; font-size: .82rem; margin-left: auto; text-align: right; }
  main { padding-block: 23px 26px; }
  .modes { display: flex; flex-wrap: wrap; gap: 7px; margin-bottom: 21px; }
  .mode { background: #fff; color: var(--muted); border: 1px solid var(--line); border-radius: 8px; padding: 8px 14px; min-height: 43px; font-size: .91rem; font-weight: 650; }
  .mode[aria-pressed="true"] { color: #fff; background: var(--green); border-color: var(--green); }
  .mode:hover:not([aria-pressed="true"]) { background: var(--pale); color: var(--green-dark); }
  .layout { display: grid; grid-template-columns: minmax(330px, .86fr) minmax(0, 1.45fr); gap: 19px; align-items: start; }
  .panel { background: var(--paper); border: 1px solid var(--line); border-radius: 13px; min-width: 0; box-shadow: 0 2px 8px #20322e07; }
  .edit { padding: 20px; }
  .edit-top { display: flex; gap: 8px; align-items: baseline; justify-content: space-between; flex-wrap: wrap; }
  h2 { font-size: 1.16rem; font-weight: 750; }
  .muted { color: var(--muted); font-size: .87rem; }
  .hint { margin: 5px 0 14px; }
  .fields { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .wide { grid-column: 1 / -1; }
  label.field { display: grid; gap: 4px; min-width: 0; font-size: .875rem; font-weight: 600; }
  input:not([type="checkbox"]), select { min-height: 41px; width: 100%; min-width: 0; border: 1px solid #c6d4d2; border-radius: 7px; padding: 7px 9px; color: var(--ink); background: #fbfdfc; }
  input::placeholder { color: #6d7e88; }
  input[type="date"] { padding-inline: 6px; }
  input.is-invalid { border-color: #ad452e; background: #fff4ef; }
  .rows { margin-top: 15px; }
  .entry { display: grid; align-items: center; gap: 8px; margin-top: 7px; }
  .entry.boss { grid-template-columns: minmax(0, 1fr) 105px 43px; }
  .entry.jobrate { grid-template-columns: minmax(0, 1fr) 81px 81px 43px; }
  .entry.stages { grid-template-columns: minmax(0, 1fr) 100px; }
  .entry.periods { grid-template-columns: minmax(0, 1fr) 86px 86px; }
  .entry.head { font-size: .875rem; font-weight: 650; color: var(--muted); padding: 0 0 3px; }
  .stage-label { font-size: .91rem; overflow-wrap: anywhere; }
  .number { text-align: right; font-variant-numeric: tabular-nums; }
  .plain { background: transparent; border: 1px solid transparent; color: var(--green-dark); border-radius: 7px; padding: 6px 8px; font-size: .87rem; font-weight: 650; min-height: 40px; }
  .plain:hover:not(:disabled) { background: var(--pale); }
  .remove { font-weight: 500; font-size: .82rem; color: #54666e; padding-inline: 2px; }
  .remove:hover:not(:disabled) { background: #faede7; color: #8b3d27; }
  .check { display: flex; gap: 8px; align-items: center; margin-top: 13px; font-size: .88rem; font-weight: 600; }
  .check input { width: 19px; height: 19px; flex-shrink: 0; accent-color: var(--green); }
  .cohort-help { margin: 4px 0 0 28px; }
  .period-end { align-self: end; padding: 7px 0; min-height: 41px; }
  .chart-choice { display: flex; flex-wrap: wrap; gap: 6px; margin-bottom: 10px; }
  .chart-choice .mode { min-height: 36px; padding: 4px 9px; font-size: .85rem; }
  .edit-actions { display: flex; justify-content: space-between; gap: 5px; flex-wrap: wrap; margin-top: 12px; }
  .trend-row { padding: 10px 11px 12px; border: 1px solid var(--line); border-radius: 9px; margin-bottom: 9px; }
  .trend-top { display: grid; grid-template-columns: 1fr 43px; gap: 7px; align-items: end; margin-bottom: 8px; }
  .trend-stage-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 7px; }
  .trend-stage-grid label { font-size: .82rem; }
  .report { overflow: hidden; }
  .report-top { display: flex; justify-content: space-between; align-items: flex-start; flex-wrap: wrap; gap: 11px; padding: 18px 20px 13px; border-bottom: 1px solid var(--line); }
  .report-title { color: var(--ink); overflow-wrap: anywhere; }
  .meta { margin-top: 4px; overflow-wrap: anywhere; }
  .downloads { display: flex; flex-wrap: wrap; gap: 7px; }
  .btn { border-radius: 7px; padding: 7px 10px; color: var(--green-dark); background: #fff; border: 1px solid #a6c6bc; font-size: .875rem; font-weight: 650; min-height: 40px; }
  .btn:hover:not(:disabled) { background: var(--pale); }
  .btn.primary { color: #fff; background: var(--green); border-color: var(--green); }
  .btn.primary:hover:not(:disabled) { background: var(--green-dark); }
  .result-body { padding: 15px 20px 18px; }
  .notice { padding: 8px 10px; margin-bottom: 10px; background: #fff3e7; color: var(--warning); border-radius: 6px; font-size: .875rem; }
  .empty { padding: 26px 10px; text-align: center; color: var(--muted); min-height: 135px; display: grid; place-items: center; }
  .chart { margin: 0; }
  .chart canvas { width: 100%; display: block; }
  .chart canvas[hidden], [hidden] { display: none !important; }
  .legend { display: flex; flex-wrap: wrap; gap: 5px 12px; margin-bottom: 6px; }
  .legend button { padding: 4px 1px; background: none; border: 0; color: var(--ink); font-size: .84rem; min-height: 34px; }
  .legend button[aria-pressed="false"] { opacity: .52; text-decoration: line-through; }
  .swatch { display: inline-block; width: 9px; height: 9px; margin-right: 5px; border-radius: 2px; }
  .table-scroll { overflow-x: auto; margin-top: 12px; }
  table { width: 100%; border-collapse: collapse; font-size: .875rem; font-variant-numeric: tabular-nums; }
  th, td { text-align: right; padding: 8px 8px; border-bottom: 1px solid #e2eae7; white-space: nowrap; }
  th:first-child, td:first-child { text-align: left; white-space: normal; min-width: 80px; overflow-wrap: anywhere; }
  th { color: #284e44; background: #edf5f1; font-weight: 700; }
  tr.total { background: #e9f3ee; font-weight: 700; }
  .pos { color: #0b674a; } .neg { color: #9b3b26; }
  .table-note { margin-top: 9px; }
  .status { min-height: 1.4em; margin-top: 9px; }
  footer { margin: 15px 0 0; }
  @media (max-width: 820px) {
    .layout { grid-template-columns: 1fr; }
    .shell { padding-inline: 16px; }
    main { padding-top: 15px; }
    .modes { margin-bottom: 13px; }
  }
  @media (max-width: 450px) {
    .mast { min-height: 70px; gap: 9px; }
    .mast small { font-size: .72rem; }
    .shell { padding-inline: 11px; }
    .modes { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
    .mode { padding-inline: 4px; }
    .edit, .report-top, .result-body { padding-left: 12px; padding-right: 12px; }
    .entry.boss { grid-template-columns: minmax(0, 1fr) 75px 42px; }
    .entry.jobrate { grid-template-columns: minmax(0, 1fr) 64px 64px 40px; gap: 5px; }
    .entry.periods { grid-template-columns: minmax(0, 1fr) 65px 65px; gap: 5px; }
    .entry.stages { grid-template-columns: minmax(0, 1fr) 84px; }
    input[type="date"] { font-size: .92rem; }
    th, td { padding-inline: 5px; }
  }
</style>
<style id="multi-report-style">
main.shell { padding-block: 20px 24px; }
.multi-layout { display: grid; gap: 19px; }
.multi-entry { padding: 20px; }
.multi-entry[open] > summary { margin-bottom: 14px; }
.multi-entry > summary { color: var(--ink); cursor: pointer; font-size: 1.1rem; font-weight: 750; }
.multi-meta { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 9px 12px; }
.multi-meta label { min-width: 0; }
.multi-meta .span-two { grid-column: span 2; }
.multi-meta .span-all { grid-column: 1 / -1; }
.multi-meta [data-multi-periods] { margin-top: 2px; }
.multi-stage-list { display: flex; flex-wrap: wrap; gap: 8px; align-items: end; grid-column: 1 / -1; }
.multi-stage-list label { flex: 1 1 140px; }
.multi-stage-list .plain { min-height: 41px; }
.multi-import { border: 1px solid var(--line); border-radius: 8px; padding: 9px 11px; margin: 13px 0 5px; }
.multi-import summary { color: var(--green-dark); font-weight: 650; cursor: pointer; }
.multi-import-actions, .multi-edit-actions { display: flex; flex-wrap: wrap; align-items: center; gap: 7px; margin-top: 9px; }
.multi-import-actions label { flex: 0 1 260px; }
.multi-import-actions input[type="file"] { max-width: 260px; }
.multi-import textarea {
  font: inherit; min-height: 108px; resize: vertical; width: 100%; display: block; margin-top: 8px;
  border: 1px solid #c6d4d2; border-radius: 7px; background: #fbfdfc; padding: 8px;
}
.multi-import textarea:focus-visible { outline: 3px solid #5ebea7; outline-offset: 2px; }
.multi-import-hint { margin-top: 7px; }
.multi-grid-wrap { overflow-x: auto; padding: 3px 1px 5px; margin-top: 12px; }
.multi-grid { min-width: max-content; }
.multi-grid td, .multi-grid th { padding: 5px 4px; }
.multi-grid td:first-child { min-width: 160px; }
.multi-grid input { width: 72px; }
.multi-grid input[data-key="name"] { width: 158px; }
.multi-grid th { white-space: nowrap; }
.multi-grid th.multi-previous { background: #f0f3f5; color: #485e6b; }
.multi-roll { margin-left: auto; }
.multi-output { min-width: 0; }
.multi-report-body { padding: 16px 20px 20px; }
.multi-heading { color: var(--ink); font-size: 1.35rem; }
.multi-period { overflow-wrap: anywhere; }
.multi-summary-scroll { overflow-x: auto; }
.multi-summary { min-width: 400px; }
.multi-summary th:first-child { min-width: 115px; }
.multi-summary td small { display: block; font-size: .75rem; color: var(--muted); }
.multi-section-title { margin: 0 0 9px; font-size: .99rem; color: var(--ink); font-weight: 750; }
.multi-section + .multi-section { margin-top: 19px; }
.multi-cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(min(290px, 100%), 1fr)); gap: 10px; }
.multi-card { border: 1px solid var(--line); padding: 11px 12px; border-radius: 9px; min-width: 0; }
.multi-card h4 { margin: 0 0 8px; font-size: .99rem; color: var(--ink); overflow-wrap: anywhere; }
.multi-card-stage { display: grid; grid-template-columns: minmax(0, 1fr) auto; column-gap: 5px; align-items: center; margin: 7px 0; font-size: .875rem; }
.multi-card-stage .numbers { text-align: right; white-space: nowrap; font-weight: 700; font-variant-numeric: tabular-nums; }
.multi-card-stage .numbers small { margin-left: 5px; color: var(--muted); font-size: .78rem; font-weight: 500; }
.multi-stage-name { overflow-wrap: anywhere; }
.multi-bar-track { grid-column: 1 / -1; background: #edf3f0; border-radius: 2px; height: 10px; margin-top: 3px; }
.multi-bar { display: block; height: 100%; margin: auto; clip-path: polygon(0 0, 100% 0, 96% 100%, 4% 100%); }
.multi-comparison { grid-column: 1 / -1; font-size: .78rem; color: var(--muted); margin-top: 2px; }
.multi-comparison.pos { color: #0b674a; }
.multi-comparison.neg { color: #9b3b26; }
.multi-summary .pos { color: #0b674a; }
.multi-summary .neg { color: #9b3b26; }
.multi-report-note { margin-top: 13px; font-size: .82rem; color: var(--muted); }
.multi-report-body .notice { overflow-wrap: anywhere; }
.multi-message { margin-top: 8px; }
.multi-message .btn { margin: 5px 0 0 6px; }
.multi-edit-actions button { max-width: 100%; }
@media (max-width: 630px) {
  main.shell { padding-block: 12px 17px; }
  .multi-entry, .multi-report-body { padding-inline: 12px; }
  .multi-meta { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .multi-meta .span-two { grid-column: span 2; }
  .multi-roll { margin-left: 0; }
  .multi-summary { min-width: 0; }
  .multi-summary th:first-child { min-width: 82px; }
  .multi-summary th, .multi-summary td { white-space: normal; overflow-wrap: anywhere; padding-inline: 4px; }
}
@media print {
  @page { margin: 13mm; size: A4; }
  body.print-multi { background: #fff; }
  body.print-multi > header, body.print-multi #modes, body.print-multi #storageNote,
  body.print-multi .multi-entry, body.print-multi #multiExports, body.print-multi .multi-message { display: none !important; }
  body.print-multi main.shell { max-width: none; margin: 0; padding: 0; }
  body.print-multi .multi-output { border: 0; box-shadow: none; }
  body.print-multi .multi-report-body, body.print-multi .report-top { padding-inline: 0; }
}
@media print {
  .multi-output { overflow: visible; }
  .multi-cards { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .multi-card, .multi-summary tr { break-inside: avoid; }
  .multi-section-title { break-after: avoid; }
  .multi-bar, .multi-bar-track, .multi-summary th { print-color-adjust: exact; -webkit-print-color-adjust: exact; }
}

</style>
</head>
<body>
<header>
  <div class="shell mast">
    <div class="mark" aria-hidden="true"><i></i><i></i><i></i></div>
    <h1>招聘图表生成器</h1>
    <small>离线填写与导出</small>
  </div>
</header>
<main class="shell">
  <nav class="modes" id="modes" aria-label="图表模式"></nav>
  <div class="layout" id="singleLayout" hidden>
    <section class="panel edit" aria-labelledby="entryTitle">
      <div class="edit-top">
        <h2 id="entryTitle">填写数据</h2>
        <button type="button" class="plain" id="loadExample"></button>
      </div>
      <p class="muted hint" id="entryHint"></p>
      <div id="editor"></div>
      <div class="edit-actions" id="editActions"></div>
      <p class="muted status" id="editStatus" role="status"></p>
    </section>
    <section class="panel report" aria-labelledby="reportTitle">
      <div class="report-top">
        <div>
          <h2 class="report-title" id="reportTitle"></h2>
          <p class="muted meta" id="reportMeta"></p>
        </div>
        <div class="downloads">
          <button type="button" class="btn" id="csv">下载 CSV</button>
          <button type="button" class="btn primary" id="png">下载 PNG</button>
        </div>
      </div>
      <div class="result-body">
        <p class="notice" id="notice" hidden></p>
        <div class="chart-choice" id="chartChoice" aria-label="四阶段图表" hidden></div>
        <div class="legend" id="legend" aria-label="趋势系列" hidden></div>
        <p class="empty" id="empty" hidden></p>
        <figure class="chart" id="chartWrap">
          <canvas id="chart" role="img" aria-label="招聘数据图表。具体数值见下方表格。"></canvas>
        </figure>
        <div class="table-scroll" id="tableWrap" tabindex="0" aria-label="统计表格"></div>
        <p class="muted table-note" id="tableNote"></p>
        <p class="muted status" id="downloadStatus" role="status" hidden></p>
      </div>
    </section>
  </div>
  <div id="multiLayout" class="multi-layout" hidden></div>
  <footer class="muted" id="storageNote">数据保留在本机浏览器。下载的 CSV 可用 Excel 打开。</footer>
</main>
<script id="report-model">
"use strict";
const FOUR = ["BOSS 沟通", "简历上传系统", "筛选通过", "进入二面"];
const SIX = ["投递简历", "安排评估", "安排面试", "申请 Offer", "接受 Offer", "新增入职"];
const STEPS = ["沟通至上传", "上传至筛选通过", "筛选通过至二面"];
const PERIODS = { week: "周报", biweek: "双周报", month: "月报", custom: "自定义" };
const COLORS = ["#12745f", "#337eaa", "#a86122", "#7e5bb1", "#bb4d6b", "#427a37"];
const INCLUDE_WORK_EXAMPLES = false;
const WORK_EXAMPLES_UI = { boss: "载入模拟数据", six: "载入模拟数据" };
const bossRows = () => ["岗位 A", "岗位 B", "岗位 C"].map(label => ({ label, current: "" }));
function exampleDraft(mode) {
  const draft = blankDraft(mode);
  draft.scope = "模拟数据";
  if (mode === "boss") {
    draft.rows.forEach((row, i) => { row.current = ["26", "18", "12"][i]; });
  } else if (mode === "four") {
    draft.current = ["100", "35", "18", "7"];
  } else if (mode === "six") {
    draft.current = ["90", "45", "22", "6", "4", "2"];
    draft.prior = ["80", "41", "18", "5", "2", "0"];
  } else if (mode === "jobrate") {
    draft.rows = [["岗位 A", "26", "3"], ["岗位 B", "18", "1"], ["岗位 C", "12", "0"]]
      .map(([label, current, second]) => ({ label, current, second }));
  } else if (isTrend(mode)) {
    const numbers = mode === "eff" ? [[80, 34, 18, 7], [95, 45, 24, 12]]
      : [[80, 41, 18, 5, 2, 0], [90, 45, 22, 6, 4, 2]];
    draft.rows = numbers.map((row, i) => trendRow(mode, "第 " + (i + 1) + " 周", row.map(String)));
  }
  return draft;
}
const MODES = {
  multi: { name: "多岗位周报" },
  boss: {
    name: "岗位周报", title: "BOSS 直聘岗位周报", sample: WORK_EXAMPLES_UI.boss,
    hint: "每个岗位填一行，右侧即时汇总人数和占比。",
  },
  four: {
    name: "四阶段漏斗", title: "四阶段招聘漏斗", sample: "载入模拟数据",
    hint: "填写 BOSS 沟通、简历上传、筛选通过和进入二面的人数。",
  },
  six: {
    name: "飞书六阶段", title: "飞书招聘六阶段", sample: WORK_EXAMPLES_UI.six,
    hint: "填写飞书的六项人数，上期人数或页面显示的环比可选填。",
  },
  jobrate: {
    name: "岗位转化", title: "各岗位整体招聘转化率", sample: "载入模拟数据",
    hint: "每个岗位分别填写沟通人数和进入二面人数。",
  },
  eff: {
    name: "转化率趋势", title: "招聘效率周度趋势", sample: "载入模拟数据",
    hint: "每周填入同批候选人的四个阶段人数，查看各步转化率。",
  },
  trend: {
    name: "飞书周趋势", title: "飞书招聘多周趋势", sample: "载入模拟数据",
    hint: "按时间顺序填写各周期。每项都可留空，图表会保留缺口。",
  },
};
const isTrend = mode => mode === "trend" || mode === "eff";
const trendInputs = mode => mode === "trend" ? SIX : FOUR;
const trendSeries = mode => mode === "trend" ? SIX : STEPS;
const trendRow = (mode, label = "", values = trendInputs(mode).map(() => "")) => ({ label, values });
function blankDraft(mode) {
  const meta = { start: "", end: "", month: "", periodType: "week", scope: "" };
  switch (mode) {
    case "multi": return MultiUI.blank();
    case "boss": return { ...meta, rows: bossRows() };
    case "four": return { ...meta, cohort: true, chartMode: "funnel", current: FOUR.map(() => "") };
    case "six": return { ...meta, comparison: "count", current: SIX.map(() => ""), prior: SIX.map(() => ""), changes: SIX.map(() => "") };
    case "jobrate": return { ...meta, cohort: true, rows: bossRows().map(r => ({ ...r, second: "" })) };
    case "trend":
    case "eff": return {
      ...meta, cohort: true, rows: [trendRow(mode), trendRow(mode)], enabled: trendSeries(mode).map(() => true),
    };
  }
}
function countValue(raw) {
  const s = String(raw ?? "").trim();
  if (!s) return null;
  if (!/^\d+$/.test(s)) return NaN;
  const value = Number(s);
  return Number.isSafeInteger(value) ? value : NaN;
}
function changeValue(raw) {
  const text = String(raw ?? "").trim().replace(/−/g, "-").replace(/%\s*$/, "").trim();
  if (!String(raw ?? "").trim()) return null;
  if (!/^[+-]?(?:\d+(?:\.\d+)?|\.\d+)$/.test(text)) return NaN;
  const value = Number(text);
  return Number.isFinite(value) && value >= -100 ? value / 100 : NaN;
}
const validCount = x => Number.isSafeInteger(x) && x >= 0;
const ratio = (a, b) => validCount(a) && validCount(b) && b > 0 ? a / b : null;
const deltaRate = (a, b) => ratio(a, b) === null ? null : (a - b) / b;
const countFormat = new Intl.NumberFormat("zh-CN");
const percentFormat = new Intl.NumberFormat("zh-CN", { style: "percent", maximumFractionDigits: 1, minimumFractionDigits: 1 });
const displayCount = v => validCount(v) ? countFormat.format(v) : "—";
const displayRate = v => typeof v === "number" ? percentFormat.format(v) : "—";
const displayDelta = v => (v > 0 ? "+" : "") + displayRate(v);
const csvCount = v => validCount(v) ? v : "";
const csvRate = (v, signed = false) => typeof v === "number" ? (signed && v > 0 ? "+" : "") + (v * 100).toFixed(1) + "%" : "";
function stageConflict(values) {
  const entered = values.filter(validCount);
  return entered.some((v, i) => i > 0 && v > entered[i - 1]);
}
function reportData(mode, draft) {
  const notes = [];
  const data = { notes, table: [], hasData: false, chartReady: false };
  const allCounts = mode === "boss" ? draft.rows.map(r => r.current)
    : mode === "jobrate" ? draft.rows.flatMap(r => [r.current, r.second])
    : isTrend(mode) ? draft.rows.flatMap(r => r.values)
    : mode === "six" && draft.comparison === "count" ? [...draft.current, ...draft.prior] : draft.current;
  const hasInvalidCount = allCounts.some(x => Number.isNaN(countValue(x)));
  const hasInvalidChange = mode === "six" && draft.comparison === "rate" && draft.changes.some(x => Number.isNaN(changeValue(x)));
  const dates = reportDates(draft);
  const badDate = !isTrend(mode) && Boolean(dates.start && dates.end && dates.start > dates.end);
  data.blocked = hasInvalidCount || hasInvalidChange || badDate;
  if (hasInvalidCount) notes.push("人数请填写非负整数，空白表示缺数。");
  if (hasInvalidChange) notes.push("环比请填写百分数数值，最低为 -100%。");
  if (badDate) notes.push("结束日期需要等于或晚于开始日期。");

  if (mode === "boss") {
    data.items = draft.rows.map(r => ({ label: r.label.trim(), current: countValue(r.current) }));
    if (data.items.some(r => validCount(r.current) && !r.label)) {
      notes.push("请为已填写人数的岗位补上名称。");
      data.blocked = true;
    }
    data.items = data.items.filter(r => r.label);
    data.hasData = data.items.some(r => validCount(r.current));
    const missing = data.items.some(r => !validCount(r.current));
    const sum = data.items.reduce((total, r) => total + (validCount(r.current) ? r.current : 0), 0);
    const total = data.hasData && Number.isSafeInteger(sum) ? sum : null;
    if (data.hasData && total === null) {
      data.blocked = true;
      notes.push("岗位合计超出了可计算的整数范围。");
    }
    const totalLabel = missing ? "已填岗位合计" : "岗位合计";
    data.items.forEach(r => { r.share = ratio(r.current, total); });
    data.csvHeaders = data.headers = ["岗位", "BOSS 沟通人数", missing ? "占已填合计" : "占合计"];
    data.table = data.items.map(r => [r.label, displayCount(r.current), displayRate(r.share)]);
    data.csvRows = data.items.map(r => [r.label, csvCount(r.current), csvRate(r.share)]);
    if (data.hasData) {
      const totalShare = total > 0 ? 1 : null;
      data.table.push([totalLabel, displayCount(total), displayRate(totalShare)]);
      data.csvRows.push([totalLabel, csvCount(total), csvRate(totalShare)]);
      data.totalRow = true;
    }
    data.metricText = data.hasData ? totalLabel + " " + displayCount(total) + " 人" : "";
    data.note = "合计为各岗位人数相加。";
    data.chartReady = data.hasData;
    data.empty = "填写任一岗位的沟通人数，即可生成图表。";
  } else if (mode === "four") {
    const values = draft.current.map(countValue);
    const entered = values.filter(validCount);
    const conflict = stageConflict(values);
    if (draft.cohort && conflict) notes.push("后续阶段人数高于前一已填阶段，请核对人数。转化率暂缺。");
    const canConvert = draft.cohort && !conflict;
    data.items = FOUR.map((label, i) => ({
      label, current: values[i], rate: canConvert && i > 0 ? ratio(values[i], values[i - 1]) : null,
      lost: canConvert && i > 0 && validCount(values[i]) && validCount(values[i - 1]) ? values[i - 1] - values[i] : null,
    }));
    const overall = canConvert && values.every(validCount) ? ratio(values[3], values[0]) : null;
    const firstDrop = data.items[1].lost;
    data.csvHeaders = data.headers = ["阶段", "人数", "较上阶段转化", "较上阶段减少"];
    data.table = data.items.map(r => [r.label, displayCount(r.current), displayRate(r.rate), displayCount(r.lost)]);
    data.csvRows = data.items.map(r => [r.label, csvCount(r.current), csvRate(r.rate), csvCount(r.lost)]);
    data.hasData = entered.length > 0;
    if (overall !== null) {
      data.metricText = "沟通到二面 " + displayRate(overall) + "    沟通到上传减少 " + displayCount(firstDrop) + " 人";
      data.csvSummary = [["沟通到二面转化率", csvRate(overall)]];
    }
    if (data.hasData && entered.length < FOUR.length) notes.push("待填写 " + FOUR.filter((_, i) => !validCount(values[i])).join("、") + "。");
    data.lossChart = draft.chartMode === "loss";
    data.lossItems = data.items.slice(1).map((r, i) => ({ label: STEPS[i], current: r.lost }));
    data.chartReady = data.lossChart ? data.lossItems.some(r => validCount(r.current)) : data.hasData;
    data.empty = data.lossChart ? "打开「计算流失转化率」并填写相邻阶段人数，即可查看流失。" : "填写任一阶段人数，即可生成漏斗。";
  } else if (mode === "six") {
    const manual = draft.comparison === "rate";
    data.items = SIX.map((label, i) => {
      const current = countValue(draft.current[i]);
      const prior = manual ? null : countValue(draft.prior[i]);
      const entered = manual ? changeValue(draft.changes[i]) : deltaRate(current, prior);
      return { label, current, prior, rate: Number.isFinite(entered) ? entered : null };
    });
    data.hasData = data.items.some(r => validCount(r.current) || validCount(r.prior));
    data.csvHeaders = data.headers = ["阶段", "本期人数", "上期人数", "较上期"];
    data.table = data.items.map(r => [r.label, displayCount(r.current), displayCount(r.prior), displayDelta(r.rate)]);
    data.csvRows = data.items.map(r => [r.label, csvCount(r.current), csvCount(r.prior), csvRate(r.rate)]);
    data.chartReady = data.items.some(r => validCount(r.current));
    data.note = manual ? "各阶段按周期计数。环比取自填入的报表数据。" : "各阶段按周期计数。上期为 0 或缺数时，环比显示为「—」。";
    data.empty = "填写任一本期人数，即可生成飞书招聘漏斗。";
  } else if (mode === "jobrate") {
    const all = draft.rows.map(r => ({ label: r.label.trim(), current: countValue(r.current), second: countValue(r.second) }));
    data.items = all.filter(r => r.label);
    if (all.some(r => !r.label && (validCount(r.current) || validCount(r.second)))) {
      notes.push("请为已填写人数的岗位补上名称。");
      data.blocked = true;
    }
    data.items.forEach(r => {
      r.conflict = stageConflict([r.current, r.second]);
      r.rate = draft.cohort && !r.conflict ? ratio(r.second, r.current) : null;
    });
    const conflicts = data.items.filter(r => r.conflict).map(r => r.label);
    if (draft.cohort && conflicts.length) notes.push("请核对二面人数高于沟通人数的岗位 " + conflicts.join("、") + "。");
    data.headers = data.csvHeaders = ["岗位", "BOSS 沟通", "进入二面", "整体转化率"];
    data.table = data.items.map(r => [r.label, displayCount(r.current), displayCount(r.second), displayRate(r.rate)]);
    data.csvRows = data.items.map(r => [r.label, csvCount(r.current), csvCount(r.second), csvRate(r.rate)]);
    data.hasData = data.items.some(r => validCount(r.current) || validCount(r.second));
    data.chartReady = data.items.some(r => r.rate !== null);
    data.note = "整体转化率为同批候选人的进入二面人数除以 BOSS 沟通人数。";
    data.empty = "填写岗位人数并打开「计算流失转化率」，即可生成转化率对比。";
  } else {
    const periods = draft.rows.map(row => ({ label: row.label.trim(), values: row.values.map(countValue) }));
    if (periods.some(row => !row.label && row.values.some(validCount))) {
      notes.push("请为已填写人数的周期补上名称。");
      data.blocked = true;
    }
    data.periods = periods.filter(row => row.label || row.values.some(validCount));
    data.enabled = draft.enabled;
    data.hasData = data.periods.some(row => row.values.some(validCount));
    data.series = trendSeries(mode);
    data.asRate = mode === "eff";
    if (data.asRate) {
      const conflicts = data.periods.filter(row => stageConflict(row.values)).map(row => row.label || "未命名周期");
      if (draft.cohort && conflicts.length) notes.push("请核对人数逐阶段增加的周期 " + conflicts.join("、") + "。");
      const ratesFor = row => STEPS.map((_, i) => draft.cohort && !stageConflict(row.values)
        ? ratio(row.values[i + 1], row.values[i]) : null);
      data.headers = ["周期", ...STEPS];
      data.csvHeaders = ["周期", ...FOUR, ...STEPS];
      data.table = data.periods.map(row => [row.label || "—", ...ratesFor(row).map(displayRate)]);
      data.csvRows = data.periods.map(row => [row.label, ...row.values.map(csvCount), ...ratesFor(row).map(v => csvRate(v))]);
      data.periods = data.periods.map(row => ({ label: row.label, values: ratesFor(row) }));
      data.note = "各步转化率按同批候选人计算，缺数处保留缺口。";
      data.empty = "至少填写两个周期，并打开「计算流失转化率」，即可查看趋势。";
    } else {
      data.csvHeaders = data.headers = ["周期", ...SIX];
      data.table = data.periods.map(row => [row.label || "—", ...row.values.map(displayCount)]);
      data.csvRows = data.periods.map(row => [row.label, ...row.values.map(csvCount)]);
      data.note = "周期按填写顺序排列。点击图例可以显示或隐藏对应阶段。";
      data.empty = "填写至少两个周期名称，并输入阶段人数，即可生成趋势图。";
    }
    data.chartReady = data.periods.length >= 2 && data.periods.some(row => row.values.some((v, i) =>
      data.enabled[i] && (data.asRate ? v !== null : validCount(v))));
  }
  return data;
}
function reportDates(draft) {
  if (draft.periodType === "custom") return { start: draft.start, end: draft.end };
  if (draft.periodType === "month") {
    if (!/^\d{4}-(0[1-9]|1[0-2])$/.test(draft.month)) return { start: "", end: "" };
    const start = draft.month + "-01";
    const last = new Date(start + "T00:00:00Z");
    last.setUTCMonth(last.getUTCMonth() + 1, 0);
    return { start, end: last.toISOString().slice(0, 10) };
  }
  if (!draft.start) return { start: "", end: "" };
  const end = new Date(draft.start + "T00:00:00Z");
  if (!Number.isFinite(end.valueOf())) return { start: "", end: "" };
  end.setUTCDate(end.getUTCDate() + (draft.periodType === "biweek" ? 13 : 6));
  return { start: draft.start, end: end.toISOString().slice(0, 10) };
}
function dateText({ start, end }) {
  if (start && end) return start + " 至 " + end;
  if (start) return "自 " + start;
  if (end) return "截至 " + end;
  return "";
}
function csvLine(row) {
  return row.map((item, i) => {
    const text = String(item ?? "");
    const safe = i === 0 && /^[\s\uFEFF]*[=+\-@]/.test(text) ? "'" + text : text;
    return '"' + safe.replace(/"/g, '""') + '"';
  }).join(",");
}
</script>
<script id="initial-multi-report" type="application/json">null</script>
<script id="multi-report-model">
globalThis.MULTI = (() => {
  "use strict";

  const DAY = 86_400_000;
  const FOUR_LABELS = ["BOSS 沟通", "简历上传系统", "筛选通过", "进入二面"];
  const SIX_LABELS = ["投递简历", "安排评估", "安排面试", "申请 Offer", "接受 Offer", "新增入职"];

  function stageLabels(report) {
    if (report.schema === "custom") return report.stages.slice();
    if (report.schema === "six") return (typeof SIX === "undefined" ? SIX_LABELS : SIX).slice();
    return (typeof FOUR === "undefined" ? FOUR_LABELS : FOUR).slice();
  }

  function textValue(value, label, required = false) {
    if (value === undefined && !required) return "";
    if (typeof value !== "string") throw new Error(label + "需填写文字。");
    const clean = value.trim();
    if (required && !clean) throw new Error("请填写" + label + "。");
    return clean;
  }

  function isoDay(raw, label, required) {
    const date = textValue(raw, label, required);
    if (!date) return "";
    const timestamp = Date.parse(date + "T00:00:00Z");
    if (!/^\d{4}-\d{2}-\d{2}$/.test(date) || date.startsWith("0000") ||
        !Number.isFinite(timestamp) || new Date(timestamp).toISOString().slice(0, 10) !== date) {
      throw new Error(label + "需使用有效的 YYYY-MM-DD 日期。");
    }
    return date;
  }

  function dateRange(startRaw, endRaw, label, required, allowPartial = false) {
    const start = isoDay(startRaw, label + "开始日期", required);
    const end = isoDay(endRaw, label + "结束日期", required);
    if (!allowPartial && Boolean(start) !== Boolean(end)) throw new Error("请填写完整的" + label + "。");
    if (start && end && end < start) throw new Error(label + "的结束日期需等于或晚于开始日期。");
    return { start, end };
  }

  function previousPeriod(startRaw, endRaw) {
    const { start, end } = dateRange(startRaw, endRaw, "本期", true);
    const first = Date.parse(start + "T00:00:00Z");
    const last = Date.parse(end + "T00:00:00Z");
    const previousStart = new Date(first - (last - first + DAY)).toISOString().slice(0, 10);
    const previousEnd = new Date(first - DAY).toISOString().slice(0, 10);
    return {
      start: isoDay(previousStart, "上期开始日期", true),
      end: isoDay(previousEnd, "上期结束日期", true),
    };
  }

  function count(raw, label) {
    if (raw === undefined || raw === null || (typeof raw === "string" && !raw.trim())) return null;
    const isNumericText = typeof raw === "string" && /^\d+$/.test(raw.trim());
    const value = isNumericText ? Number(raw) : raw;
    if (!Number.isSafeInteger(value) || value < 0) throw new Error(label + "需填写非负整数或留空。");
    return value;
  }

  function columns(source, width, label, required = false) {
    if (source === undefined && !required) return Array(width).fill(null);
    if (!Array.isArray(source) || source.length > width) {
      throw new Error(label + "需为至多 " + width + " 项的数组。");
    }
    return Array.from({ length: width }, (_, i) => count(source[i], label + "第 " + (i + 1) + " 项"));
  }

  function normalize(input, strictDates) {
    if (!input || typeof input !== "object" || Array.isArray(input)) throw new Error("报告需使用 JSON 对象。");
    const schema = input.schema === undefined ? "four" : input.schema;
    if (!["four", "six", "custom"].includes(schema)) throw new Error("请选择四阶段、飞书六阶段或自定义阶段。");
    const { start, end } = dateRange(input.start, input.end, "本期", strictDates, !strictDates);
    const previous = dateRange(input.previousStart, input.previousEnd, "上期", false, !strictDates);
    const report = {
      title: textValue(input.title, "报告名称"),
      scope: textValue(input.scope, "招聘范围"),
      start,
      end,
      previousStart: previous.start,
      previousEnd: previous.end,
      schema,
    };
    if (schema === "custom") {
      if (!Array.isArray(input.stages) || input.stages.length < 2 || input.stages.length > 8) {
        throw new Error("自定义报告需填写 2 至 8 个阶段。");
      }
      report.stages = Array.from(input.stages, (name, i) => textValue(name, "第 " + (i + 1) + " 个阶段名称", true));
      if (new Set(report.stages).size !== report.stages.length) throw new Error("请为每个阶段使用不同的名称。");
    }
    if (!Array.isArray(input.jobs) || (strictDates && input.jobs.length === 0)) {
      throw new Error("请至少提供一个岗位。");
    }
    const width = stageLabels(report).length;
    const names = new Set();
    report.jobs = Array.from(input.jobs, (row, i) => {
      const label = "第 " + (i + 1) + " 个岗位";
      if (!row || typeof row !== "object" || Array.isArray(row)) throw new Error(label + "需使用 JSON 对象。");
      const name = textValue(row.name, label + "名称", true);
      if (names.has(name)) throw new Error("岗位名称重复。请合并「" + name + "」的同周期数据。");
      names.add(name);
      return {
        name,
        current: columns(row.current, width, name + "本期人数", true),
        previous: columns(row.previous, width, name + "上期人数"),
      };
    });
    return report;
  }

  function parseImport(input) {
    if (typeof input === "string") {
      try {
        input = JSON.parse(input.trim());
      } catch {
        throw new Error("JSON 格式无效，请检查括号、引号和逗号。");
      }
    }
    return normalize(input, true);
  }

  function compare(current, previous, allowed = true) {
    const delta = allowed && current !== null && previous !== null ? current - previous : null;
    return { current, previous, delta, rate: delta !== null && previous > 0 ? delta / previous : null };
  }

  function sum(values) {
    if (!values.length) return null;
    const total = values.reduce((acc, n) => acc + n, 0);
    return Number.isSafeInteger(total) ? total : null;
  }

  function calculate(input) {
    const report = normalize(input, false);
    const labels = stageLabels(report);
    const jobs = report.jobs.map(row => ({
      name: row.name,
      stages: row.current.map((current, i) => compare(current, row.previous[i])),
    }));
    const mismatches = [];
    const overflows = [];
    const totals = labels.map((label, i) => {
      const rows = jobs.map(job => job.stages[i]);
      const currentValues = rows.map(row => row.current).filter(n => n !== null);
      const previousValues = rows.map(row => row.previous).filter(n => n !== null);
      const currentCoverage = currentValues.length;
      const previousCoverage = previousValues.length;
      const pairedCoverage = rows.filter(row => row.current !== null && row.previous !== null).length;
      const comparisonScopeMatches = pairedCoverage > 0 && pairedCoverage === currentCoverage && pairedCoverage === previousCoverage;
      const current = sum(currentValues);
      const previous = sum(previousValues);
      if (!comparisonScopeMatches && currentCoverage && previousCoverage) mismatches.push(label);
      if ((currentCoverage && current === null) || (previousCoverage && previous === null)) overflows.push(label);
      return {
        ...compare(current, previous, comparisonScopeMatches),
        currentCoverage,
        previousCoverage,
        pairedCoverage,
        comparisonScopeMatches,
      };
    });
    const warnings = [];
    if (mismatches.length) warnings.push("本期和上期有数据的岗位不同。以下阶段的合计变化留空：" + mismatches.join("、") + "。");
    if (overflows.length) warnings.push("以下阶段合计超出可计算的整数范围：" + overflows.join("、") + "。");
    return { report, jobs, totals, warnings };
  }

  function parseTSV(input) {
    if (typeof input !== "string") throw new Error("请粘贴表格文本。");
    const text = input.replace(/^\uFEFF/, "").replace(/\r\n?/g, "\n");
    const rows = [];
    let row = [];
    let value = "";
    let quoted = false;
    for (let i = 0; i < text.length; i++) {
      const char = text[i];
      if (char === '"' && quoted) {
        if (text[i + 1] === '"') {
          value += '"';
          i++;
        } else quoted = false;
      } else if (char === '"' && !value && (i === 0 || /[\t\n]/.test(text[i - 1]))) {
        quoted = true;
      } else if (!quoted && (char === "\t" || char === "\n")) {
        row.push(value);
        value = "";
        if (char === "\n") {
          if (row.some(cell => cell.trim())) rows.push(row);
          row = [];
        }
      } else {
        value += char;
      }
    }
    if (quoted) throw new Error("粘贴的文本有未闭合的引号，请检查表格。");
    row.push(value);
    if (row.some(cell => cell.trim())) rows.push(row);
    return rows;
  }

  return Object.freeze({ parseImport, calculate, previousPeriod, stageLabels, parseTSV });
})();

</script>
<script id="multi-report-ui">
const MultiUI = (() => {
  "use strict";
  const customStages = ["平台简历", "简历过筛", "业务筛选通过", "业务一面", "业务二面"];
  const colors = ["#12745f", "#337eaa", "#a86122", "#7e5bb1", "#bb4d6b", "#427a37", "#467084", "#756b43"];
  const day = 86_400_000;
  let draft, onSave, content, editor, notice, buttons, message;
  let data, dateInfo, batchOpen = false;

  function blank() {
    return { title: "", scope: "", start: "", end: "", previousStart: "", previousEnd: "",
      schema: "four", stages: customStages.slice(), jobs: [{ name: "", current: [], previous: [] }] };
  }
  function fromReport(report) {
    return { ...blank(), ...report };
  }
  function addDays(date, count) {
    const time = Date.parse(date + "T00:00:00Z");
    return Number.isFinite(time) ? new Date(time + count * day).toISOString().slice(0, 10) : "";
  }
  function duration(start, end) {
    return Math.round((Date.parse(end + "T00:00:00Z") - Date.parse(start + "T00:00:00Z")) / day) + 1;
  }
  function resolved() {
    const current = { start: draft.start, end: draft.end || addDays(draft.start, 6) };
    let previous = { start: "", end: "" };
    if (current.start && current.end && current.end >= current.start) {
      previous = MULTI.previousPeriod(current.start, current.end);
    }
    const previousStart = draft.previousStart || previous.start;
    const previousEnd = draft.previousEnd || (draft.previousStart && current.start && current.end
      ? addDays(previousStart, duration(current.start, current.end) - 1) : previous.end);
    const ready = Boolean(current.start && current.end && previousStart && previousEnd);
    const adjacent = ready && previousEnd === addDays(current.start, -1) &&
      duration(current.start, current.end) === duration(previousStart, previousEnd);
    return {
      input: { ...draft, ...current, previousStart, previousEnd },
      ready, adjacent, weekly: adjacent && duration(current.start, current.end) === 7,
    };
  }
  function seededExample() {
    return {
      ...blank(), title: "招聘周报 · 演示", scope: "模拟数据",
      start: "2026-11-20",
      jobs: [
        { name: "岗位 A", current: [26, 13, 8, 3], previous: [20, 10, 5, 0] },
        { name: "岗位 B", current: [18, 9, 4, 1], previous: [24, 11, 6, 2] },
        { name: "岗位 C", current: [12, 6, 2, 0], previous: [null, 4, 2, 0] },
      ],
    };
  }
  function anyValue(row) {
    return [...row.current, ...(row.previous || [])].some(value => value !== null && value !== undefined && String(value).trim());
  }
  function draftJobs() {
    return draft.jobs.filter(row => row.name.trim() || anyValue(row));
  }
  function hasInput() {
    return Boolean(draftJobs().length || draft.start || draft.end || draft.title || draft.scope || draft.previousStart || draft.previousEnd);
  }
  function setMessage(text, error = false) {
    message.textContent = text;
    message.className = error ? "notice multi-message" : "muted multi-message";
    message.hidden = !text;
  }
  function optionSelect(label, items, current) {
    const select = el("select");
    select.setAttribute("aria-label", label);
    for (const [value, name] of items) {
      const option = el("option", "", name);
      option.value = value;
      select.append(option);
    }
    select.value = current;
    return select;
  }
  function displayRange(start, end) {
    if (!start && !end) return "日期待填";
    return [start, end].filter(Boolean).join(" 至 ");
  }
  function labelsForPeriod() {
    return dateInfo.weekly ? ["本周", "上周"] : ["本期", "上期"];
  }
  function replaceDraft(next) {
    for (const key of Object.keys(draft)) delete draft[key];
    Object.assign(draft, next);
    renderInputs();
    refresh();
  }
  function changeSchema(next) {
    if (next === draft.schema) return;
    const prior = MULTI.stageLabels(draft);
    const nextLabels = MULTI.stageLabels({ ...draft, schema: next });
    const removed = prior.some((label, i) => !nextLabels.includes(label) &&
      draft.jobs.some(job => [job.current[i], job.previous[i]].some(value => value !== null && value !== undefined && value !== "")));
    if (removed && !window.confirm("切换阶段后，只保留同名阶段的数据。切换吗？")) {
      renderInputs();
      return;
    }
    draft.jobs.forEach(job => {
      job.current = nextLabels.map(label => job.current[prior.indexOf(label)] ?? null);
      job.previous = nextLabels.map(label => job.previous[prior.indexOf(label)] ?? null);
    });
    draft.schema = next;
    renderInputs();
    refresh();
  }
  function makeInput(label, key, type = "text", extra = "") {
    const node = field(type, key, draft[key], label, type === "text" ? label : "");
    if (type === "text") node.maxLength = 100;
    node.addEventListener("input", () => { draft[key] = node.value; refresh(); });
    return namedField(label, node, extra);
  }
  function renderStages(parent) {
    const list = el("div", "multi-stage-list");
    draft.stages.forEach((value, i) => {
      const node = field("text", "stage." + i, value, "阶段 " + (i + 1));
      node.maxLength = 32;
      node.addEventListener("change", () => {
        draft.stages[i] = node.value;
        renderInputs();
        refresh();
      });
      list.append(namedField("阶段 " + (i + 1), node));
    });
    const add = button("添加阶段", () => {
      let i = draft.stages.length + 1;
      while (draft.stages.includes("阶段 " + i)) i++;
      draft.stages.push("阶段 " + i);
      renderInputs();
      refresh();
    });
    const remove = button("移除末尾阶段", () => {
      const index = draft.stages.length - 1;
      const occupied = draft.jobs.some(job => [job.current[index], job.previous[index]]
        .some(value => value !== undefined && value !== null && value !== ""));
      if (occupied && !window.confirm("移除末尾阶段及对应人数？")) return;
      draft.stages.pop();
      draft.jobs.forEach(job => {
        job.current = job.current.slice(0, index);
        job.previous = job.previous.slice(0, index);
      });
      renderInputs();
      refresh();
    });
    add.disabled = draft.stages.length >= 8;
    remove.disabled = draft.stages.length <= 2;
    list.append(add, remove);
    parent.append(list);
  }
  function parseCells(cells, labels, periods) {
    const width = labels.length;
    if (cells.length < 2 || cells.length > width * periods + 1) {
      throw new Error("岗位「" + cells[0] + "」的列数与阶段对不上。请按粘贴区提示排列。");
    }
    const values = cells.slice(1);
    return Array.from({ length: width * periods }, (_, index) => {
      const text = (values[index] || "").trim();
      if (!text || text === "—" || text === "--") return null;
      const neutral = /^\d{1,3}(,\d{3})+$/.test(text) ? text.replace(/,/g, "") : text;
      const number = countValue(neutral);
      if (!validCount(number)) throw new Error("岗位「" + cells[0] + "」的「" + labels[index % width] + "」需填写整数或留空。");
      return number;
    });
  }
  function importBatch(text, side) {
    if (/^[\s\uFEFF]*\{/.test(text)) {
      const report = MULTI.parseImport(String(text).replace(/^\uFEFF/, ""));
      if (hasInput() && !window.confirm("用导入的 JSON 替换目前这份报告？")) return;
      replaceDraft(fromReport(report));
      setMessage("已导入 " + report.jobs.length + " 个岗位及报告日期。");
      return;
    }
    const labels = MULTI.stageLabels(draft);
    const rows = MULTI.parseTSV(text);
    if (rows.length && /^(岗位|岗位名称|职位|职位名称)$/.test(rows[0][0].trim())) {
      const captions = rows.shift().slice(1).map(label => label.trim().replace(/^(本周|上周|本期|上期)\s*/, "").replace(/\s*人数$/, ""));
      const expected = side === "both" ? [...labels, ...labels] : labels;
      if (captions.some((label, i) => label && label !== expected[i])) {
        throw new Error("表头顺序与当前阶段不同。请按提示排列，或只粘贴数据行。");
      }
    }
    if (!rows.length) throw new Error("请先复制包含岗位名称和人数的表格。");
    const jobs = new Map();
    for (const row of draftJobs()) {
      const name = row.name.trim();
      if (!name || jobs.has(name)) throw new Error("请先补全现有岗位名称，并合并同名岗位。");
      jobs.set(name, { name, current: row.current.slice(), previous: row.previous.slice() });
    }
    const encountered = new Set();
    for (const line of rows) {
      const name = line[0].trim();
      if (!name) throw new Error("每一行的第一列都要有岗位名称。");
      if (encountered.has(name)) throw new Error("岗位「" + name + "」出现在多行，请合并后再粘贴。");
      encountered.add(name);
      const values = parseCells(line, labels, side === "both" ? 2 : 1);
      const job = jobs.get(name) || { name, current: [], previous: [] };
      if (side === "both" || side === "current") job.current = values.slice(0, labels.length);
      if (side === "both" || side === "previous") job.previous = side === "both" ? values.slice(labels.length) : values;
      jobs.set(name, job);
    }
    draft.jobs = [...jobs.values()];
    renderInputs();
    refresh();
    setMessage("已录入 " + rows.length + " 个岗位。同名岗位已更新所选周期。");
  }
  function renderImport() {
    const box = el("details", "multi-import");
    box.open = batchOpen;
    box.addEventListener("toggle", () => { batchOpen = box.open; });
    const heading = el("summary", "", "批量粘贴或导入");
    const side = optionSelect("粘贴哪一周的数据", [
      ["current", "只填本周"], ["previous", "只填上周"], ["both", "同时填两周"],
    ], "current");
    const labels = MULTI.stageLabels(draft);
    const hint = el("p", "muted multi-import-hint", "在 Excel 或飞书复制数据行。列顺序为岗位、" + labels.join("、") + "。同时填两周时，先放本周各列，再接上周各列。");
    const input = el("textarea");
    input.setAttribute("aria-label", "粘贴岗位数据");
    input.placeholder = "在这里粘贴表格数据";
    const importButton = button("录入表格", () => {
      try { importBatch(input.value, side.value); }
      catch (error) { setMessage(error.message, true); }
    }, "btn");
    const json = el("input");
    json.type = "file";
    json.accept = ".json,application/json";
    json.addEventListener("change", async () => {
      try {
        if (!json.files[0]) return;
        const report = MULTI.parseImport((await json.files[0].text()).replace(/^\uFEFF/, ""));
        if (hasInput() && !window.confirm("用导入的 JSON 替换目前这份报告？")) return;
        replaceDraft(fromReport(report));
        setMessage("已导入 " + report.jobs.length + " 个岗位及报告日期。");
      } catch (error) { setMessage(error.message, true); }
      finally { json.value = ""; }
    });
    const actions = el("div", "multi-import-actions");
    actions.append(namedField("录入周期", side), importButton, namedField("导入 OpenClaw 数据（JSON）", json));
    box.append(heading, hint, input, actions);
    return box;
  }
  function renderGrid() {
    const wrap = el("div", "multi-grid-wrap");
    wrap.setAttribute("aria-label", "按岗位填写各阶段人数");
    wrap.tabIndex = 0;
    const table = el("table", "multi-grid");
    const labels = MULTI.stageLabels(draft);
    const top = table.createTHead().insertRow();
    const head = (label, css = "") => {
      const th = el("th", css, label);
      th.scope = "col";
      top.append(th);
    };
    head("岗位");
    labels.forEach(label => head("本周 " + label));
    labels.forEach(label => head("上周 " + label, "multi-previous"));
    head("操作");
    const body = table.createTBody();
    draft.jobs.forEach((job, i) => {
      const row = body.insertRow();
      const name = field("text", "name." + i, job.name, "岗位 " + (i + 1) + " 名称", "岗位名称");
      name.maxLength = 120;
      name.dataset.key = "name";
      name.addEventListener("input", () => { job.name = name.value; refresh(); });
      row.insertCell().append(name);
      for (const [period, title] of [["current", "本周"], ["previous", "上周"]]) {
        labels.forEach((label, j) => {
          const node = numberField(period + "." + i + "." + j, job[period][j] ?? "", "岗位 " + (i + 1) + " " + title + label);
          node.addEventListener("input", () => {
            while (job[period].length < j) job[period].push(null);
            job[period][j] = node.value;
            const invalid = Number.isNaN(countValue(node.value));
            node.classList.toggle("is-invalid", invalid);
            if (invalid) node.setAttribute("aria-invalid", "true");
            else node.removeAttribute("aria-invalid");
            refresh();
          });
          row.insertCell().append(node);
        });
      }
      const remove = button("移除", () => {
        if (anyValue(job) && !window.confirm("移除「" + (job.name || "未命名岗位") + "」及两周人数？")) return;
        draft.jobs.splice(i, 1);
        if (!draft.jobs.length) draft.jobs.push({ name: "", current: [], previous: [] });
        renderInputs();
        refresh();
      }, "plain remove");
      remove.setAttribute("aria-label", "移除岗位 " + (i + 1));
      row.insertCell().append(remove);
    });
    wrap.append(table);
    return wrap;
  }
  function applyRollWeek(report) {
    const length = duration(report.start, report.end);
    replaceDraft({
      ...draft, start: addDays(report.end, 1), end: addDays(report.end, length),
      previousStart: report.start, previousEnd: report.end,
      jobs: report.jobs.map(job => ({ name: job.name, current: [], previous: job.current.slice() })),
    });
    setMessage("已保留岗位，并把原本周人数带到上周列。");
  }
  function rollWeek() {
    if (!data || !data.jobs.some(job => job.stages.some(stage => stage.current !== null)) || !dateInfo.input.start) {
      setMessage("请先填写本周开始日期和本周人数。", true);
      return;
    }
    const report = data.report;
    setMessage("这会把当前本周人数带到下一份报告的上周列。可以先保存当前报告。");
    const confirmRoll = button("确认沿用到下周", () => applyRollWeek(report), "btn primary");
    const cancelRoll = button("取消", () => setMessage(""), "btn");
    message.append(confirmRoll, cancelRoll);
    confirmRoll.focus();
  }
  function renderInputs() {
    const meta = el("div", "multi-meta");
    const schema = optionSelect("阶段设置", [["four", "招聘四阶段"], ["six", "飞书六阶段"], ["custom", "自定义阶段"]], draft.schema);
    schema.addEventListener("change", () => changeSchema(schema.value));
    meta.append(
      makeInput("报告标题", "title", "text", "span-two"),
      makeInput("招聘范围（选填）", "scope"),
      namedField("阶段设置", schema),
      makeInput("本周开始", "start", "date"),
      makeInput("本周结束（自动，可改）", "end", "date"),
      makeInput("上周开始（自动，可改）", "previousStart", "date"),
      makeInput("上周结束（自动，可改）", "previousEnd", "date"),
    );
    const ranges = el("p", "muted span-all");
    ranges.dataset.multiPeriods = "";
    meta.append(ranges);
    if (draft.schema === "custom") renderStages(meta);
    const actions = el("div", "multi-edit-actions");
    actions.append(
      button("添加岗位", () => {
        draft.jobs.push({ name: "", current: [], previous: [] });
        renderInputs();
        refresh();
        const inputs = editor.querySelectorAll('[data-key="name"]');
        inputs[inputs.length - 1].focus();
      }, "btn"),
      button("载入演示", () => {
        if (hasInput() && !window.confirm("用演示数据替换当前填写的报告？")) return;
        replaceDraft(seededExample());
      }),
      button("沿用到下周", rollWeek, "plain multi-roll"),
    );
    editor.replaceChildren(meta, renderImport(), renderGrid(), actions, message);
  }
  function changeText(stage) {
    if (stage.delta === null) return "—";
    if (stage.previous === 0 && stage.current > 0) return "新增 " + displayCount(stage.current) + " 人";
    if (stage.delta === 0) return stage.rate === null ? "持平" : "持平 · 0.0%";
    const label = stage.delta > 0 ? "增加 " : "减少 ";
    return label + displayCount(Math.abs(stage.delta)) + " 人 · " + displayDelta(stage.rate);
  }
  function comparisonClass(stage) {
    return stage.delta > 0 ? "pos" : stage.delta < 0 ? "neg" : "";
  }
  function withCoverage(cell, coverage, total) {
    if (coverage < total) cell.append(el("small", "", coverage + "/" + total + " 个岗位有数据"));
  }
  function reportSummary(result, stageNames, labels) {
    const section = el("section", "multi-section");
    section.append(el("h3", "multi-section-title", "各阶段汇总"));
    const scroll = el("div", "multi-summary-scroll");
    const table = el("table", "multi-summary");
    const top = table.createTHead().insertRow();
    for (const label of ["阶段", labels[0] + "人数", labels[1] + "人数", "人数及环比变化"]) {
      const th = el("th", "", label);
      th.scope = "col";
      top.append(th);
    }
    const body = table.createTBody();
    result.totals.forEach((stage, i) => {
      const row = body.insertRow();
      row.insertCell().textContent = stageNames[i];
      const current = row.insertCell();
      const previous = row.insertCell();
      current.textContent = displayCount(stage.current);
      previous.textContent = displayCount(stage.previous);
      withCoverage(current, stage.currentCoverage, result.jobs.length);
      withCoverage(previous, stage.previousCoverage, result.jobs.length);
      const change = row.insertCell();
      change.textContent = changeText(stage);
      change.className = comparisonClass(stage);
    });
    scroll.append(table);
    section.append(scroll);
    return section;
  }
  function jobCards(result, stageNames, labels) {
    const section = el("section", "multi-section");
    section.append(el("h3", "multi-section-title", "分岗位漏斗"));
    const cards = el("div", "multi-cards");
    for (const job of result.jobs) {
      const card = el("article", "multi-card");
      card.append(el("h4", "", job.name));
      const max = Math.max(1, ...job.stages.map(stage => stage.current ?? 0));
      job.stages.forEach((stage, i) => {
        const row = el("div", "multi-card-stage");
        const counts = el("span", "numbers", labels[0] + " " + displayCount(stage.current));
        counts.append(el("small", "", labels[1] + " " + displayCount(stage.previous)));
        row.append(el("span", "multi-stage-name", stageNames[i]), counts);
        const track = el("div", "multi-bar-track");
        track.setAttribute("aria-hidden", "true");
        if (stage.current > 0) {
          const bar = el("span", "multi-bar");
          bar.style.width = (stage.current / max * 100) + "%";
          bar.style.backgroundColor = colors[i];
          track.append(bar);
        }
        row.append(track, el("span", "multi-comparison " + comparisonClass(stage), changeText(stage)));
        card.append(row);
      });
      cards.append(card);
    }
    section.append(cards);
    return section;
  }
  function makeOutput(parent, title) {
    const result = data;
    const dates = dateInfo.input;
    const labels = labelsForPeriod();
    const heading = el("div");
    heading.append(el("h2", "multi-heading", title));
    heading.append(el("p", "muted multi-period", [
      dates.scope.trim(), labels[0] + " " + displayRange(dates.start, dates.end),
      labels[1] + " " + displayRange(dates.previousStart, dates.previousEnd),
    ].filter(Boolean).join(" ｜ ")));
    parent.append(heading);
    if (!result || !result.jobs.some(job => job.stages.some(stage => stage.current !== null || stage.previous !== null))) {
      parent.append(el("p", "empty", "填写或粘贴岗位人数后，所有岗位会汇总在这份报告里。"));
      return;
    }
    const stageNames = MULTI.stageLabels(result.report);
    const warnings = result.warnings.slice();
    if (!result.totals.some(stage => stage.previousCoverage)) warnings.unshift("上周尚无人数，报告先展示本周数据。");
    if (!result.totals.some(stage => stage.currentCoverage)) warnings.unshift("本周人数待填写。");
    if (!dateInfo.ready) warnings.unshift("填写本周开始日期后，报告会列出两周日期，并启用下载。");
    else if (!dateInfo.adjacent) warnings.unshift("两个日期区间需前后相接且天数相同。请核对日期后查看环比。");
    for (const warning of warnings) parent.append(el("p", "notice", warning));
    parent.append(reportSummary(result, stageNames, labels), jobCards(result, stageNames, labels));
    parent.append(el("p", "multi-report-note",
      "空白数据用「—」显示。各岗位的图条按本周人数绘制。上周为 0、本周大于 0 时显示新增人数。"));
  }
  function cleanTitle() {
    return dateInfo.input.title.trim() || "招聘周报";
  }
  function safeName(suffix) {
    const stem = cleanTitle().replace(/[<>:"/\\|?*\u0000-\u001F]/g, "_").trim().slice(0, 70) || "招聘周报";
    return stem + "_" + dateInfo.input.start + "_" + dateInfo.input.end + suffix;
  }
  function saveHtml() {
    const wrapper = el("main", "shell");
    const article = el("article", "panel multi-output");
    const body = el("div", "multi-report-body");
    makeOutput(body, cleanTitle());
    article.append(body);
    wrapper.append(article);
    const title = el("title", "", cleanTitle());
    const css = Array.from(document.querySelectorAll("style")).map(style => style.textContent).join("\n");
    const page = '<!doctype html><html lang="zh-CN"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width, initial-scale=1">' +
      title.outerHTML + "<style>" + css + "</style></head>" + '<body class="print-multi">' + wrapper.outerHTML + "</body></html>";
    saveFile(new Blob([page], { type: "text/html;charset=utf-8" }), safeName(".html"));
    setMessage("已生成一份包含全部岗位的 HTML 报告。");
  }
  function exportCsv() {
    const stageNames = MULTI.stageLabels(data.report);
    const labels = labelsForPeriod();
    const rows = [["报告", cleanTitle()], ["招聘范围", data.report.scope], [labels[0], displayRange(data.report.start, data.report.end)],
      [labels[1], displayRange(data.report.previousStart, data.report.previousEnd)], [],
      ["岗位", "阶段", labels[0] + "人数", labels[1] + "人数", "增减人数", "环比", "本期有数据岗位数", "上期有数据岗位数"]];
    function write(job, metrics) {
      metrics.forEach((stage, i) => {
        rows.push([job, stageNames[i], csvCount(stage.current), csvCount(stage.previous), stage.delta ?? "",
          csvRate(stage.rate), stage.currentCoverage ?? "", stage.previousCoverage ?? ""]);
      });
    }
    write("合计", data.totals);
    data.jobs.forEach(job => write(job.name, job.stages));
    function csvCell(value) {
      const raw = String(value ?? "");
      const text = typeof value === "string" && /^[\s\uFEFF]*[=+\-@]/.test(raw) && !/^-\d+(\.\d+)?%$/.test(raw)
        ? "'" + raw : raw;
      return '"' + text.replace(/"/g, '""') + '"';
    }
    const csv = rows.map(row => row.map(csvCell).join(",")).join("\r\n") + "\r\n";
    saveFile(new Blob(["\uFEFF", csv], { type: "text/csv;charset=utf-8" }), safeName(".csv"));
    setMessage("已生成全部岗位的 CSV 表格。");
  }
  function refresh() {
    data = null;
    let error = "";
    try {
      dateInfo = resolved();
      data = MULTI.calculate({ ...dateInfo.input, jobs: draftJobs() });
      if (!dateInfo.adjacent) {
        [...data.totals, ...data.jobs.flatMap(job => job.stages)].forEach(stage => { stage.delta = null; stage.rate = null; });
      }
    } catch (err) {
      error = err.message;
      dateInfo ||= { input: draft, adjacent: false, ready: false };
    }
    notice.hidden = !error;
    notice.textContent = error;
    content.replaceChildren();
    if (!error) makeOutput(content, cleanTitle());
    const ready = Boolean(data && dateInfo.ready && data.jobs.some(job => job.stages.some(stage => stage.current !== null || stage.previous !== null)));
    buttons.forEach(item => { item.disabled = !ready; });
    const dates = editor.querySelector("[data-multi-periods]");
    if (dates) {
      const r = dateInfo.input;
      dates.textContent = "本周 " + displayRange(r.start, r.end) + " ｜ 上周 " + displayRange(r.previousStart, r.previousEnd);
    }
    onSave();
  }
  function mount(root, nextDraft, save, prepared = false) {
    draft = nextDraft;
    onSave = save;
    dateInfo = undefined;
    const input = el("details", "panel multi-entry");
    input.open = !prepared;
    input.append(el("summary", "", "录入岗位与两周数据"));
    editor = el("div");
    message = el("p", "muted multi-message");
    message.setAttribute("role", "status");
    message.hidden = true;
    input.append(editor);
    const report = el("section", "panel multi-output");
    const top = el("div", "report-top");
    top.id = "multiExports";
    top.append(el("span", "muted", "所有岗位共用一份报告"));
    const actions = el("div", "downloads");
    buttons = [
      button("下载报告 HTML", saveHtml, "btn primary"),
      button("打印 / 保存 PDF", () => { document.body.classList.add("print-multi"); window.print(); }, "btn"),
      button("下载 CSV", exportCsv, "btn"),
    ];
    actions.append(...buttons);
    top.append(actions);
    notice = el("p", "notice");
    notice.setAttribute("role", "status");
    notice.hidden = true;
    content = el("div");
    const body = el("div", "multi-report-body");
    body.append(notice, content);
    report.append(top, body);
    root.replaceChildren(...(prepared ? [report, input] : [input, report]));
    renderInputs();
    refresh();
  }
  window.addEventListener("afterprint", () => document.body.classList.remove("print-multi"));
  return Object.freeze({ blank, fromReport, mount });
})();

</script>
<script id="report-app">
"use strict";
const byId = id => document.getElementById(id);
const storageKey = "recruit-report-offline-v1";
let active = "multi";
let preparedReport = null;
let preparedError = "";
let storageAvailable = true;
let drafts = Object.fromEntries(Object.keys(MODES).map(mode => [mode, mode === "boss" && INCLUDE_WORK_EXAMPLES ? exampleDraft(mode) : blankDraft(mode)]));
let currentReport;
let currentMeta;
let chartFrame;
let lastWidth = 0;
let exportVersion = 0;
try {
  const raw = byId("initial-multi-report").textContent.trim();
  if (raw && raw !== "null") preparedReport = MULTI.parseImport(raw);
} catch (error) {
  preparedError = error.message;
}
try {
  const store = preparedReport ? null : localStorage.getItem(storageKey);
  if (store) {
    const saved = JSON.parse(store);
    if (Object.hasOwn(MODES, saved.active) && saved.drafts) {
      for (const mode of Object.keys(MODES)) {
        if (saved.drafts[mode]) drafts[mode] = { ...drafts[mode], ...saved.drafts[mode] };
      }
      active = saved.active;
    }
  }
} catch {
  storageAvailable = false;
}
if (preparedReport) {
  drafts.multi = MultiUI.fromReport(preparedReport);
  active = "multi";
}
function saveDrafts() {
  if (preparedReport) {
    byId("storageNote").textContent = "这是预填报告。编辑后可另存报告或下载 CSV。";
    return;
  }
  if (storageAvailable) {
    try {
      localStorage.setItem(storageKey, JSON.stringify({ active, drafts }));
    } catch {
      storageAvailable = false;
    }
  }
  byId("storageNote").textContent = storageAvailable
    ? "填写内容保存在当前浏览器。下载的 CSV 可用 Excel 打开。"
    : "此浏览器未开启本地保存。关闭前可下载图表和数据表。";
}
function el(tag, className = "", text = "") {
  const node = document.createElement(tag);
  if (className) node.className = className;
  if (text) node.textContent = text;
  return node;
}
function field(type, path, value, label, placeholder = "") {
  const node = el("input");
  node.type = type;
  node.dataset.path = path;
  node.setAttribute("aria-label", label);
  if (type === "checkbox") node.checked = value;
  else node.value = value;
  if (placeholder) node.placeholder = placeholder;
  if (path === "scope" || path.endsWith(".label")) node.maxLength = 42;
  return node;
}
function namedField(label, node, extra = "") {
  const wrapper = el("label", "field" + (extra ? " " + extra : ""), label);
  wrapper.append(node);
  return wrapper;
}
function numberField(path, value, label) {
  const node = field("text", path, value, label, "—");
  node.inputMode = "numeric";
  node.autocomplete = "off";
  node.maxLength = 16;
  node.className = "number";
  node.dataset.count = "";
  return node;
}
function rateField(path, value, label) {
  const node = field("text", path, value, label, "—");
  node.autocomplete = "off";
  node.maxLength = 16;
  node.className = "number";
  node.dataset.rate = "";
  return node;
}
function button(text, callback, className = "plain") {
  const node = el("button", className, text);
  node.type = "button";
  node.addEventListener("click", callback);
  return node;
}
function removeButton(row, label, minLength = 1) {
  const node = button("移除", () => {
    drafts[active].rows.splice(row, 1);
    renderEditor();
    updateReport();
  }, "plain remove");
  node.setAttribute("aria-label", "移除" + label);
  node.disabled = drafts[active].rows.length <= minLength;
  return node;
}
function cohortCheck(draft) {
  const wrapper = el("div");
  const check = el("label", "check");
  const input = field("checkbox", "cohort", draft.cohort, "计算流失转化率");
  input.setAttribute("aria-describedby", "cohortHelp");
  check.append(
    input,
    el("span", "", "计算流失转化率"),
  );
  const help = el("p", "muted cohort-help", "各阶段都填最初沟通的那批人。例如，本周沟通 58 人，其中 20 人下周上传简历，这批人的上传人数填 20。");
  help.id = "cohortHelp";
  wrapper.append(check, help);
  return wrapper;
}
function choiceButton(label, pressed, action) {
  const node = button(label, action, "mode");
  node.setAttribute("aria-pressed", String(pressed));
  return node;
}
function renderModeButtons() {
  const nav = byId("modes");
  if (!nav.children.length) {
    Object.entries(MODES).forEach(([mode, info]) => {
      const item = button(info.name, () => {
        active = mode;
        renderModeButtons();
        showActiveMode();
      }, "mode");
      item.dataset.mode = mode;
      nav.append(item);
    });
  }
  for (const item of nav.children) item.setAttribute("aria-pressed", String(item.dataset.mode === active));
}
function showActiveMode() {
  const isMulti = active === "multi";
  byId("singleLayout").hidden = isMulti;
  byId("multiLayout").hidden = !isMulti;
  if (isMulti) {
    MultiUI.mount(byId("multiLayout"), drafts.multi, saveDrafts, Boolean(preparedReport));
    if (preparedError) byId("storageNote").textContent = "预填报告读取失败。" + preparedError;
    return;
  }
  renderEditor();
  updateReport();
}
function renderEditor() {
  const draft = drafts[active];
  const editor = byId("editor");
  const actions = byId("editActions");
  editor.replaceChildren();
  actions.replaceChildren();
  byId("entryHint").textContent = MODES[active].hint;
  byId("loadExample").textContent = MODES[active].sample;
  byId("editStatus").textContent = "";
  const fields = el("div", "fields");
  if (!isTrend(active)) {
    const period = el("select");
    period.dataset.path = "periodType";
    period.setAttribute("aria-label", "报表周期");
    Object.entries(PERIODS).forEach(([value, label]) => {
      const option = el("option", "", label);
      option.value = value;
      period.append(option);
    });
    period.value = draft.periodType;
    fields.append(namedField("报表周期", period, "wide"));
    if (draft.periodType === "month") {
      const month = field("month", "month", draft.month, "统计月份", "YYYY-MM");
      fields.append(namedField("统计月份", month));
    } else {
      fields.append(namedField("开始日期", field("date", "start", draft.start, "开始日期")));
    }
    if (draft.periodType === "custom") fields.append(namedField("结束日期", field("date", "end", draft.end, "结束日期")));
    else {
      const end = el("p", "muted period-end");
      end.id = "derivedEnd";
      fields.append(end);
    }
  }
  fields.append(namedField("招聘范围（选填）", field("text", "scope", draft.scope, "招聘范围", "如 2027 校招、某岗位或某负责人"), "wide"));
  editor.append(fields);
  const rows = el("div", "rows");
  if (active === "boss" || active === "jobrate") {
    const isRate = active === "jobrate";
    const layout = isRate ? "jobrate" : "boss";
    const top = el("div", "entry " + layout + " head");
    top.append(el("span", "", "岗位"), el("span", "number", isRate ? "沟通" : "沟通人数"));
    if (isRate) top.append(el("span", "number", "二面"));
    top.append(el("span"));
    rows.append(top);
    draft.rows.forEach((row, i) => {
      const wrapper = el("div", "entry " + layout);
      wrapper.append(
        field("text", "rows." + i + ".label", row.label, "岗位 " + (i + 1) + " 名称", "岗位名称"),
        numberField("rows." + i + ".current", row.current, "岗位 " + (i + 1) + " 沟通人数"),
      );
      if (isRate) wrapper.append(numberField("rows." + i + ".second", row.second, "岗位 " + (i + 1) + " 二面人数"));
      wrapper.append(removeButton(i, row.label || "岗位 " + (i + 1)));
      rows.append(wrapper);
    });
    if (isRate) rows.append(cohortCheck(draft));
    actions.append(button("添加岗位", () => {
      draft.rows.push(isRate ? { label: "", current: "", second: "" } : { label: "", current: "" });
      renderEditor();
      updateReport();
      editor.querySelector('[data-path="rows.' + (draft.rows.length - 1) + '.label"]').focus();
    }));
  } else if (active === "four" || active === "six") {
    const six = active === "six";
    const manual = six && draft.comparison === "rate";
    const labels = six ? SIX : FOUR;
    const layout = six ? "periods" : "stages";
    if (six) {
      const switches = el("div", "chart-choice");
      switches.setAttribute("aria-label", "环比填法");
      [["count", "填上期人数"], ["rate", "直接填环比"]].forEach(([key, title]) => {
        switches.append(choiceButton(title, draft.comparison === key, () => {
          draft.comparison = key;
          renderEditor();
          updateReport();
        }));
      });
      rows.append(switches);
    }
    const top = el("div", "entry " + layout + " head");
    top.append(el("span", "", "阶段"), el("span", "number", six ? "本期" : "人数"));
    if (six) top.append(el("span", "number", manual ? "环比 (%)" : "上期"));
    rows.append(top);
    labels.forEach((label, i) => {
      const wrapper = el("div", "entry " + layout);
      wrapper.append(el("span", "stage-label", label), numberField("current." + i, draft.current[i], label + "本期人数"));
      if (six) wrapper.append(manual
        ? rateField("changes." + i, draft.changes[i], label + "环比百分数")
        : numberField("prior." + i, draft.prior[i], label + "上期人数"));
      rows.append(wrapper);
    });
    if (!six) rows.append(cohortCheck(draft));
  } else {
    draft.rows.forEach((row, i) => {
      const wrapper = el("div", "trend-row");
      const top = el("div", "trend-top");
      top.append(namedField("周期 " + (i + 1), field("text", "rows." + i + ".label", row.label, "周期 " + (i + 1) + " 名称", "如 11/13–11/19")), removeButton(i, "周期 " + (i + 1), 2));
      const grid = el("div", "trend-stage-grid");
      trendInputs(active).forEach((label, j) => {
        grid.append(namedField(label, numberField("rows." + i + ".values." + j, row.values[j], "周期 " + (i + 1) + " " + label)));
      });
      wrapper.append(top, grid);
      rows.append(wrapper);
    });
    if (active === "eff") rows.append(cohortCheck(draft));
    actions.append(button("添加周期", () => {
      draft.rows.push(trendRow(active));
      renderEditor();
      updateReport();
      editor.querySelector('[data-path="rows.' + (draft.rows.length - 1) + '.label"]').focus();
    }));
  }
  editor.append(rows);
  actions.append(button("清空本模式", () => {
    if (!window.confirm("清空当前模式填写的数据？")) return;
    drafts[active] = blankDraft(active);
    renderEditor();
    updateReport();
    byId("editStatus").textContent = "当前模式已清空。";
  }));
}
function renderTable(data) {
  const root = byId("tableWrap");
  const table = el("table");
  const head = table.createTHead().insertRow();
  data.headers.forEach(value => {
    const cell = el("th", "", value);
    cell.scope = "col";
    head.append(cell);
  });
  const body = table.createTBody();
  data.table.forEach((values, index) => {
    const row = body.insertRow();
    if (data.totalRow && index === data.table.length - 1) row.className = "total";
    values.forEach((value, column) => {
      const cell = row.insertCell();
      if (active === "six" && column === 3) {
        if (data.items[index].rate > 0) cell.className = "pos";
        if (data.items[index].rate < 0) cell.className = "neg";
      }
      cell.textContent = value;
    });
  });
  root.replaceChildren(table);
  root.hidden = data.table.length === 0;
}
function renderLegend(data) {
  const legend = byId("legend");
  legend.hidden = !isTrend(active) || !data.hasData;
  if (legend.hidden) {
    legend.replaceChildren();
    return;
  }
  if (legend.dataset.mode !== active || !legend.children.length) {
    legend.dataset.mode = active;
    legend.replaceChildren();
    data.series.forEach((name, i) => {
      const item = button("", () => {
        const enabled = drafts[active].enabled;
        if (enabled[i] && enabled.filter(Boolean).length === 1) return;
        enabled[i] = !enabled[i];
        updateReport();
      }, "");
      const swatch = el("span", "swatch");
      swatch.style.backgroundColor = COLORS[i];
      item.append(swatch, document.createTextNode(name));
      legend.append(item);
    });
  }
  Array.from(legend.children).forEach((item, i) => {
    item.setAttribute("aria-pressed", String(data.enabled[i]));
    item.disabled = data.enabled[i] && data.enabled.filter(Boolean).length === 1;
  });
}
function renderChartChoice() {
  const wrap = byId("chartChoice");
  wrap.hidden = active !== "four";
  if (wrap.hidden) return;
  const draft = drafts.four;
  const options = [["funnel", "人数漏斗"], ["loss", "阶段流失"]];
  if (!wrap.children.length) {
    options.forEach(([key, title]) => {
      wrap.append(choiceButton(title, draft.chartMode === key, () => {
        drafts.four.chartMode = key;
        updateReport();
      }));
    });
  }
  Array.from(wrap.children).forEach((item, i) => {
    item.setAttribute("aria-pressed", String(draft.chartMode === options[i][0]));
  });
}
function updateReport() {
  exportVersion++;
  const draft = drafts[active];
  const data = reportData(active, draft);
  currentReport = data;
  const dates = reportDates(draft);
  const label = isTrend(active) ? "" : PERIODS[draft.periodType];
  currentMeta = [label, dateText(dates), draft.scope.trim()].filter(Boolean).join(" ｜ ");
  const derivedEnd = byId("derivedEnd");
  if (derivedEnd) derivedEnd.textContent = dates.end ? "截止 " + dates.end : "自动计算结束日期";
  byId("reportTitle").textContent = MODES[active].title;
  byId("reportMeta").textContent = currentMeta;
  byId("reportMeta").hidden = !currentMeta;
  byId("notice").textContent = data.notes.join(" ");
  byId("notice").hidden = data.notes.length === 0;
  byId("chartWrap").hidden = !data.chartReady;
  byId("empty").hidden = data.chartReady;
  byId("empty").textContent = data.empty;
  byId("tableNote").textContent = data.note || "";
  byId("tableNote").hidden = !data.note;
  byId("csv").disabled = !data.hasData || data.blocked;
  byId("png").disabled = !data.chartReady || data.blocked;
  byId("downloadStatus").hidden = true;
  for (const node of byId("editor").querySelectorAll("[data-count], [data-rate]")) {
    const parse = node.hasAttribute("data-rate") ? changeValue : countValue;
    const invalid = Number.isNaN(parse(node.value));
    node.classList.toggle("is-invalid", invalid);
    if (invalid) node.setAttribute("aria-invalid", "true");
    else node.removeAttribute("aria-invalid");
  }
  renderTable(data);
  renderLegend(data);
  renderChartChoice();
  scheduleChart();
  saveDrafts();
}
byId("editor").addEventListener("input", event => {
  const node = event.target;
  if (!node.dataset.path) return;
  if (node.dataset.path === "periodType") {
    const draft = drafts[active];
    const previous = reportDates(draft);
    if (node.value === "custom") {
      draft.start = previous.start;
      draft.end = previous.end;
    } else if (node.value === "month" && !draft.month && previous.start) {
      draft.month = previous.start.slice(0, 7);
    } else if (!draft.start) draft.start = previous.start;
    draft.periodType = node.value;
    renderEditor();
    byId("editor").querySelector("select").focus();
    updateReport();
    return;
  }
  const segments = node.dataset.path.split(".");
  const key = segments.pop();
  const target = segments.reduce((value, segment) => value[segment], drafts[active]);
  target[key] = node.type === "checkbox" ? node.checked : node.value;
  updateReport();
});
byId("loadExample").addEventListener("click", () => {
  const current = drafts[active];
  const filled = JSON.stringify(current) !== JSON.stringify(blankDraft(active));
  if (filled && !window.confirm("用示例替换当前模式的数据？")) return;
  drafts[active] = exampleDraft(active);
  renderEditor();
  updateReport();
  byId("editStatus").textContent = "已载入数据。";
});

const chartFont = '"Microsoft YaHei","PingFang SC",sans-serif';
function paintText(ctx, value, x, y, options = {}) {
  const { size = 14, weight = 500, color = "#213f49", align = "left", maxWidth } = options;
  ctx.font = weight + " " + size + "px " + chartFont;
  ctx.fillStyle = color;
  ctx.textAlign = align;
  ctx.textBaseline = "middle";
  if (maxWidth) {
    const chars = Array.from(value);
    while (chars.length > 1 && ctx.measureText(chars.join("")).width > maxWidth) {
      chars.pop();
      if (chars[chars.length - 1] === "…") chars.pop();
      chars[chars.length - 1] = "…";
    }
    ctx.fillText(chars.join(""), x, y);
  } else ctx.fillText(value, x, y);
}
function line(ctx, x1, y1, x2, y2, color, size = 1) {
  ctx.strokeStyle = color;
  ctx.lineWidth = size;
  ctx.beginPath();
  ctx.moveTo(x1, y1);
  ctx.lineTo(x2, y2);
  ctx.stroke();
}
function stageHeight(mode, data, width) {
  if (isTrend(mode)) return width < 460 ? 250 : 295;
  if (mode === "boss" || mode === "jobrate" || data.lossChart) {
    return ((data.lossChart ? data.lossItems : data.items).length * 39) + 53;
  }
  return (data.items.length * (mode === "six" ? 51 : 55)) + (data.metricText && width < 440 ? 76 : 42);
}
function paintBars(ctx, width, data, kind) {
  const min = width < 430;
  const label = min ? Math.min(109, width * .31) : Math.min(165, width * .31);
  const edge = 4;
  const items = kind === "loss" ? data.lossItems : data.items;
  const isRate = kind === "jobrate";
  const showShare = kind === "boss";
  const barWidth = Math.max(15, width - label - (showShare ? (min ? 105 : 132) : 74));
  const values = items.map(r => isRate ? r.rate : r.current);
  const max = Math.max(isRate ? .01 : 1, ...values.map(v => Number.isFinite(v) ? v : 0));
  const title = data.metricText && showShare ? data.metricText : isRate ? "从 BOSS 沟通到进入二面" : "各阶段减少人数";
  paintText(ctx, title, edge, 15, { size: 17, weight: 750, color: "#174c40", maxWidth: width - 8 });
  items.forEach((r, i) => {
    const y = 50 + 39 * i;
    const value = values[i];
    paintText(ctx, r.label, edge, y, { size: min ? 13 : 14, maxWidth: label - 8 });
    ctx.fillStyle = "#eaf0ed";
    ctx.fillRect(label, y - 8, barWidth, 16);
    if (value > 0) {
      ctx.fillStyle = i % 2 === 0 ? "#11735f" : "#36937e";
      ctx.fillRect(label, y - 8, Math.max(1, barWidth * value / max), 16);
    }
    paintText(ctx, isRate ? displayRate(value) : displayCount(value), label + barWidth + 12, y, { size: 14, weight: 700, maxWidth: showShare && min ? 48 : 60 });
    if (showShare) paintText(ctx, displayRate(r.share), width - edge, y, { size: 12, align: "right", color: "#536775" });
  });
}
function paintFunnel(ctx, mode, width, data) {
  const isSix = mode === "six";
  const small = width < 440;
  const area = Math.min(width * (small ? .43 : .55), 440);
  const x = area / 2;
  const labelStart = area + (small ? 12 : 20);
  const max = Math.max(1, ...data.items.map(r => validCount(r.current) ? r.current : 0));
  const rowHeight = isSix ? 51 : 55;
  let offset = 18;
  if (data.metricText) {
    const metrics = data.metricText.split("    ");
    paintText(ctx, metrics[0], 4, 15, { size: 16, weight: 750, color: "#174c40" });
    if (small) paintText(ctx, metrics[1], 4, 39, { size: 13, color: "#536775" });
    else paintText(ctx, metrics[1], 204, 15, { size: 13, color: "#536775" });
    offset = small ? 62 : 40;
  }
  const span = area - 14;
  data.items.forEach((r, i) => {
    const y = offset + i * rowHeight;
    const bar = validCount(r.current) ? span * r.current / max : 0;
    const bottom = bar * .78;
    const color = COLORS[i];
    line(ctx, 5, y + 14, area - 5, y + 14, "#e5edeb");
    if (bar > 0) {
      ctx.fillStyle = color;
      ctx.beginPath();
      ctx.moveTo(x - bar / 2, y - 1);
      ctx.lineTo(x + bar / 2, y - 1);
      ctx.lineTo(x + bottom / 2, y + 28);
      ctx.lineTo(x - bottom / 2, y + 28);
      ctx.closePath();
      ctx.fill();
    } else if (r.current === 0) {
      line(ctx, x, y + 8, x, y + 20, color, 2);
    }
    paintText(ctx, r.label, labelStart, y + 4, { size: small ? 13 : 15, weight: 650, maxWidth: width - labelStart - 4 });
    let secondary = displayCount(r.current) + (validCount(r.current) ? " 人" : "");
    if (isSix && r.rate !== null) secondary += "   环比 " + displayDelta(r.rate);
    if (!isSix && r.rate !== null) secondary += "   转化 " + displayRate(r.rate);
    paintText(ctx, secondary, labelStart, y + 24, { size: small ? 12 : 13, color: "#536775", maxWidth: width - labelStart - 4 });
  });
}
function paintTrend(ctx, width, height, data) {
  const isValue = v => data.asRate ? Number.isFinite(v) : validCount(v);
  const maxValue = Math.max(data.asRate ? .04 : 1, ...data.periods.flatMap(row => row.values.filter((v, i) => data.enabled[i] && isValue(v))));
  const power = 10 ** Math.floor(Math.log10(maxValue / 4));
  const unit = Math.max(data.asRate ? .01 : 1, Math.ceil(maxValue / 4 / power) * power);
  const max = unit * 4;
  const left = width < 430 ? 38 : 48;
  const top = 14;
  const bottom = height - 46;
  const right = width - 14;
  for (let i = 0; i <= 4; i++) {
    const y = bottom - (bottom - top) * i / 4;
    line(ctx, left, y, right, y, i === 0 ? "#a9bdb7" : "#e0ebe7");
    const tick = data.asRate ? Math.round(unit * i * 100) + "%" : countFormat.format(unit * i);
    paintText(ctx, tick, left - 7, y, { size: 11, align: "right", maxWidth: left - 8, color: "#536775" });
  }
  const steps = data.periods.length - 1;
  const spacing = (right - left) / steps;
  const perLabel = Math.max(1, Math.ceil(76 / spacing));
  const pointX = index => left + index * spacing;
  const pointY = value => bottom - (bottom - top) * value / max;
  data.periods.forEach((row, i) => {
    const last = i === steps;
    const enoughGap = steps - i >= perLabel;
    if (last || (i % perLabel === 0 && (enoughGap || i === 0))) {
      paintText(ctx, row.label || "—", pointX(i), bottom + 19, { size: 11, align: i === 0 ? "left" : last ? "right" : "center", maxWidth: 72, color: "#536775" });
    }
  });
  data.series.forEach((_, i) => {
    if (!data.enabled[i]) return;
    let previous = null;
    data.periods.forEach((row, j) => {
      const v = row.values[i];
      if (!isValue(v)) {
        previous = null;
        return;
      }
      const point = [pointX(j), pointY(v)];
      if (previous) line(ctx, ...previous, ...point, COLORS[i], 2);
      ctx.beginPath();
      ctx.fillStyle = COLORS[i];
      ctx.arc(...point, 3.5, 0, Math.PI * 2);
      ctx.fill();
      previous = point;
    });
  });
}
function paintChart(ctx, mode, width, height, data) {
  if (mode === "boss" || mode === "jobrate" || data.lossChart) paintBars(ctx, width, data, data.lossChart ? "loss" : mode);
  else if (isTrend(mode)) paintTrend(ctx, width, height, data);
  else paintFunnel(ctx, mode, width, data);
}
function scheduleChart() {
  cancelAnimationFrame(chartFrame);
  if (!currentReport.chartReady) return;
  chartFrame = requestAnimationFrame(() => {
    const chart = byId("chart");
    const width = Math.floor(byId("chartWrap").getBoundingClientRect().width);
    if (width <= 0) return;
    lastWidth = width;
    const height = stageHeight(active, currentReport, width);
    const scale = Math.min(2, window.devicePixelRatio || 1);
    chart.width = Math.floor(width * scale);
    chart.height = Math.floor(height * scale);
    chart.style.height = height + "px";
    const ctx = chart.getContext("2d");
    ctx.scale(scale, scale);
    paintChart(ctx, active, width, height, currentReport);
  });
}
if (window.ResizeObserver) {
  new ResizeObserver(() => {
    const width = Math.floor(byId("chartWrap").getBoundingClientRect().width);
    if (width > 0 && width !== lastWidth) scheduleChart();
  }).observe(byId("chartWrap"));
} else window.addEventListener("resize", scheduleChart);

function fileName(extension) {
  const { start, end } = reportDates(drafts[active]);
  const date = [start, end].filter(Boolean).join("_");
  const title = extension === "png" && currentReport.lossChart ? "各阶段流失人数" : MODES[active].title;
  return title + (date ? "_" + date : "") + "." + extension;
}
function saveFile(blob, name) {
  const url = URL.createObjectURL(blob);
  const link = el("a");
  link.href = url;
  link.download = name;
  document.body.append(link);
  link.click();
  link.remove();
  setTimeout(() => URL.revokeObjectURL(url), 30000);
}
function showExportStatus(text) {
  byId("downloadStatus").hidden = false;
  byId("downloadStatus").textContent = text;
}
byId("csv").addEventListener("click", () => {
  const draft = drafts[active];
  const { start, end } = reportDates(draft);
  const metadata = isTrend(active) ? [] : [["报表周期", PERIODS[draft.periodType]], ["开始日期", start], ["结束日期", end]];
  if (draft.scope.trim()) metadata.push([draft.scope.trim()]);
  if (currentReport.csvSummary) metadata.push(...currentReport.csvSummary);
  currentReport.notes.forEach(note => metadata.push([note]));
  const rows = [[MODES[active].title], ...metadata, [], currentReport.csvHeaders, ...currentReport.csvRows];
  const text = rows.map(csvLine).join("\r\n") + "\r\n";
  saveFile(new Blob(["\uFEFF", text], { type: "text/csv;charset=utf-8" }), fileName("csv"));
  showExportStatus("已生成 CSV，可在浏览器下载记录查看。");
});
byId("png").addEventListener("click", () => {
  const width = 760;
  const side = 36;
  const chartWidth = width - 2 * side;
  const data = currentReport;
  const mode = active;
  const height = stageHeight(mode, data, chartWidth);
  const chartTop = isTrend(mode) ? 140 : 110;
  const fullHeight = chartTop + height + 46;
  const image = el("canvas");
  const scale = Math.max(1, Math.min(2, 8192 / fullHeight));
  image.width = width * scale;
  image.height = Math.ceil(fullHeight * scale);
  const ctx = image.getContext("2d");
  ctx.scale(scale, scale);
  ctx.fillStyle = "#fff";
  ctx.fillRect(0, 0, width, fullHeight);
  ctx.fillStyle = "#126b57";
  ctx.fillRect(0, 0, width, 8);
  paintText(ctx, data.lossChart ? "各阶段流失人数" : MODES[mode].title, side, 44, { size: 26, weight: 750, color: "#182d3a" });
  paintText(ctx, currentMeta, side, 78, { size: 14, color: "#536775", maxWidth: chartWidth });
  if (isTrend(mode)) {
    let x = side;
    data.series.forEach((name, i) => {
      if (!data.enabled[i]) return;
      ctx.fillStyle = COLORS[i];
      ctx.fillRect(x, 108, 9, 9);
      paintText(ctx, name, x + 14, 112, { size: 12 });
      x += data.asRate ? 142 : 102;
    });
  }
  ctx.save();
  ctx.translate(side, chartTop);
  paintChart(ctx, mode, chartWidth, height, data);
  ctx.restore();
  const caption = data.notes.join(" ") || data.note;
  if (caption) paintText(ctx, caption, side, fullHeight - 21, { size: 12, color: "#536775", maxWidth: chartWidth });
  const name = fileName("png");
  const version = exportVersion;
  image.toBlob(blob => {
    if (blob) saveFile(blob, name);
    if (version === exportVersion) showExportStatus(blob ? "已生成 PNG，可在浏览器下载记录查看。" : "图片生成失败，请减少图表行数后重试。");
  }, "image/png");
});

renderModeButtons();
showActiveMode();
</script>
</body>
</html>
```
<!-- OPENCLAW_REPORT_HTML_END -->
