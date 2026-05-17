# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

单文件 HTML 狼人杀法官助手（`werewolf-judge.html`），纯前端无依赖，直接在浏览器打开使用。

## 如何开发/测试

```
start "" "E:\First-Claude Code\Werewolf-judge\werewolf-judge.html"
```

无构建、无 lint、无测试套件。修改后刷新浏览器即可验证。

## 架构

### 屏幕/阶段

三屏切换：`setup` → `assign` → `game`，由 `showScreen(name)` 控制。

游戏内通过 `state.isDay` 区分白天/夜晚，`dayMode` 细分为 `'normal' | 'deathShoot' | 'pickWolf'`。

### 核心状态 (`state`)

| 字段 | 说明 |
|---|---|
| `mode` | 9 或 12 人局 |
| `badgeRule` | 警徽规则开关（仅12人） |
| `wolfBossType` / `guardianType` | 12人局狼王类型（wolfking/whitewolf）和守卫类型（guard/fool） |
| `players[]` | 每个玩家：`{number, role, isAlive, antidoteUsed, poisonUsed, canShoot, lastGuarded, guardedThisNight, exploded}` |
| `night` | 当前夜晚编号（从1开始） |
| `phaseOrder` / `phaseIdx` | 夜晚行动阶段队列和当前位置 |
| `badgeHolder` / `badgeLocked` / `badgeTornUp` / `badgeEverAssigned` / `badgeLastHolder` | 警徽系统状态（见下方警徽系统章节） |

### 夜晚行动阶段

`buildPhaseOrder()` 按狼人→预言家→女巫→守卫顺序生成阶段队列（只加入存活角色）。

每个阶段点击号码牌执行行动，反复点击可更改选择。点击"进入下一阶段"时 `nextPhase()` 记录最终行动日志并推进阶段。全部阶段完成后 `resolveNight()` 结算死亡。

### 白天行动

- **投票放逐**：点击号码选中 → 投票放逐按钮。猎人/狼王被放逐后进入 `deathShoot` 模式。
- **自爆**：白狼王自爆（进入 `deathShoot` 模式带人）和狼人自爆（多狼时进入 `pickWolf` 模式选择具体狼人）。
- **死亡开枪**：`deathShootInfo` 记录谁在开枪，`deathShootEndsDay` 决定开枪后是否自动进入黑夜。

### 警徽系统

五个状态字段控制流程：

| 字段 | 说明 |
|---|---|
| `badgeHolder` | 当前警徽持有者号码（null=无） |
| `badgeLocked` | 分配确认后锁定，持有者死亡时解锁 |
| `badgeTornUp` | 撕毁后本局不再使用警徽 |
| `badgeEverAssigned` | 是否曾分配过警徽（区分首次分配和死亡后重新分配） |
| `badgeLastHolder` | 前任持有者号码（用于撕毁日志记录是谁撕毁） |

流程：

1. **首次分配**：白天点击号码 → `showBadgeAssignConfirm(n)` 弹窗确认 → `confirmBadgeAssign(n)` 锁定，记录「X号Y获得警徽」
2. **持有者死亡**：`handleBadgeTransfer()` 清空 holder 并解锁，UI 显示撕毁按钮 + 重新分配入口
3. **撕毁**：`showTearUpBadgeConfirm()` → `confirmTearUpBadge()` 设置 `badgeTornUp=true`，记录「X号Y撕毁警徽」
4. **锁定期**：`badgeLocked=true` 期间点击号码正常进入投票选择，不会误触警徽变更
5. **历史记录**：`buildGameSummary()` 过滤掉「死亡，警徽待转移」过渡日志，只保留获得/撕毁里程碑

### 关键函数职责

