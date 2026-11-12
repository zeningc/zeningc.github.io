---
title: 'From MDP to PPO: Deriving It by Hand'
date: '2026-11-11T00:00:00-07:00'
draft: false
math: true
tags: ['reinforcement-learning', 'ppo']
summary: 'Starting from the most basic definitions in reinforcement learning and deriving, step by step, the PPO clipped objective — with what each step proves.'
---

These notes follow the order of the derivation: starting from the most basic definitions in reinforcement learning and ending at PPO's clipped objective. For each step they list the **result**, **what it proves**, **why we need it**, and the mistakes that are easy to make along the way.

## Roadmap

| Stage | Topic | Output |
|---|---|---|
| 0 | Basic definitions | Trajectory probability, return, $V$/$Q$/$A$, $\delta_t$ |
| 1 | Objective | $J(\theta)$ |
| 2 | Policy gradient | $\nabla J=\mathbb{E}[\nabla\log\pi\cdot A]$ |
| 3 | Importance sampling | Ratio $\rho_t(\theta)$ |
| 4 | Surrogate objective | $L^{\text{CPI}}=\mathbb{E}[\rho\hat A]$ |
| 5 | Clipping | $L^{\text{CLIP}}$ |

---

## Stage 0: Basic Definitions

### 0.1 MDP and Trajectory Probability

An MDP is described by $(\mathcal S,\mathcal A,P,r,\gamma)$:

