# BUGFIX — v0.1.6

## Bug 标题

删除全部预设后保存，内置 jailbreak 预设被静默“复活”并重新激活——删除永远存不下来。

## 证据链（修复前）

设置页提供对每套预设的删除（含二次确认），`core.deletePreset()` 也明确处理了“删掉激活预设 / 删光全部”的情形（`activePresetId` 置 `""`）。但开箱状态下只有一套内置预设 `jailbreak`，用户把它删除并点击保存后，它立刻原样回来且仍是激活预设。

离线复现（纯 Node 直接加载 `lib/core.js`，`DSH_HOME` 指向临时目录，走设置页的真实路径：`deletePreset` 删光 → `saveMultiPreset` 保存 → `loadMultiPreset` 重读）：

- `first-run seeds builtin jailbreak` — PASS（首次初始化，预期行为）
- `deletePreset empties the doc` — PASS（内存中的文档确实被清空）
- **`save of emptied doc keeps 0 presets` — FAIL：保存后 `presets=jailbreak, active=jailbreak`**
- **`reload keeps 0 presets` — FAIL：重读后 `presets=jailbreak, active=jailbreak`**
- **`no active preset after deletion` — FAIL：`loadActivePreset()` 返回 jailbreak**
- 对照组（缺 `presets` 字段的文档应被内置补齐、正常文档往返）— 均 PASS

修复前脚本 exit 1（3 项失败）；修复后同一脚本 exit 0（7 项全过）。

## 根因

`lib/core.js` 的 `normalizeMultiPreset()` 有一条兜底规则：“规范化后预设数为 0 就用内置预设补齐”，本意是修复损坏/旧格式文档。但它不区分两种空：

1. 文档**根本没有** `presets` 字段（损坏/旧格式）——应该补齐；
2. 文档**显式带有空对象** `presets: {}`（用户在设置页删光全部预设后的合法状态）——不应补齐。

而 `saveMultiPreset()` 与 `loadMultiPreset()` 都要经过 `normalizeMultiPreset()`，于是情形 2 在每次保存、每次读取时都被静默改写为“内置 jailbreak + 设为激活”，删除结果永远无法持久。附带影响：导入一份 `presets: {}` 的多条结构 JSON 时，也会把 jailbreak 凭空合并进用户文档。

## 改动文件

- `lib/core.js`（唯一逻辑改动，`normalizeMultiPreset`，+10/−6）：
  - 新增 `hasPresetsField` 判定（`presets` 为非数组对象才算显式带有）；
  - 内置补齐仅在**未显式带有** `presets` 字段时触发；
  - 显式空对象时 `activePresetId` 回退为 `""`（与 `deletePreset` 既有约定一致），不再指向不存在的内置 id。
- `package.json`：version `0.1.5 → 0.1.6`；`repository.url` 改指本 fork 落点仓。包内无其他版本戳常量；`pnpm-lock.yaml` 中的 `version: 4.0.1` 是 `@deepseek-ai/cordis` 的 peer 规格、非本包版本，未动。
- `README.md`：顶部追加中文 lhg fork 头（修复版说明 + 出处 + 本版修复一句），上游原文一字未删。
- 新增 `UPSTREAM.md`、本文件；`LICENSE` 原样保留。

## 复验结果

- 复现脚本：修复前 exit 1（3 项失败）→ 修复后 exit 0（7 项全过），含两项回归对照：缺字段文档仍被内置补齐、正常文档往返不丢预设；首次运行仍种子化内置预设。
- `npm run lint`（`node --check` 三个入口）：修复前后均通过。
- `npm run build`（`scripts/assert-build.mjs` 发布校验）：修复前后均通过。
- 改动面：`git diff --stat` 仅 `lib/core.js` 一处逻辑文件。

## 验证边界

- 上游无自动化测试套件（`package.json` 只有 lint/build/release 脚本），本版以 core 层最小复现脚本做前后对照；上游 4 条 open issue 均为模糊的效果反馈/安装失败报告，无可离线复现的细节，未作为本版依据，本版缺陷由静态审查 + 复现实锤。
- 未在真实 DSH 宿主中端到端点击设置页验证；但设置页删除→保存走的正是 `POST /dsh-preset-plus/save → core.saveMultiPreset()` 同一代码路径，复现即覆盖该路径的服务端语义。
- 首次初始化（无保存文件时种子化内置预设，`initMultiPreset`）行为未改动；损坏到无法 JSON 解析的文件仍走原有初始化兜底，未受影响。
