---
title: '从 MDP 到 PPO：手推笔记'
date: '2026-11-11T00:00:00-07:00'
draft: false
math: true
tags: ['reinforcement-learning', 'ppo']
summary: '从强化学习最基本的定义出发，一步步推到 PPO 的 clip 目标函数，并说明每一步证明了什么。'
---

本笔记按推导顺序整理：从强化学习最基本的定义出发，一路推到 PPO 的 clip 目标函数。每一步列出**结论**、**证明了什么**、**为什么需要它**，以及推导时容易犯的错误。

## 路线图

| 阶段 | 内容 | 产出 |
|---|---|---|
| 0 | 基本定义 | 轨迹概率、回报、$V$/$Q$/$A$、$\delta_t$ |
| 1 | 目标函数 | $J(\theta)$ |
| 2 | 策略梯度 | $\nabla J=\mathbb{E}[\nabla\log\pi\cdot A]$ |
| 3 | 重要性采样 | 比值 $\rho_t(\theta)$ |
| 4 | 替代目标 | $L^{\text{CPI}}=\mathbb{E}[\rho\hat A]$ |
| 5 | Clip | $L^{\text{CLIP}}$ |

---

## 第 0 阶段：基本定义

### 0.1 MDP 与轨迹概率

MDP 由 $(\mathcal S,\mathcal A,P,r,\gamma)$ 描述：

