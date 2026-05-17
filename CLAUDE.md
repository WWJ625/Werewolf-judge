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
| `wolfBossType` / `guardianType` | 12人局狼王类型和守卫/愚者选择 |
| `players[]` | 每个玩家：`{number, role, isAlive, antidoteUsed, poisonUsed, canShoot, lastGuarded, guardedThisNight, exploded}` |
| `night` | 当前夜晚编号（从1开始） |
| `phaseOrder` / `phaseIdx` | 夜晚行动阶段队列和当前位置 |

### 夜晚行动阶段

`buildPhaseOrder()` 按狼人→预言家→女巫→守卫顺序生成阶段队列（只加入存活角色）。

每个阶段点击号码牌执行行动，反复点击可更改选择。点击"进入下一阶段"时 `nextPhase()` 记录最终行动日志并推进阶段。全部阶段完成后 `resolveNight()` 结算死亡。

### 关键函数职责

- `updateRoleConfig()` — 渲染设置界面身份卡片 + 12p切换卡片
- `getRoleConfig()` — 返回当前模式的实际角色列表（切换选择已展开）
- `renderAssignment()` — 渲染身份分配界面（平民不在调色板，自动填充）
- `confirmAssignment()` — 确认分配，未分配号码自动变平民
- `onPlayerClick(n)` — 全局点击分发（根据 `isDay`/`dayMode`/`phase` 路由到不同处理）
- `nextPhase()` — 阶段推进，记录日志，预判屠边，调用 `checkWinCondition()`
- `resolveNight()` — 夜晚结算：守卫/解药/毒药/同守同救逻辑
- `checkWinCondition()` — 屠边判定（狼人全灭→好人胜，神职全灭/平民全灭→狼人胜）
- `predictiveWinCheck()` — 狼人选杀后预判是否必然屠边（无守卫/解药可救时提前结束）

### 女巫药水

- `witch.antidoteUsed` / `witch.poisonUsed` — 永久标记
- `state.witchUsedAntidote` — 当夜标记（每夜重置），阻止同夜使用两种药水
- 毒药选择可撤销（点击切换），在 `nextPhase()` 时确认生效

### 游戏结束

`checkWinCondition()` 在8个节点被调用：`nextPhase()`、`resolveNight()`、`startNextNight()`、`voteOut()`、`executeDeathShoot()`、`skipDeathShoot()`、`whitewolfSelfDestruct()`、`predictiveWinCheck()`。

两步弹窗：先 `showGameOverBanner()` 宣告胜负，用户点击后 `showGameSummary()` 展示每回合行动记录。

### 日志

`addLog()` 写入 `state.log[]`。阶段行动在 `nextPhase()` 中一次性记录（不在每次点击时记录），结算结果在 `resolveNight()` 中记录。`buildGameSummary()` 按 `=== ` 开头的里程碑分割回合。