- `updateRoleConfig()` — 渲染设置界面身份卡片 + 12p切换卡片
- `getRoleConfig()` — 返回当前模式的实际角色列表（切换选择已展开）
- `renderAssignment()` — 渲染身份分配界面（平民不在调色板，自动填充）
- `confirmAssignment()` — 确认分配，未分配号码自动变平民
- `onPlayerClick(n)` — 全局点击分发（优先级：deathShoot → pickWolf → badge → 正常投票）
- `nextPhase()` — 阶段推进，记录日志，预判屠边，调用 `checkWinCondition()`
- `resolveNight()` — 夜晚结算：守卫/解药/毒药/同守同救逻辑
- `checkWinCondition()` — 屠边判定（狼人全灭→好人胜，神职全灭/平民全灭→狼人胜）
- `predictiveWinCheck()` — 狼人选杀后预判是否必然屠边（无守卫/解药可救时提前结束）
- `showBadgeAssignConfirm(n)` / `confirmBadgeAssign(n)` — 警徽分配确认流程
- `showTearUpBadgeConfirm()` / `confirmTearUpBadge()` — 警徽撕毁确认流程

### 女巫药水

- `witch.antidoteUsed` / `witch.poisonUsed` — 永久标记
- `state.witchUsedAntidote` — 当夜标记（每夜重置），阻止同夜使用两种药水
- 毒药选择可撤销（点击切换），在 `nextPhase()` 时确认生效

### 游戏结束

`checkWinCondition()` 在8个节点被调用：`nextPhase()`、`resolveNight()`、`startNextNight()`、`voteOut()`、`executeDeathShoot()`、`skipDeathShoot()`、`whitewolfSelfDestruct()`、`predictiveWinCheck()`。

两步弹窗：先 `showGameOverBanner()` 宣告胜负，用户点击后 `showGameSummary()` 展示每回合行动记录。

### 日志

`addLog()` 写入 `state.log[]`。阶段行动在 `nextPhase()` 中一次性记录（不在每次点击时记录），结算结果在 `resolveNight()` 中记录。`buildGameSummary()` 按 `=== ` 开头的里程碑分割回合，并在历史展示中过滤掉警徽过渡日志。

## 主题系统

设计风格：「月夜哥特」(Moonlit Gothic) — 深邃暗色 + 古铜金点缀 + 微妙噪点纹理 + 多层径向渐变背景。

### CSS 变量

| 变量 | 暗色值 | 亮色值 | 用途 |
|---|---|---|---|
| `--bg` | `#08081a` | `#ede5d5` | 页面背景 |
| `--bg-panel` | `#0d0d24` | `#faf6ee` | 面板/区域背景 |
| `--bg-card` | `#111130` | `#f5f0e5` | 卡片/号码牌背景 |
| `--bg-input` | `#0f0f28` | `#ede5d8` | 按钮/输入类背景 |
| `--text` | `#e4dcc8` | `#2a2218` | 主文字色 |
| `--text-secondary` | `#a09880` | `#5a4e3a` | 次级文字 |
| `--text-muted` | `#6b6458` | `#7a6e5a` | 弱化文字 |
| `--border` | `#1c1c3a` | `#d8ccb0` | 主边框 |
| `--border-card` | `#1c1c3a` | `#d0c4a8` | 卡片边框 |
| `--border-strong` | `#282850` | `#b8a880` | 强调边框 |
| `--accent` | `#c8a84e` | `#a08030` | 古铜金强调色 |
| `--toggle-bg` | `#2a2a4a` | `#c0b898` | 开关背景 |

角色/状态语义色（如 `#f44336` 狼人、`#4caf50` 好人）保持不变，不纳入主题变量。

### 字体

通过 Google Fonts 引入 **ZCOOL XiaoWei**（站酷小薇）用于标题和号码（支持中文），正文使用系统中文字体栈 `'Microsoft YaHei', 'PingFang SC', sans-serif`。离线时回退到系统字体。

### 主题切换

`initTheme()` 从 `localStorage['werewolf-theme']` 读取偏好，`toggleTheme()` 切换 `body.light` class 并持久化。右上角固定按钮触发。暗色主题还包含 `body::before` 伪元素的 SVG 噪点纹理叠加和径向渐变光晕效果。