- $P(s'\mid s,a)$：环境的状态转移概率；
- $r(s,a)$：奖励函数，下文记 $r_t=r(s_t,a_t)$；
- $\gamma\in[0,1)$：折扣因子。

每一步：$a_t\sim\pi_\theta(\cdot\mid s_t)$，$s_{t+1}\sim P(\cdot\mid s_t,a_t)$。由马尔可夫性，一条轨迹的概率是

$$
p_\theta(\tau)=\rho_0(s_0)\prod_{t}\pi_\theta(a_t\mid s_t)\,P(s_{t+1}\mid s_t,a_t)
$$

**证明了什么：** 轨迹概率可以拆成"初始分布 × 策略 × 环境转移"的连乘。

**为什么需要：** 三项中**只有 $\pi_\theta$ 依赖 $\theta$**。第 2 阶段取对数再求梯度时，$\rho_0$ 和 $P$ 都会消失，因此策略梯度不需要环境模型。

> 易错：把 $P(s_0,a_0)$ 当成一项。$s_0$ 来自环境的初始分布 $\rho_0$，$a_0$ 来自策略，两者要分开写。

### 0.2 回报

$$
G_t=\sum_{k\ge0}\gamma^k r_{t+k},\qquad G_t=r_t+\gamma G_{t+1}
$$

若 $|r_t|\le R_{\max}$，由三角不等式和等比级数 $\sum_k\gamma^k=\frac1{1-\gamma}$：

$$
|G_t|\le\sum_k\gamma^k|r_{t+k}|\le\frac{R_{\max}}{1-\gamma}
$$

**证明了什么：** 回报满足一步递推；只要 $\gamma<1$，回报就有界。

**为什么需要：** 递推式是 Bellman 方程的来源；有界性保证 $V$ 和 $Q$ 是有限的数。$\frac1{1-\gamma}$ 称为**有效视野**，$\gamma=0.99$ 时约为 100 步。

> 易错：递推写成 $G_t=\gamma G_{t+1}$，漏掉当前奖励 $r_t$。

**补充：等比级数。** 部分和 $S_n=1+\gamma+\dots+\gamma^{n-1}$，两边乘 $\gamma$ 再相减得 $S_n=\frac{1-\gamma^n}{1-\gamma}$。当 $0\le\gamma<1$ 时 $\gamma^n\to0$，所以无穷和收敛到 $\frac1{1-\gamma}$；当 $\gamma\ge1$ 时发散。

### 0.3 价值函数的定义

$G_t$ 是随机变量，评价一个状态或动作要对它取条件期望：

$$
V^\pi(s)=\mathbb{E}_\pi[G_t\mid s_t=s],\qquad Q^\pi(s,a)=\mathbb{E}_\pi[G_t\mid s_t=s,a_t=a]
$$

竖线右边是**固定的**量，下标 $\pi$ 表示**之后**的动作按 $\pi$ 采样。

| | 被平均掉的随机变量 |
|---|---|
| $V^\pi(s)$ | $a_t,\ s_{t+1},\ a_{t+1},\ \dots$ |
| $Q^\pi(s,a)$ | $s_{t+1},\ a_{t+1},\ \dots$ |

**证明了什么：** $V$ 和 $Q$ 只差当前这一步动作 $a_t$ 的随机性。

> 易错：把 $Q$ 的下标写成 $a_t\sim\pi$。$a_t$ 已经被固定，不再随机。

### 0.4 $V$ 与 $Q$ 的两座桥

工具：**全期望公式** $\mathbb{E}[X\mid Y]=\sum_z P(z\mid Y)\,\mathbb{E}[X\mid Y,z]$。

- 对 $V$ 取 $z=a_t$：

$$
V^\pi(s)=\sum_a\pi(a\mid s)\,Q^\pi(s,a)
$$

- 对 $Q$ 代入 $G_t=r_t+\gamma G_{t+1}$，再取 $z=s_{t+1}$，并用马尔可夫性去掉多余的条件：

$$
Q^\pi(s,a)=r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)\,V^\pi(s')
$$

**证明了什么：** $V$ 是 $Q$ 对动作的平均；$Q$ 是即时奖励加上 $V$ 对下一状态的平均。两式互相代入即得 Bellman 方程：

$$
V^\pi(s)=\sum_a\pi(a\mid s)\Big[r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)V^\pi(s')\Big]
$$

$$
Q^\pi(s,a)=r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)\sum_{a'}\pi(a'\mid s')Q^\pi(s',a')
$$

> 易错：
> - 漏掉 $\gamma$；
> - 右边写 $V^\pi(s)$ 而非 $V^\pi(s')$；
> - 写 $Q=r+\gamma V(s_{t+1})$ 而没有对 $s'$ 求和。左边是确定的数，右边不能留随机量；
> - 用 $P(a\mid s)$ 表示策略。策略写 $\pi$，环境写 $P$，两者在求梯度时处理方式不同。

### 0.5 优势函数与 TD 误差

$$
A^\pi(s,a)=Q^\pi(s,a)-V^\pi(s)
$$

**结论 1：**

$$
\mathbb{E}_{a\sim\pi}[A^\pi(s,a)]=\sum_a\pi(a\mid s)Q^\pi(s,a)-V^\pi(s)=0
$$

**证明了什么：** 优势是相对于策略平均水平的**相对量**，按 $\pi$ 平均为零，所以总有一些动作优于平均、一些劣于平均。$A>0$ 应增大 $\pi(a\mid s)$，$A<0$ 应减小。

**结论 2：** 定义 $\delta_t=r_t+\gamma V^\pi(s_{t+1})-V^\pi(s_t)$，则

$$
\mathbb{E}[\delta_t\mid s_t,a_t]=r+\gamma\sum_{s'}P(s'\mid s,a)V^\pi(s')-V^\pi(s)=Q^\pi-V^\pi=A^\pi(s_t,a_t)
$$

（把常数放进对 $s'$ 的求和，用到 $\sum_{s'}P(s'\mid s,a)=1$。）

**证明了什么：** 只需一步采样 $(s_t,a_t,r_t,s_{t+1})$ 和一个 $V$ 网络，就能得到 $A^\pi$ 的**无偏估计**（前提是 $V$ 准确）。

**为什么需要：** $\delta_t$ 是 PPO 中 GAE 的基本单元。

**补充：两个概念。**
- **哑变量：** 只在求和号内部起作用的变量，换名字不改变结果，例如 $\sum_a\pi(a\mid s)Q(s,a)=\sum_{a'}\pi(a'\mid s)Q(s,a')$。
- **无偏 vs 一致：** 无偏指 $\mathbb{E}[\hat X]=X$，对单个样本就成立；一致指样本越多平均值越接近真值。无偏样本取平均会收敛到真值；有偏估计即使收敛，也会收敛到错误的值。例如 $V$ 学得不准时，$\delta_t$ 就有系统偏差。

---

## 第 1 阶段：目标函数

$$
J(\theta)=\sum_{s_0}\rho_0(s_0)\,V^{\pi_\theta}(s_0)=\mathbb{E}_{\tau\sim p_\theta}[R(\tau)]=\sum_\tau p_\theta(\tau)\,R(\tau)
$$

其中 $R(\tau)=\sum_k\gamma^k r(s_k,a_k)=G_0$。

**关键观察：** $\theta$ 只出现在**采样分布** $p_\theta(\tau)$ 中，$R(\tau)$ 本身不依赖 $\theta$。所以对 $J$ 求导时，导数落在分布上，这正是第 2 阶段要解决的困难。

> 易错：把初始分布写成 $\rho_\theta(s_0)$。初始分布由环境决定，与 $\theta$ 无关。

---

## 第 2 阶段：策略梯度

### 2.1 对数导数技巧与 REINFORCE

直接求导得 $\nabla J=\sum_\tau\nabla p_\theta(\tau)R(\tau)$，它不再是期望，无法用采样估计。

**恒等式：** $\nabla\log f=\frac{\nabla f}{f}$，所以 $\nabla p_\theta=p_\theta\nabla\log p_\theta$。代入得

$$
\nabla_\theta J=\mathbb{E}_{\tau\sim p_\theta}\big[\nabla_\theta\log p_\theta(\tau)\,R(\tau)\big]
$$

取对数把连乘变连加：

$$
\log p_\theta(\tau)=\log\rho_0(s_0)+\sum_t\log\pi_\theta(a_t\mid s_t)+\sum_t\log P(s_{t+1}\mid s_t,a_t)
$$

第一、三项与 $\theta$ 无关，导数为 0：

$$
\boxed{\nabla_\theta J=\mathbb{E}\Big[\sum_t\nabla_\theta\log\pi_\theta(a_t\mid s_t)\cdot R(\tau)\Big]}
$$

**证明了什么：** 对分布的求导可以重新写回期望，从而**用采样估计梯度**；环境转移 $P$ 被消掉，**不需要环境模型**。

> 易错：拆开连乘时只给第一项加 $\log$。$\log$ 作用在每一项上，结果是 $\nabla\log\pi$，不是 $\nabla\pi$。

### 2.2 核心引理

$$
\mathbb{E}_{a\sim\pi_\theta}[\nabla_\theta\log\pi_\theta(a\mid s)]=\sum_a\pi_\theta\frac{\nabla\pi_\theta}{\pi_\theta}=\sum_a\nabla\pi_\theta=\nabla\sum_a\pi_\theta(a\mid s)=\nabla1=0
$$

**证明了什么：** 所有动作概率的变化量之和为零。

**为什么需要：** 后面所有"某一项可以去掉而不改变梯度期望"的论证都依赖它。

### 2.3 基线

对任意只依赖状态的 $b(s_t)$：

$$
\mathbb{E}_{s_t,a_t}\big[\nabla\log\pi_\theta(a_t\mid s_t)\,b(s_t)\big]=\mathbb{E}_{s_t}\Big[b(s_t)\cdot\underbrace{\mathbb{E}_{a_t}[\nabla\log\pi_\theta(a_t\mid s_t)]}_{=0}\Big]=0
$$

**证明了什么：** 从回报中减去任何**不依赖 $a_t$** 的函数，梯度期望不变，但方差可以显著降低。取 $b=V^\pi(s_t)$ 效果很好，因为它是 $G_t$ 在 $s_t$ 下的平均值，减掉后剩下的量围绕 0 波动。

> 易错：把 $\nabla\log\pi\cdot b$ 当成 $\nabla\log(\pi b)$。$b$ 乘在外面，不进入 $\log$。

### 2.4 因果性

$a_t$ 之前的奖励只取决于 $a_{<t}$ 和 $s_{\le t}$，对 $a_t$ 来说已经确定，作用和基线一样，贡献的梯度期望为零。所以

$$
R(\tau)\ \longrightarrow\ \sum_{k\ge t}\gamma^k r_k=\gamma^t G_t
$$

（$\gamma^t$ 因子实践中通常省略。）

**证明了什么：** 去掉了与 $a_t$ 无关的噪声。这是 2.3 的特例，用的是同一个引理。

### 2.5 带优势的策略梯度

三步替换，每一步期望都不变：

$$
R(\tau)\ \xrightarrow{2.4}\ G_t\ \xrightarrow{2.3,\ b=V}\ G_t-V^\pi(s_t)\ \xrightarrow{\text{全期望}}\ A^\pi(s_t,a_t)
$$

最后一步的依据：$\mathbb{E}[f(X)\,Y]=\mathbb{E}_X\big[f(X)\,\mathbb{E}[Y\mid X]\big]$。取 $X=(s_t,a_t)$，$f=\nabla\log\pi_\theta(a_t\mid s_t)$ 只依赖 $X$，可以提出内层期望：

$$
\mathbb{E}[G_t-V^\pi(s_t)\mid s_t,a_t]=Q^\pi(s_t,a_t)-V^\pi(s_t)=A^\pi(s_t,a_t)
$$

$$
\boxed{\nabla_\theta J=\mathbb{E}\Big[\sum_t\nabla_\theta\log\pi_\theta(a_t\mid s_t)\cdot A^\pi(s_t,a_t)\Big]}
$$

**证明了什么：** 策略梯度可以写成"对数概率的梯度 × 优势"。三种形式期望相同，噪声依次减小：

| 形式 | 乘的是 | 噪声 |
|---|---|---|
| REINFORCE | $R(\tau)$ | 最大 |
| 加基线 | $G_t-V$（一次采样） | 中等 |
| 优势 | $A^\pi$（精确平均） | 最小，但需要估计 |

**成立条件：** 条件里必须包含前面因子所依赖的**全部**变量。若因子换成 $\nabla\log\pi_\theta(a_{t+1}\mid s_{t+1})$，就要固定 $(s_t,a_t,s_{t+1},a_{t+1})$，此时条件期望是 $r_t+\gamma Q^\pi(s_{t+1},a_{t+1})$，不再是 $Q^\pi(s_t,a_t)$。

> 易错：展开 $G_t=r_t+\gamma G_{t+1}$ 后认为 $G_{t+1}$、$V(s_{t+1})$ 与 $a_t$ 无关。$s_{t+1}\sim P(\cdot\mid s_t,a_t)$ 由当前动作决定。若真如此，梯度只依赖即时奖励；在 CartPole 中每步奖励都是 1，梯度会恒为零，策略什么也学不到。

---

## 第 3 阶段：重要性采样

### 3.1 动机

上面的期望要求样本来自**当前**策略 $\pi_\theta$。每更新一步，旧数据就失效，采样代价太高。我们希望用旧策略 $\pi_{\theta_{\text{old}}}$ 采一批数据，然后连续更新多步。

### 3.2 重要性采样恒等式

$$
\mathbb{E}_{x\sim p}[f(x)]=\sum_x q(x)\frac{p(x)}{q(x)}f(x)=\mathbb{E}_{x\sim q}\Big[\frac{p(x)}{q(x)}f(x)\Big]
$$

前提：$p(x)>0$ 时必须 $q(x)>0$。

### 3.3 应用到动作

定义

$$
\rho_t(\theta)=\frac{\pi_\theta(a_t\mid s_t)}{\pi_{\theta_{\text{old}}}(a_t\mid s_t)}
$$

$$
\mathbb{E}_{a\sim\pi_\theta}[\nabla\log\pi_\theta\cdot A]=\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\rho_t\,\nabla\log\pi_\theta\cdot A]
$$

且由于分母不依赖 $\theta$：

$$
\rho_t\,\nabla_\theta\log\pi_\theta=\frac{\pi_\theta}{\pi_{\theta_{\text{old}}}}\cdot\frac{\nabla\pi_\theta}{\pi_\theta}=\nabla_\theta\rho_t
$$

**证明了什么：** 对动作分布的修正是**精确的**，并且梯度可以写成 $\nabla\rho_t\cdot A$。

### 3.4 $\rho$ 的直觉与来历

- **本意：样本重加权。** 新策略选 $a$ 的概率是旧策略的 2 倍，旧数据里 $a$ 就"少了一半"，每个样本应算 2 次。$\rho$ 是为了重用旧数据而**推导出来**的权重。
- **副作用：度量策略变化。** $\rho=1$ 表示该样本上新旧策略一致；偏离 1 越多，变化越大。PPO 利用了这一点。
- **历史顺序：** CPI（2002）引入 $\mathbb{E}[\rho\hat A]$，此时 $\rho$ 只是重要性权重 → TRPO（2015）用 KL 散度约束新旧策略的距离，需要二阶优化 → PPO（2017）直接限制目标函数中已有的 $\rho$，一阶优化即可。
- **局限：** $\rho$ 只看被采样到的动作，KL 看整个动作分布。所以 clip 是一个粗略的信任区域，实践中还要监控 `approx_kl`。
- **小性质：** $\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\rho]=\sum_a\pi_\theta(a\mid s)=1$。

### 3.5 状态访问分布 $d^\pi$

把轨迹期望改写成对单个状态的期望：

$$
\mathbb{E}_\tau\Big[\sum_t\gamma^t g(s_t)\Big]=\sum_s\Big[\sum_t\gamma^t\Pr(s_t=s)\Big]g(s)
$$

归一化后得到

$$
d^\pi(s)=(1-\gamma)\sum_{t\ge0}\gamma^t\Pr(s_t=s\mid\pi)
$$

含义：按 $\pi$ 行动时，智能体有多大比例的时间处于 $s$。于是

$$
\nabla_\theta J=\frac1{1-\gamma}\,\mathbb{E}_{s\sim d^{\pi_\theta}}\,\mathbb{E}_{a\sim\pi_\theta}\big[\nabla\log\pi_\theta(a\mid s)\,A^{\pi_\theta}(s,a)\big]
$$

常数 $\frac1{1-\gamma}$ 可被学习率吸收。**$d^\pi$ 依赖策略**：CartPole 中好策略集中在杆子竖直的状态，差策略分散在歪斜的状态。训练时，**缓冲区里的状态就是 $d^{\pi_{\theta_{\text{old}}}}$ 的样本**。

---

## 第 4 阶段：替代目标

### 4.1 从精确梯度到可计算的梯度

| 步骤 | 式子 | 精确？ |
|---|---|---|
| 精确梯度 | $\mathbb{E}_{s\sim d^{\pi_\theta}}\mathbb{E}_{a\sim\pi_\theta}[\nabla\log\pi_\theta\cdot A^{\pi_\theta}]$ | — |
| 动作重要性采样 | $\mathbb{E}_{s\sim d^{\pi_\theta}}\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\nabla\rho\cdot A^{\pi_\theta}]$ | 精确 |
| 状态换成旧分布 | $s\sim d^{\pi_\theta}\to d^{\pi_{\theta_{\text{old}}}}$ | **近似 ①**：只有旧策略访问过的状态 |
| 优势换成旧估计 | $A^{\pi_\theta}\to\hat A\approx A^{\pi_{\theta_{\text{old}}}}$ | **近似 ②**：只能用旧回报估计 |

两处近似在 $\theta=\theta_{\text{old}}$ 时都精确，$\theta$ 越远误差越大。

- **近似 ① 的失效：** 新策略可能会去旧数据中几乎没有的状态，我们无法在那里评估它。
- **近似 ② 的失效：** $A^{\pi_{\theta_{\text{old}}}}(s,a)$ 假设之后按旧策略行动，而实际执行的是新策略。

**更深一层（性能差异引理）：**

$$
J(\theta)-J(\theta_{\text{old}})=\frac1{1-\gamma}\,\mathbb{E}_{s\sim d^{\pi_\theta}}\,\mathbb{E}_{a\sim\pi_\theta}\big[A^{\pi_{\theta_{\text{old}}}}(s,a)\big]
$$

这是一个精确等式，里面出现的本来就是旧策略的优势。从这个角度看，真正的近似只有状态分布一处。

此外，$\hat A$ 本身还有**估计噪声**（单次采样），它与策略变化无关，靠 GAE、增大批次、优势归一化来降低。

### 4.2 替代目标

需要一个标量函数交给自动求导，使其梯度等于 $\mathbb{E}_{\pi_{\theta_{\text{old}}}}[\nabla\rho_t\cdot\hat A_t]$。因为 $\hat A_t$ 与 $\theta$ 无关、采样分布与 $\theta$ 无关，$\nabla$ 可以提到最外面：

$$
\boxed{L^{\text{CPI}}(\theta)=\mathbb{E}_{s,a\sim\pi_{\theta_{\text{old}}}}\big[\rho_t(\theta)\,\hat A_t\big]}
$$

批次形式：$L=\frac1N\sum_i\rho_i(\theta)\hat A_i$，代码为 `loss = -(ratio * adv).mean()`（取负号是因为优化器做最小化）。

> 易错：写成 $L=\nabla\mathbb{E}[\dots]$。$L$ 是函数本身，$\nabla L$ 才是梯度。

### 4.3 在 $\theta_{\text{old}}$ 处梯度正确

先求梯度再代值。$\rho_t(\theta_{\text{old}})=1$，但 $\nabla\rho_t\ne0$：

$$
\nabla\rho_t\Big|_{\theta_{\text{old}}}=\frac{\nabla\pi_\theta}{\pi_{\theta_{\text{old}}}}\Big|_{\theta_{\text{old}}}=\frac{\nabla\pi_\theta}{\pi_\theta}=\nabla\log\pi_\theta
$$

$$
\nabla L^{\text{CPI}}\Big|_{\theta_{\text{old}}}=\mathbb{E}_{\pi_{\theta_{\text{old}}}}[\nabla\log\pi_\theta\cdot\hat A_t]=\nabla J\Big|_{\theta_{\text{old}}}
$$

**证明了什么：** $L^{\text{CPI}}$ 和 $J$ 在 $\theta_{\text{old}}$ 处梯度相同，局部朝同一方向倾斜；但这个保证只在附近成立。

### 4.4 无约束优化的问题

单个样本的贡献 $\rho\hat A$ 是关于 $\rho$ 的直线：

- $\hat A>0$：优化器把 $\rho$ 一路推大，几乎没有上限（$\rho\le1/\pi_{\theta_{\text{old}}}$）；
- $\hat A<0$：把 $\rho$ 推向 0。

后果：过拟合 $\hat A$ 的噪声；两处近似失效，$L$ 上升而 $J$ 下降；下一轮数据由变坏的策略采集，训练可能崩溃。

**结论：** 必须限制每一轮中 $\rho$ 偏离 1 的幅度。

---

## 第 5 阶段：Clip

### 5.1 目标函数

$$
\boxed{L^{\text{CLIP}}(\theta)=\mathbb{E}_{s,a\sim\pi_{\theta_{\text{old}}}}\Big[\min\big(\rho_t\hat A_t,\ \text{clip}(\rho_t,1-\epsilon,1+\epsilon)\,\hat A_t\big)\Big]}
$$

### 5.2 数值验证（$\epsilon=0.2$）

$\hat A=+2$：

| $\rho$ | $\rho\hat A$ | $\text{clip}(\rho)\hat A$ | $\min$ |
|---|---|---|---|
| 0.5 | 1 | 1.6 | 1 |
| 1.0 | 2 | 2 | 2 |
| 1.5 | 3 | 2.4 | 2.4 |

$\hat A=-2$：

| $\rho$ | $\rho\hat A$ | $\text{clip}(\rho)\hat A$ | $\min$ |
|---|---|---|---|
| 0.5 | −1 | −1.6 | −1.6 |
| 1.0 | −2 | −2 | −2 |
| 1.5 | −3 | −2.4 | −3 |

### 5.3 图像形状

- $\hat A>0$：$\rho\le1+\epsilon$ 时为斜率 $\hat A$ 的直线，之后水平。
- $\hat A<0$：$\rho<1-\epsilon$ 时水平（高度 $(1-\epsilon)\hat A$），之后为斜率 $\hat A$ 的直线，一路向下。

> 易错：设计时让函数在 $1+\epsilon$ 处跌到 0。这会惩罚已经在区间外的点，并且函数不连续。正确做法是保持水平，梯度为 0。

### 5.4 梯度表

| $\partial/\partial\rho$ | $\rho<1-\epsilon$ | $[1-\epsilon,1+\epsilon]$ | $\rho>1+\epsilon$ |
|---|---|---|---|
| $\hat A>0$ | $\hat A$（可以纠正） | $\hat A$ | $0$（截断） |
| $\hat A<0$ | $0$（截断） | $\hat A$ | $\hat A$（可以纠正） |

**证明了什么：** clip 只截断"**朝改进方向**走得太远"的那一侧；如果 $\rho$ 被推到了错误的一侧，梯度保留，可以拉回来。

### 5.5 为什么要取 $\min$

只用 $\text{clip}(\rho)\hat A$ 时，两端都是水平的。以 $\hat A>0$、$\rho=0.5$ 为例：只用 clip 时梯度为 0，错误无法纠正；取 $\min$ 时落在 $\rho\hat A$ 上，梯度为 $\hat A$，会被拉回。

### 5.6 悲观下界

$\min(x,y)\le x$，所以

$$
L^{\text{CLIP}}(\theta)\le L^{\text{CPI}}(\theta),\quad\text{当 }\rho\in[1-\epsilon,1+\epsilon]\text{ 时相等}
$$

**证明了什么：** $L^{\text{CLIP}}$ 是 $L^{\text{CPI}}$ 的**悲观下界**：超出信任区域的收益不计入，损失全额计入。在 $\theta_{\text{old}}$ 处两者的值和梯度都相同，所以 4.3 的结论仍然成立。

---

## 总结

| 阶段 | 结果 | 解决的问题 |
|---|---|---|
| 0 | $V,Q,A,\delta_t$ | 定义好坏；一步采样即可估计优势 |
| 1 | $J(\theta)=\mathbb{E}_\tau[R]$ | 用一个标量衡量策略 |
| 2 | $\nabla J=\mathbb{E}[\nabla\log\pi\cdot A]$ | 无需环境模型的梯度；基线降方差 |
| 3 | $\rho_t$ | 用旧数据估计新策略的梯度 |
| 4 | $L^{\text{CPI}}=\mathbb{E}[\rho\hat A]$ | 可交给自动求导的目标，但只在局部可信 |
| 5 | clip + min | 限制每轮更新幅度，用悲观下界保证稳定 |

### 与代码的对应（典型 PyTorch 实现）

| 代码 | 公式 |
|---|---|
| `delta = reward + gamma * next_value - value` | $\delta_t$（0.5） |
| `gae = delta + gamma * lam * ... * gae` | GAE 估计 $\hat A_t$ |
| 优势归一化 `(adv - mean) / std` | 减常数基线不改变梯度期望（2.3） |
| `ratio = exp(new_log_probs - old_log_probs)` | $\rho_t$（3.3） |
| `surr1 = ratio * adv`，`surr2 = clamp(ratio, 1-eps, 1+eps) * adv` | min 中的两项（5.1） |
| `approx_kl` | 监控 $\rho$ 之外的真实 KL（3.4） |
| `clip_fraction` | 落在梯度表"0"格子里的样本比例（5.4） |
| `value_loss` | 训练 $V$，因为 $\delta_t$ 需要准确的 $V$ |
| `entropy` | 鼓励探索（不属于策略梯度推导） |

### 可继续推导

1. **GAE：** 证明 $\hat A_t^{\text{GAE}}=\sum_l(\gamma\lambda)^l\delta_{t+l}$ 在 $\lambda=0$ 时为 $\delta_t$，在 $\lambda=1$ 时为 $G_t-V(s_t)$，并解释偏差–方差权衡。
2. **性能差异引理：** 证明 4.1 中的等式，说明为什么使用旧策略的优势是合理的。
