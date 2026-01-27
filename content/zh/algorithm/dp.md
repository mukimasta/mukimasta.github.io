---
author : ['Mukii']
title: "动态规划 (Dynamic Programming)"
description: "LeetCode 笔记：动态规划"
date: 2026-01-26
lastmod: 2026-01-26
type: post
draft: false
translationKey: algorithm_dp
coffee: 2
tags: ['algorithm', 'dynamic_programming']
categories: ['algorithm']
Math: true
---


## 1  概述

### 1.1  状态

状态是一组量化参数，用于描述一个特定局面。

例如：
- 象棋：状态可以是棋盘上所有棋子的坐标
- 自动驾驶控制：状态可以是 `(位置 x, 位置 y, 速度 v, 航向角 θ, 角速度 ω)`
- 下一个 Token 预测 (LLM本质)：状态可以是当前所有文本序列

**为状态赋予问题：**
$$
\text{状态 (State)} \xrightarrow[\text{算法 (Algorithm)}]{\text{问题 (Problem)}} \text{解/值 (Solution/Value)}
$$
`V = f(State)`

例如：
- 最短路径
	$\text{当前节点 } u \xrightarrow[\text{Dijkstra / BFS}]{\text{求到终点的最小累积权重}} \text{最短距离数值 } d[u]$
- 自动驾驶控制
	$\text{车辆状态 } (x, y, v, \theta) \xrightarrow[\text{MPC (模型预测控制)}]{\text{求未来 N 秒代价最小的轨迹}} \text{控制指令 (转角, 油门)}$
- 下一个 Token 预测 (LLM本质)
	$\text{上下文序列 } (t_1, \dots, t_k) \xrightarrow[\text{Transformer + Softmax}]{\text{求出现概率最大的下一个词}} \text{Token } t_{k+1} \text{ 的概率分布}$
- 量化高频交易
	$\text{订单快照 + 持仓} \xrightarrow[\text{统计套利 / 强化学习}]{\text{求预期收益最大化的操作}} \text{交易信号 (买/卖/停)}$

状态的值通常是最优值、方案数、概率
例如：
- 象棋：当前状态下红方最应该如何行动
- 自动驾驶控制：当前状态下的撞车概率
- 下一个 Token 预测 (LLM本质)：当前状态下下一个 Token 的预测

### 1.2  状态转移与路径

状态在规则和约束下，可以转移到另一个状态，同时伴随着状态值的变化。
$$
\text{State A} (V_A) \xrightarrow[\text{规则 (Rules)，约束 (Constraints)}]{\text{行动 (Action)}} \text{State B} (V_B)
$$
例如：
- 象棋
	$\text{局面A} (红方优势+2子) \xrightarrow[\text{棋规}]{\text{移动炮}} \text{局面B} (红方优势+3子)$
- 自动驾驶
	$(x, y, v, \theta) (\text{到达时间 10 分钟}) \xrightarrow[\text{物理定律, 避障}]{\text{加速+转向}} (x', y', v', \theta') (\text{到达时间 5 分钟})$

路径是一条由连续的状态转移构成的状态序列

例如：
初始状态 → 状态 A → 状态 B → ... → 目标状态

子状态：路径上离目标状态更近的状态
子问题：子状态下的同一问题


### 1.3  动态规划（问题）三要素

符合以下要素的问题适合用动态规划求解：
- **最优子结构**：原问题的最优解一定包含（依赖）子问题的最优解
	“原问题的最优解可以直接从子问题的最优解构建得来”
- **无后效性**（马尔可夫性）：后续决策只依赖当前状态，而与达到该状态的具体路径无关
	“论前面怎么走的，只要到达该状态，后面都一样处理”
	“只需记住“当前局面”，不用关心怎么到达这里的过程”
- **重复子问题**：有大量相同子问题被多次重复求解

Ask AI: 
```
举几个例子，符合最优子结构但不符合无后效性，以及反过来
```

### 1.4  动态规划

如果问题满足上述三要素，
可以将目标问题分解为子问题求解：
`f(S) -> f(s1), f(s2), ...`

$$
V = f(S)
\rightarrow
V = g[f(s1), f(s2), ..., s1, s2, ...]
$$

动态规划是一种策略：通过“记忆”子问题的解，来避免重复计算，从而高效解决原问题的算法。

1. 找出合适的状态和问题（通常就是原问题）
2. 使用 `dp[state]` 数组记录各状态下的值
3. 写状态转移方程 `dp[S] = f(dp[s1], dp[s2], ..., s1, s2, ...)`记录当前状态的值如何从子状态的值转移得来，从初始状态一直记录到目标状态
4. 最后由各状态的值求出最终解（通常是目标状态的值）