- $P(s'\mid s,a)$: the environment's state-transition probability;
- $r(s,a)$: the reward function; below we write $r_t=r(s_t,a_t)$;
- $\gamma\in[0,1)$: the discount factor.

At each step: $a_t\sim\pi_\theta(\cdot\mid s_t)$, $s_{t+1}\sim P(\cdot\mid s_t,a_t)$. By the Markov property, the probability of a trajectory is

$$
p_\theta(\tau)=\rho_0(s_0)\prod_{t}\pi_\theta(a_t\mid s_t)\,P(s_{t+1}\mid s_t,a_t)
$$

**What it proves:** a trajectory's probability factors into "initial distribution × policy × environment transitions".

**Why we need it:** of the three factors, **only $\pi_\theta$ depends on $\theta$**. When we take the log and differentiate in Stage 2, $\rho_0$ and $P$ drop out, so the policy gradient does not require a model of the environment.

> Common mistake: treating $P(s_0,a_0)$ as a single factor. $s_0$ comes from the environment's initial distribution $\rho_0$ and $a_0$ comes from the policy; write them separately.

### 0.2 Return

$$
G_t=\sum_{k\ge0}\gamma^k r_{t+k},\qquad G_t=r_t+\gamma G_{t+1}
$$

If $|r_t|\le R_{\max}$, then by the triangle inequality and the geometric series $\sum_k\gamma^k=\frac1{1-\gamma}$:

$$
|G_t|\le\sum_k\gamma^k|r_{t+k}|\le\frac{R_{\max}}{1-\gamma}
$$

**What it proves:** the return satisfies a one-step recursion, and as long as $\gamma<1$ it is bounded.

**Why we need it:** the recursion is where the Bellman equation comes from; boundedness guarantees that $V$ and $Q$ are finite numbers. $\frac1{1-\gamma}$ is called the **effective horizon**: about 100 steps for $\gamma=0.99$.

> Common mistake: writing the recursion as $G_t=\gamma G_{t+1}$ and dropping the current reward $r_t$.

**Aside: the geometric series.** For the partial sum $S_n=1+\gamma+\dots+\gamma^{n-1}$, multiply both sides by $\gamma$ and subtract to get $S_n=\frac{1-\gamma^n}{1-\gamma}$. When $0\le\gamma<1$, $\gamma^n\to0$, so the infinite sum converges to $\frac1{1-\gamma}$; when $\gamma\ge1$ it diverges.

### 0.3 Defining the Value Functions

$G_t$ is a random variable, so to evaluate a state or an action we take a conditional expectation:

$$
V^\pi(s)=\mathbb{E}_\pi[G_t\mid s_t=s],\qquad Q^\pi(s,a)=\mathbb{E}_\pi[G_t\mid s_t=s,a_t=a]
$$

What is to the right of the bar is **held fixed**; the subscript $\pi$ means that **subsequent** actions are sampled from $\pi$.

| | Random variables averaged out |
|---|---|
| $V^\pi(s)$ | $a_t,\ s_{t+1},\ a_{t+1},\ \dots$ |
| $Q^\pi(s,a)$ | $s_{t+1},\ a_{t+1},\ \dots$ |

**What it proves:** $V$ and $Q$ differ only in the randomness of the current action $a_t$.

> Common mistake: writing the subscript of $Q$ as $a_t\sim\pi$. $a_t$ is already fixed and is no longer random.

### 0.4 Two Bridges Between $V$ and $Q$

Tool: the **law of total expectation**, $\mathbb{E}[X\mid Y]=\sum_z P(z\mid Y)\,\mathbb{E}[X\mid Y,z]$.

- For $V$, take $z=a_t$:

$$
V^\pi(s)=\sum_a\pi(a\mid s)\,Q^\pi(s,a)
$$

- For $Q$, substitute $G_t=r_t+\gamma G_{t+1}$, take $z=s_{t+1}$, and use the Markov property to drop the extra conditions:

$$
Q^\pi(s,a)=r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)\,V^\pi(s')
$$

**What it proves:** $V$ is the average of $Q$ over actions; $Q$ is the immediate reward plus the average of $V$ over next states. Substituting one into the other gives the Bellman equations:

$$
V^\pi(s)=\sum_a\pi(a\mid s)\Big[r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)V^\pi(s')\Big]
$$

$$
Q^\pi(s,a)=r(s,a)+\gamma\sum_{s'}P(s'\mid s,a)\sum_{a'}\pi(a'\mid s')Q^\pi(s',a')
$$

> Common mistakes:
> - dropping $\gamma$;
> - writing $V^\pi(s)$ on the right-hand side instead of $V^\pi(s')$;
> - writing $Q=r+\gamma V(s_{t+1})$ without summing over $s'$. The left side is a fixed number, so the right side cannot contain a random quantity;
> - using $P(a\mid s)$ for the policy. Write $\pi$ for the policy and $P$ for the environment; they are treated differently when we take gradients.

### 0.5 Advantage and TD Error

$$
A^\pi(s,a)=Q^\pi(s,a)-V^\pi(s)
$$

**Result 1:**

$$
\mathbb{E}_{a\sim\pi}[A^\pi(s,a)]=\sum_a\pi(a\mid s)Q^\pi(s,a)-V^\pi(s)=0
$$

**What it proves:** the advantage is a **relative** quantity — measured against the policy's own average — and it averages to zero under $\pi$. So some actions are always better than average and some worse. If $A>0$ we should increase $\pi(a\mid s)$; if $A<0$, decrease it.

**Result 2:** define $\delta_t=r_t+\gamma V^\pi(s_{t+1})-V^\pi(s_t)$. Then

$$
\mathbb{E}[\delta_t\mid s_t,a_t]=r+\gamma\sum_{s'}P(s'\mid s,a)V^\pi(s')-V^\pi(s)=Q^\pi-V^\pi=A^\pi(s_t,a_t)
$$

(Moving the constants inside the sum over $s'$ uses $\sum_{s'}P(s'\mid s,a)=1$.)

**What it proves:** with a single transition $(s_t,a_t,r_t,s_{t+1})$ and a $V$ network, we get an **unbiased estimate** of $A^\pi$ (assuming $V$ is exact).

**Why we need it:** $\delta_t$ is the building block of GAE in PPO.

**Aside: two concepts.**
- **Dummy variable:** a variable that only lives inside a summation; renaming it does not change the result, e.g. $\sum_a\pi(a\mid s)Q(s,a)=\sum_{a'}\pi(a'\mid s)Q(s,a')$.
- **Unbiased vs. consistent:** unbiased means $\mathbb{E}[\hat X]=X$, which holds even for a single sample; consistent means the sample average approaches the true value as samples grow. Averaging unbiased samples converges to the true value; a biased estimator may still converge, but to the wrong value. For example, when $V$ is inaccurate, $\delta_t$ has a systematic bias.

---

## Stage 1: The Objective

$$
J(\theta)=\sum_{s_0}\rho_0(s_0)\,V^{\pi_\theta}(s_0)=\mathbb{E}_{\tau\sim p_\theta}[R(\tau)]=\sum_\tau p_\theta(\tau)\,R(\tau)
$$

where $R(\tau)=\sum_k\gamma^k r(s_k,a_k)=G_0$.

**Key observation:** $\theta$ appears only in the **sampling distribution** $p_\theta(\tau)$; $R(\tau)$ itself does not depend on $\theta$. So when we differentiate $J$, the derivative lands on the distribution — exactly the difficulty Stage 2 has to solve.

> Common mistake: writing the initial distribution as $\rho_\theta(s_0)$. The initial distribution is set by the environment and does not depend on $\theta$.

---

## Stage 2: The Policy Gradient

### 2.1 The Log-Derivative Trick and REINFORCE

Differentiating directly gives $\nabla J=\sum_\tau\nabla p_\theta(\tau)R(\tau)$, which is no longer an expectation and cannot be estimated by sampling.

**Identity:** $\nabla\log f=\frac{\nabla f}{f}$, so $\nabla p_\theta=p_\theta\nabla\log p_\theta$. Substituting:

$$
\nabla_\theta J=\mathbb{E}_{\tau\sim p_\theta}\big[\nabla_\theta\log p_\theta(\tau)\,R(\tau)\big]
$$

Taking the log turns the product into a sum:

$$
\log p_\theta(\tau)=\log\rho_0(s_0)+\sum_t\log\pi_\theta(a_t\mid s_t)+\sum_t\log P(s_{t+1}\mid s_t,a_t)
$$

The first and third terms do not depend on $\theta$, so their gradients are zero:

$$
\boxed{\nabla_\theta J=\mathbb{E}\Big[\sum_t\nabla_\theta\log\pi_\theta(a_t\mid s_t)\cdot R(\tau)\Big]}
$$

**What it proves:** the derivative of a distribution can be rewritten as an expectation, so the gradient **can be estimated by sampling**; the environment transitions $P$ cancel out, so **no environment model is needed**.

> Common mistake: applying $\log$ only to the first factor when splitting the product. The $\log$ applies to every factor, so the result is $\nabla\log\pi$, not $\nabla\pi$.

### 2.2 The Key Lemma

$$
\mathbb{E}_{a\sim\pi_\theta}[\nabla_\theta\log\pi_\theta(a\mid s)]=\sum_a\pi_\theta\frac{\nabla\pi_\theta}{\pi_\theta}=\sum_a\nabla\pi_\theta=\nabla\sum_a\pi_\theta(a\mid s)=\nabla1=0
$$

**What it proves:** the changes in all action probabilities sum to zero.

**Why we need it:** every later argument of the form "this term can be dropped without changing the expected gradient" relies on it.

### 2.3 Baselines

For any $b(s_t)$ that depends only on the state:

$$
\mathbb{E}_{s_t,a_t}\big[\nabla\log\pi_\theta(a_t\mid s_t)\,b(s_t)\big]=\mathbb{E}_{s_t}\Big[b(s_t)\cdot\underbrace{\mathbb{E}_{a_t}[\nabla\log\pi_\theta(a_t\mid s_t)]}_{=0}\Big]=0
$$

**What it proves:** subtracting any function that **does not depend on $a_t$** from the return leaves the expected gradient unchanged, while it can reduce variance substantially. $b=V^\pi(s_t)$ works well because it is the average of $G_t$ given $s_t$, so what remains fluctuates around zero.

> Common mistake: reading $\nabla\log\pi\cdot b$ as $\nabla\log(\pi b)$. $b$ multiplies from the outside; it does not go inside the $\log$.

### 2.4 Causality

Rewards earned before $a_t$ depend only on $a_{<t}$ and $s_{\le t}$; from $a_t$'s point of view they are already fixed and act like a baseline, so their contribution to the expected gradient is zero. Therefore

$$
R(\tau)\ \longrightarrow\ \sum_{k\ge t}\gamma^k r_k=\gamma^t G_t
$$

(The $\gamma^t$ factor is usually dropped in practice.)

**What it proves:** it removes noise that is unrelated to $a_t$. It is a special case of 2.3 and uses the same lemma.

### 2.5 The Policy Gradient with Advantages

Three substitutions, each leaving the expectation unchanged:

$$
R(\tau)\ \xrightarrow{2.4}\ G_t\ \xrightarrow{2.3,\ b=V}\ G_t-V^\pi(s_t)\ \xrightarrow{\text{total exp.}}\ A^\pi(s_t,a_t)
$$

The justification for the last step: $\mathbb{E}[f(X)\,Y]=\mathbb{E}_X\big[f(X)\,\mathbb{E}[Y\mid X]\big]$. Take $X=(s_t,a_t)$; since $f=\nabla\log\pi_\theta(a_t\mid s_t)$ depends only on $X$, it can be pulled out of the inner expectation:

$$
\mathbb{E}[G_t-V^\pi(s_t)\mid s_t,a_t]=Q^\pi(s_t,a_t)-V^\pi(s_t)=A^\pi(s_t,a_t)
$$

$$
\boxed{\nabla_\theta J=\mathbb{E}\Big[\sum_t\nabla_\theta\log\pi_\theta(a_t\mid s_t)\cdot A^\pi(s_t,a_t)\Big]}
$$

**What it proves:** the policy gradient can be written as "gradient of log-probability × advantage". All three forms have the same expectation, with decreasing noise:

| Form | Multiplier | Noise |
|---|---|---|
| REINFORCE | $R(\tau)$ | Highest |
| With baseline | $G_t-V$ (one sample) | Medium |
| Advantage | $A^\pi$ (exact average) | Lowest, but must be estimated |

**When it holds:** the conditioning must include **every** variable the leading factor depends on. If the factor were $\nabla\log\pi_\theta(a_{t+1}\mid s_{t+1})$, we would have to fix $(s_t,a_t,s_{t+1},a_{t+1})$, and the conditional expectation would be $r_t+\gamma Q^\pi(s_{t+1},a_{t+1})$ rather than $Q^\pi(s_t,a_t)$.

> Common mistake: expanding $G_t=r_t+\gamma G_{t+1}$ and concluding that $G_{t+1}$ and $V(s_{t+1})$ do not depend on $a_t$. $s_{t+1}\sim P(\cdot\mid s_t,a_t)$ is determined by the current action. If that conclusion were true, the gradient would depend only on immediate rewards; in CartPole every step's reward is 1, so the gradient would be identically zero and the policy would learn nothing.

---

## Stage 3: Importance Sampling

### 3.1 Motivation

The expectation above requires samples from the **current** policy $\pi_\theta$. After every update the old data becomes stale, and sampling is expensive. We would like to collect a batch with the old policy $\pi_{\theta_{\text{old}}}$ and then take several update steps on it.

### 3.2 The Importance Sampling Identity

$$
\mathbb{E}_{x\sim p}[f(x)]=\sum_x q(x)\frac{p(x)}{q(x)}f(x)=\mathbb{E}_{x\sim q}\Big[\frac{p(x)}{q(x)}f(x)\Big]
$$

Requirement: whenever $p(x)>0$, we must have $q(x)>0$.

### 3.3 Applying It to Actions

Define

$$
\rho_t(\theta)=\frac{\pi_\theta(a_t\mid s_t)}{\pi_{\theta_{\text{old}}}(a_t\mid s_t)}
$$

$$
\mathbb{E}_{a\sim\pi_\theta}[\nabla\log\pi_\theta\cdot A]=\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\rho_t\,\nabla\log\pi_\theta\cdot A]
$$

and since the denominator does not depend on $\theta$:

$$
\rho_t\,\nabla_\theta\log\pi_\theta=\frac{\pi_\theta}{\pi_{\theta_{\text{old}}}}\cdot\frac{\nabla\pi_\theta}{\pi_\theta}=\nabla_\theta\rho_t
$$

**What it proves:** the correction for the action distribution is **exact**, and the gradient can be written as $\nabla\rho_t\cdot A$.

### 3.4 Where $\rho$ Comes From, Intuitively

- **Original purpose: reweighting samples.** If the new policy picks $a$ twice as often as the old one, then $a$ is "under-represented by half" in the old data, so each of its samples should count twice. $\rho$ is a weight **derived** from the need to reuse old data.
- **Side effect: it measures how much the policy changed.** $\rho=1$ means the old and new policies agree on that sample; the further from 1, the bigger the change. PPO exploits this.
- **Historical order:** CPI (2002) introduced $\mathbb{E}[\rho\hat A]$, where $\rho$ was just an importance weight → TRPO (2015) constrained the KL divergence between old and new policies, which needs second-order optimization → PPO (2017) directly limited the $\rho$ already present in the objective, so first-order optimization suffices.
- **Limitation:** $\rho$ only looks at the sampled action, while KL looks at the whole action distribution. So clipping is a rough trust region, and in practice one also monitors `approx_kl`.
- **A small property:** $\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\rho]=\sum_a\pi_\theta(a\mid s)=1$.

### 3.5 The State Visitation Distribution $d^\pi$

Rewrite the trajectory expectation as an expectation over individual states:

$$
\mathbb{E}_\tau\Big[\sum_t\gamma^t g(s_t)\Big]=\sum_s\Big[\sum_t\gamma^t\Pr(s_t=s)\Big]g(s)
$$

After normalization:

$$
d^\pi(s)=(1-\gamma)\sum_{t\ge0}\gamma^t\Pr(s_t=s\mid\pi)
$$

Meaning: when acting with $\pi$, the fraction of time the agent spends in $s$. Hence

$$
\nabla_\theta J=\frac1{1-\gamma}\,\mathbb{E}_{s\sim d^{\pi_\theta}}\,\mathbb{E}_{a\sim\pi_\theta}\big[\nabla\log\pi_\theta(a\mid s)\,A^{\pi_\theta}(s,a)\big]
$$

The constant $\frac1{1-\gamma}$ can be absorbed into the learning rate. **$d^\pi$ depends on the policy**: in CartPole, a good policy concentrates on upright-pole states, while a bad one spreads over tilted states. During training, **the states in the rollout buffer are samples from $d^{\pi_{\theta_{\text{old}}}}$**.

---

## Stage 4: The Surrogate Objective

### 4.1 From the Exact Gradient to a Computable One

| Step | Expression | Exact? |
|---|---|---|
| Exact gradient | $\mathbb{E}_{s\sim d^{\pi_\theta}}\mathbb{E}_{a\sim\pi_\theta}[\nabla\log\pi_\theta\cdot A^{\pi_\theta}]$ | — |
| Importance sampling on actions | $\mathbb{E}_{s\sim d^{\pi_\theta}}\mathbb{E}_{a\sim\pi_{\theta_{\text{old}}}}[\nabla\rho\cdot A^{\pi_\theta}]$ | Exact |
| Swap in the old state distribution | $s\sim d^{\pi_\theta}\to d^{\pi_{\theta_{\text{old}}}}$ | **Approximation ①**: we only have states visited by the old policy |
| Swap in the old advantage | $A^{\pi_\theta}\to\hat A\approx A^{\pi_{\theta_{\text{old}}}}$ | **Approximation ②**: we can only estimate from old returns |

Both approximations are exact at $\theta=\theta_{\text{old}}$; the further $\theta$ moves, the larger the error.

- **When ① breaks:** the new policy may visit states that barely appear in the old data, where we cannot evaluate it.
- **When ② breaks:** $A^{\pi_{\theta_{\text{old}}}}(s,a)$ assumes the old policy is followed afterwards, but the new policy is what actually runs.

**One level deeper (performance difference lemma):**

$$
J(\theta)-J(\theta_{\text{old}})=\frac1{1-\gamma}\,\mathbb{E}_{s\sim d^{\pi_\theta}}\,\mathbb{E}_{a\sim\pi_\theta}\big[A^{\pi_{\theta_{\text{old}}}}(s,a)\big]
$$

This is an exact identity, and the advantage that appears in it is the old policy's. Seen this way, the only real approximation is the state distribution.

Separately, $\hat A$ also carries **estimation noise** (it comes from a single sample). This is unrelated to how far the policy moved, and is reduced by GAE, larger batches, and advantage normalization.

### 4.2 The Surrogate

We need a scalar function to hand to autograd whose gradient equals $\mathbb{E}_{\pi_{\theta_{\text{old}}}}[\nabla\rho_t\cdot\hat A_t]$. Since $\hat A_t$ does not depend on $\theta$ and neither does the sampling distribution, $\nabla$ can be pulled all the way out:

$$
\boxed{L^{\text{CPI}}(\theta)=\mathbb{E}_{s,a\sim\pi_{\theta_{\text{old}}}}\big[\rho_t(\theta)\,\hat A_t\big]}
$$

Batch form: $L=\frac1N\sum_i\rho_i(\theta)\hat A_i$; in code, `loss = -(ratio * adv).mean()` (negated because optimizers minimize).

> Common mistake: writing $L=\nabla\mathbb{E}[\dots]$. $L$ is the function itself; $\nabla L$ is its gradient.

### 4.3 The Gradient Is Correct at $\theta_{\text{old}}$

Take the gradient first, then plug in the value. $\rho_t(\theta_{\text{old}})=1$, but $\nabla\rho_t\ne0$:

$$
\nabla\rho_t\Big|_{\theta_{\text{old}}}=\frac{\nabla\pi_\theta}{\pi_{\theta_{\text{old}}}}\Big|_{\theta_{\text{old}}}=\frac{\nabla\pi_\theta}{\pi_\theta}=\nabla\log\pi_\theta
$$

$$
\nabla L^{\text{CPI}}\Big|_{\theta_{\text{old}}}=\mathbb{E}_{\pi_{\theta_{\text{old}}}}[\nabla\log\pi_\theta\cdot\hat A_t]=\nabla J\Big|_{\theta_{\text{old}}}
$$

**What it proves:** $L^{\text{CPI}}$ and $J$ have the same gradient at $\theta_{\text{old}}$, so locally they slope in the same direction; but this guarantee only holds nearby.

### 4.4 The Problem with Unconstrained Optimization

A single sample's contribution $\rho\hat A$ is a straight line in $\rho$:

- $\hat A>0$: the optimizer keeps pushing $\rho$ up, with essentially no ceiling ($\rho\le1/\pi_{\theta_{\text{old}}}$);
- $\hat A<0$: it pushes $\rho$ toward 0.

Consequences: overfitting to the noise in $\hat A$; both approximations break down, so $L$ goes up while $J$ goes down; the next batch is collected by a policy that has already gotten worse, and training can collapse.

**Conclusion:** we must limit how far $\rho$ can move away from 1 in each round.

---

## Stage 5: Clipping

### 5.1 The Objective

$$
\boxed{L^{\text{CLIP}}(\theta)=\mathbb{E}_{s,a\sim\pi_{\theta_{\text{old}}}}\Big[\min\big(\rho_t\hat A_t,\ \text{clip}(\rho_t,1-\epsilon,1+\epsilon)\,\hat A_t\big)\Big]}
$$

### 5.2 A Numerical Check ($\epsilon=0.2$)

$\hat A=+2$:

| $\rho$ | $\rho\hat A$ | $\text{clip}(\rho)\hat A$ | $\min$ |
|---|---|---|---|
| 0.5 | 1 | 1.6 | 1 |
| 1.0 | 2 | 2 | 2 |
| 1.5 | 3 | 2.4 | 2.4 |

$\hat A=-2$:

| $\rho$ | $\rho\hat A$ | $\text{clip}(\rho)\hat A$ | $\min$ |
|---|---|---|---|
| 0.5 | −1 | −1.6 | −1.6 |
| 1.0 | −2 | −2 | −2 |
| 1.5 | −3 | −2.4 | −3 |

### 5.3 Shape of the Curve

- $\hat A>0$: for $\rho\le1+\epsilon$ it is a line with slope $\hat A$; beyond that it is flat.
- $\hat A<0$: for $\rho<1-\epsilon$ it is flat (at height $(1-\epsilon)\hat A$); beyond that it is a line with slope $\hat A$, heading down without bound.

> Common mistake: designing the function to drop to 0 at $1+\epsilon$. That penalizes points already outside the interval and makes the function discontinuous. The right choice is to stay flat, so the gradient is 0.

### 5.4 The Gradient Table

| $\partial/\partial\rho$ | $\rho<1-\epsilon$ | $[1-\epsilon,1+\epsilon]$ | $\rho>1+\epsilon$ |
|---|---|---|---|
| $\hat A>0$ | $\hat A$ (can be corrected) | $\hat A$ | $0$ (clipped) |
| $\hat A<0$ | $0$ (clipped) | $\hat A$ | $\hat A$ (can be corrected) |

**What it proves:** clipping only cuts off the side where the update has gone **too far in the direction of improvement**; if $\rho$ has been pushed to the wrong side, the gradient is kept so it can be pulled back.

### 5.5 Why Take the $\min$

Using only $\text{clip}(\rho)\hat A$, both ends are flat. Take $\hat A>0$, $\rho=0.5$: with clip alone the gradient is 0 and the mistake cannot be corrected; with the $\min$ we land on $\rho\hat A$, the gradient is $\hat A$, and it gets pulled back.

### 5.6 A Pessimistic Lower Bound

Since $\min(x,y)\le x$,

$$
L^{\text{CLIP}}(\theta)\le L^{\text{CPI}}(\theta),\quad\text{with equality when }\rho\in[1-\epsilon,1+\epsilon]
$$

**What it proves:** $L^{\text{CLIP}}$ is a **pessimistic lower bound** on $L^{\text{CPI}}$: gains beyond the trust region are not counted, while losses are counted in full. At $\theta_{\text{old}}$ the two have the same value and gradient, so the conclusion of 4.3 still holds.

---

## Summary

| Stage | Result | Problem it solves |
|---|---|---|
| 0 | $V,Q,A,\delta_t$ | Defines "good" and "bad"; one transition suffices to estimate the advantage |
| 1 | $J(\theta)=\mathbb{E}_\tau[R]$ | Measures a policy with one scalar |
| 2 | $\nabla J=\mathbb{E}[\nabla\log\pi\cdot A]$ | A model-free gradient; baselines reduce variance |
| 3 | $\rho_t$ | Estimates the new policy's gradient from old data |
| 4 | $L^{\text{CPI}}=\mathbb{E}[\rho\hat A]$ | An objective autograd can use, but only trustworthy locally |
| 5 | clip + min | Limits each round's update; a pessimistic lower bound keeps it stable |

### Mapping to Code (a typical PyTorch implementation)

| Code | Formula |
|---|---|
| `delta = reward + gamma * next_value - value` | $\delta_t$ (0.5) |
| `gae = delta + gamma * lam * ... * gae` | GAE estimate of $\hat A_t$ |
| Advantage normalization `(adv - mean) / std` | Subtracting a constant baseline leaves the expected gradient unchanged (2.3) |
| `ratio = exp(new_log_probs - old_log_probs)` | $\rho_t$ (3.3) |
| `surr1 = ratio * adv`, `surr2 = clamp(ratio, 1-eps, 1+eps) * adv` | The two terms inside the min (5.1) |
| `approx_kl` | Monitors the actual KL beyond what $\rho$ captures (3.4) |
| `clip_fraction` | Fraction of samples landing in the "0" cells of the gradient table (5.4) |
| `value_loss` | Trains $V$, since $\delta_t$ needs an accurate $V$ |
| `entropy` | Encourages exploration (not part of the policy-gradient derivation) |

### Further Derivations

1. **GAE:** show that $\hat A_t^{\text{GAE}}=\sum_l(\gamma\lambda)^l\delta_{t+l}$ reduces to $\delta_t$ at $\lambda=0$ and to $G_t-V(s_t)$ at $\lambda=1$, and explain the bias–variance trade-off.
2. **Performance difference lemma:** prove the identity in 4.1, which explains why using the old policy's advantage is justified.
