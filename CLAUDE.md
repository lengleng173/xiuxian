# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

修仙题材 · 女性向文字养成游戏 demo（合欢宗第七峰）。整个游戏是一个**单文件 HTML 应用**：`合欢魔女养成Demo.html`（约 13,000 行 / 2.1 MB）。无依赖、无构建步骤、无测试、无 lint。

## 运行

直接用浏览器打开 `合欢魔女养成Demo.html` 即可，无需安装或构建。存档通过 `localStorage` 自动保存在浏览器本地（key 见 `SAVE_KEY`）。修改后刷新浏览器即可看到效果。

## 文件结构

`合欢魔女养成Demo.html` 是一个自包含文件，内部按以下顺序分层：

1. **`<style>`（约第 7–377 行）** — 全部 CSS。主题变量集中在 `:root`（`--bg`、`--rouge`、`--gold`、`--serif` 字体等）。
2. **`<script>`（约第 413–13367 行）** — 全部逻辑，用 `/* ================= X ================= */` 注释分段：
   - **`数据`**（约 413–12,405 行，占文件绝大部分）：所有内容/数据常量，见下文。
   - **`工具`**（约 12,445 行起）：状态工具、日志输出、进度阈值函数。
   - 游戏机制函数：行动、社交、事件触发、突破、结局。
   - **`绑定`**（约 13,339 行起）：DOM 事件监听，最后调用 `showIntro()` 启动。

## 核心架构：数据驱动

游戏逻辑（`freshState`、`render`、`performAction`、`rollLocEvent` 等）是通用引擎；**所有内容都是顶部定义的纯数据常量**。新增内容（剧情、事件、NPC、词条）几乎不需要改引擎代码，只需在对应数据常量中追加条目。

### 全局状态 `S`

`let S = null`（约 12,406 行），由 `freshState(originIdx, name)` 创建。结构：

- 修炼：`realm`（0–5）、`stage`、`xp`；媚术 `charmArts`/`charmExp`
- 四维：`charm` / `body` / `mind` / `demonic`（均以 `cap()` 限制在 0–300）
- 资源：`stones`（灵石）、`pei`（培元丹）、`jing`（静心丹）
- `npc`：`{ [id]: { fav, seen, talkSeen, routeStage, mainStage, lastRandTalk, lastInvite } }`
- 其他：`loc`（当前地点 id）、`buffs`、`flags`（解锁标志）、`over`/`endType`、`turns`、`sel`（当前选中 NPC）、`dialogueNpc`、`pendingChoices`

### 关键数据常量（均位于「数据」段）

| 常量 | 含义 |
| --- | --- |
| `REALMS` / `STAGES` / `ALIASES` / `SEALCH` | 境界、小阶段、称号、朱印字 |
| `ORIGINS` | 3 个开局出身（noble / servant / prodigy），含初始四维与资源 |
| `REGIONS` / `REGION_BY_ID` | 3 大地域：`hehuan` / `city` / `outer` |
| `LOCS` / `LOC_BY_ID` | 13 个地点，含 `region`、`npcs`、`desc`、`tag` |
| `LOC_ACTIONS` | 地点专属行动（温泉小憩、翻阅典籍、采药等） |
| `LORE` | 40 条见闻录词条，含 `flag`（解锁标志）、`cat`、`title`、`text` |
| `LOC_EVENTS`、`LOC_EVENTS_EXTRA`、`LOC_EVENTS_3` … `LOC_EVENTS_18` | 各地点的事件池（按 `locId` 分组）。注意 `LOC_EVENTS_18`/`LOC_EVENTS_17` 声明在 `LOC_EVENTS_EXTRA` 之前 |
| `NPCS` | 全部角色定义（`master`、`feiyu`、`jinghong`、`wenyu`、`wugou`、`yelan`、`suhe`、`xuanli`、`zhuzong`），含 `visits`、`gift`、`invite`、`events`、`talks`、`randTalks` 等 |
| `ZHONGZHU_FULL` | 宗主月华长篇主线（六卷 77 场），数组，每条 `{ title, need, text, apply }` |
| `PERSONAL_ROUTES` | 7 位角色个人线，`{ [id]: [ { title, need, text, apply } ] }` |
| `TALKS` / `TALKS_EXTRA` / `RAND_TALKS` / `INVITE_VARIANTS` | 对话话题、随机闲聊、邀约文案变体 |
| `EVENTS` / `FRIEND_EVENTS` | 修炼奇遇事件 / 姐妹友情事件 |
| `SAVE_KEY` | `'hehuan_monv_demo_v4'`，存档 key |

### 事件数据格式

两类事件对象（务必遵循此 shape）：

1. **选择分支事件**（`LOC_EVENTS*`、`EVENTS` 等）：
   ```js
   { id, title, weight, cond: ()=>bool, desc, choices: [
       { label, fn: (s, L) => { /* 修改 S，向 L push 日志 */ } }
   ] }
   ```
   - `weight` 用于随机加权；`cond(S)` 决定是否进入池。
   - 若事件含 `begin(s, L)` 而非 `choices`，则该函数返回选择项（见 `rollCultivationEvent`）。

2. **剧情推进条目**（`ZHONGZHU_FULL`、`PERSONAL_ROUTES`）：
   ```js
   { title, need, text, apply: (s, L) => { /* 修改 S，push 日志 */ } }
   ```
   `need` 是触发所需好感门槛；个人线用 `st.routeStage` 记录进度，宗主主线用 `st.mainStage`。

### 日志机制

事件效果通过 `L` 数组（`[cssClass, htmlString]` 元组）传递，最后 `flushLines(L)` 逐条渲染到 `#log`。日志 class 常用：`gain` / `dim` / `info`（见 `logLine`）。文案中嵌入数值与属性增减时用 `Math.min/max` + `cap()` 约束，如 `s.charm = Math.min(300, s.charm+4)`。

## 关键机制函数（引擎，一般无需改）

- `advanceMonth` / `checkEnd` / `endGame` / `tryBreakthrough` / `backlash` — 时间推进、失败/通关判定、突破
- `stageReq()` / `artLevelReq(l)` — 境界突破所需修为 / 媚术升级所需经验（公式集中在此，调整难度改这里）
- `performAction(id)` — 修行行动（`ACTIONS` 数组定义）；`performSocial(id)` — 社交行动（交谈/赠礼/邀约/主线/个人线）
- `rollLocEvent` / `rollCultivationEvent` / `rollFriendship` / `checkCharEvents` — 各类随机/条件事件触发
- `render()` — 唯一的渲染入口，按状态重绘整个 UI（顶栏、地图、行动区、人物区）

## 数据 vs 引擎的分界

修改内容时，优先在「数据」段追加常量；只有需要新机制（新行动类型、新结算规则、新 UI 区域）时才动引擎函数和 `render()`。注意 `render()` 与 `performSocial`/`performAction` 中存在大量按角色/地点硬编码的分支，改动某类行为时需同时检查这些分支是否受影响。
