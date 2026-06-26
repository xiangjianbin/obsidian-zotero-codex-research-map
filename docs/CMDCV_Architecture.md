# CM-DCV 模型架构详细技术文档

> **Cross-Modal Depth-Stratified Sensitivity Volume — 跨模态深度分层灵敏度代价体**
> **当前版本**: v11（信息瓶颈解耦耦合）
> **代码位置**: `MR03_mainline/code/cm_dcv_v9_dualtrack.py`
> **基线代码**: `MR03_mainline/code/cm_dcv_feasibility.py` (v7 headline)
> **训练脚本**: `MR03_mainline/code/train_v2.py`
> **状态**: 已实跑验证，v11 IB 在 K=32 全量数据上减缓"先峰后崩" 3.5x

---

## 目录

1. [定位与科学命题](#1-定位与科学命题)
2. [整体架构](#2-整体架构)
3. [模块逐一详解](#3-模块逐一详解)
   - 3.1 [PhysicsNormDualEncoder — 物理归一化双分支编码器](#31-physicsnormdualencoder--物理归一化双分支编码器)
   - 3.2 [StructureEvidence — 半显式结构证据头](#32-structureevidence--半显式结构证据头)
   - 3.3 [DepthStratifiedSensitivityVolume — 深度分层灵敏度代价体](#33-depthstratifiedsensitivityvolume--深度分层灵敏度代价体)
   - 3.4 [CouplingV5 — 物理耦合（峰层门控 + 1D OT）](#34-couplingv5--物理耦合峰层门控--1d-ot)
   - 3.5 [V3CouplingHead — 学习残差耦合](#35-v3couplinghead--学习残差耦合)
   - 3.6 [EntropyBottleneck — 信息瓶颈 (v11 新增)](#36-entropybottleneck--信息瓶颈-v11-新增)
   - 3.7 [CMDCV_v11 主模型组装](#37-cmdcv_v11-主模型组装)
   - 3.8 [MixtureDecoder3D — B'-lite 物性解码器](#38-mixturedecoder3d--b-lite-物性解码器)
4. [张量形状流](#4-张量形状流)
5. [损失函数体系](#5-损失函数体系)
6. [训练策略](#6-训练策略)
7. [版本演进与关键设计决策](#7-版本演进与关键设计决策)
8. [与已有方法的机制区分](#8-与已有方法的机制区分)
9. [可证伪框架 (Bets)](#9-可证伪框架-bets)
10. [附录: 符号表与文件索引](#10-附录符号表与文件索引)

---

## 1. 定位与科学命题

### 1.1 问题背景

重磁联合反演是地球物理中的经典逆问题。地表观测的重力异常 $g$ 和磁异常 $m$ 都来源于地下同一组密度 $\rho$ 和磁化率 $\kappa$ 的三维分布。但反演具有严重的**非唯一性（多解性）**，尤其是**深度分辨率极差**——浅源和深源可能产生几乎相同的地面观测。

### 1.2 核心思想

**将"深度"从隐含的体素坐标变成显式的竞争维度。**

1. 沿深度方向枚举 $K$ 个假设（depth hypotheses），每个假设代表"异常源主要位于该深度层"
2. 对每个深度假设，分别计算重力单模态代价 $\tilde{C}_g$、磁力单模态代价 $\tilde{C}_m$ 和跨模态结构耦合代价 $C_{couple}$
3. 通过 softmax 将代价转换为深度后验概率 $p_g/p_m/p_{gm}$
4. 用信息增益 $IG = \min\{H[p_g], H[p_m]\} - H[p_{gm}]$ 量化跨模态联合对深度不确定性的减少

### 1.3 科学命题

> **条件跨模态深度信息增益命题**：在重力-磁力观测对共享地下结构的有效域内，跨模态联合反演的深度后验熵 $H[p_{gm}]$ 显著小于各单模态后验熵的较小者 $\min(H[p_g], H[p_m])$，即 $IG > 0$ 且通过 SESOI 效应量门控。

**硬边界**（承 INH-1 零空间定律）：
- 不主张解决零空间
- 不主张全域唯一
- $p(z)$ 是残留多解的诚实表达
- IG 是"条件信息增益"非"消除非唯一"

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CM-DCV v11 完整架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                                   │
│  │   g      │  │   m      │  │ ξ_g, ξ_m │                                   │
│  │(B,1,H,W) │  │(B,1,H,W) │  │ (B,meta) │                                   │
│  └────┬─────┘  └────┬─────┘  └─────┬────┘                                   │
│       │             │              │                                          │
│       ▼             ▼              ▼                                          │
│  ┌────────────────────────────────────────────────┐                          │
│  │        PhysicsNormDualEncoder (§3.1)            │                          │
│  │  E_g: Conv→GN→GELU→Conv(stride2)→GELU→Conv→GELU │                         │
│  │  E_m: 同上（独立权重）                            │                         │
│  │  ξ 注入: meta_linear(feat) + spatial_broadcast    │                         │
│  │  ⚠️ 双分支独立，不在观测层融合                       │                          │
│  └──────────────┬───────────────────┬───────────────┘                         │
│                 │                   │                                           │
│          f_g (B,C,h,w)       f_m (B,C,h,w)                                    │
│         ┌────┴────┐          ┌────┴────┐                                      │
│         ▼         ▼          ▼         ▼                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│  │DepthCost │ │StructEv. │ │DepthCost │ │StructEv. │                          │
│  │  (§3.3)  │ │  (§3.2)  │ │  (§3.3)  │ │  (§3.2)  │                          │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                          │
│       │            │            │            │                                 │
│  C̃_g(B,K)    Sg(z,geo)  C̃_m(B,K)    Sm(z,geo)                               │
│       │            │            │            │                                 │
│       └────────────┼────────────┼────────────┘                                 │
│                    ▼            ▼                                              │
│           ┌─────────────────────────────┐                                     │
│           │  V5 物理耦合 (§3.4)          │  ← 用原始尖锐 Sg_z/Sm_z             │
│           │  峰层门控 → Sinkhorn OT      │                                     │
│           │  → R_V5 (B,K)               │                                     │
│           └──────────────┬──────────────┘                                     │
│                          │                                                    │
│           ┌──────────────┴──────────────┐                                     │
│           │  EntropyBottleneck (§3.6) ★  │  ← v11 新增                        │
│           │  Sg_z → Sg_z_ib (平滑版)     │                                     │
│           │  Sm_z → Sm_z_ib (平滑版)     │                                     │
│           │  保证 H ≥ H_min = 60%·logK   │                                     │
│           └──────────────┬──────────────┘                                     │
│                          │                                                    │
│           ┌──────────────┴──────────────┐                                     │
│           │  V3 学习残差 (§3.5)          │  ← 用平滑版 Sg_z_ib/Sm_z_ib        │
│           │  Conv-MLP → R_θ (B,K)       │                                     │
│           └──────────────┬──────────────┘                                     │
│                          │                                                    │
│              R = (1-α)·R_V5 + α·R_θ    (α=0.1, V5主导)                       │
│                          │                                                    │
│              C_couple = -λ · R          (负 reward, λ 可学)                     │
│                          │                                                    │
│              C_gm = C̃_g + C̃_m + C_couple  (B,K)                             │
│                          │                                                    │
│                          ▼ T = softplus(T_param)+0.1                          │
│              ┌───────────────────────────────────┐                             │
│              │     校准后验 + 信息增益 (§3.7)      │                             │
│              │  p_g  = softmax(-C̃_g / T)  (B,K)  │                             │
│              │  p_m  = softmax(-C̃_m / T)  (B,K)  │                             │
│              │  p_gm = softmax(-C_gm / T)  (B,K)  │                             │
│              │                                    │                             │
│              │  IG = min(H_g, H_m) - H_gm  (B,)  │                             │
│              │  conflict = MSE+Wasserstein(p_g,p_m)│                             │
│              │  reliability, OOD = sigmoid(aux)    │                             │
│              └───────────────────┬───────────────┘                             │
│                                  │                                             │
│               ┌──────────────────┴──────────────────┐                          │
│               │  输出: {p_g, p_m, p_gm, IG,          │                          │
│               │        conflict, reliability, OOD,   │                          │
│               │        C_base, C_couple, R,          │                          │
│               │        Sg_z, Sm_z,                   │                          │
│               │        Sg_z_ib, Sm_z_ib, ★           │  ← v11 新增              │
│               │        H_sg_ib, H_sm_ib ★}           │                          │
│               └────────────────────────────────────┘                          │
│                                  │ (可选)                                       │
│                                  ▼                                             │
│               ┌──────────────────────────────────────┐                          │
│               │  MixtureDecoder3D (§3.8)              │                          │
│               │  TopM(p_gm) → M个深度 mode            │                          │
│               │  共享权重 3D decoder + mode embedding  │                          │
│               │  → ρ_mixture / κ_mixture              │                          │
│               └──────────────────────────────────────┘                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 模块逐一详解

### 3.1 PhysicsNormDualEncoder — 物理归一化双分支编码器

**继承自**: `CMDCV` (v7 基线类)

#### 输入

| 张量 | 形状 | 含义 |
|------|------|------|
| `g` | `(B, 1, H, W)` | 地表重力异常 $g_z$（mGal 量级，×5 缩放） |
| `m` | `(B, 1, H, W)` | 地表磁异常 TMI（nT 量级，×20 缩放） |
| `ξ_g` | `(B, meta_dim)` | 重力物理元数据 |
| `ξ_m` | `(B, meta_dim)` | 磁力物理元数据 |

#### 元数据 ξ 的 8+ 维定义

| 维度 | 含义 | 作用 |
|------|------|------|
| 0 | SNR 归一化（重力） | 数据质量先验 |
| 1 | SNR 归一化（磁力） | 数据质量先验 |
| 2-7 | Template one-hot / 深度层编码 | 构造类型 + 深度先验 |

**核心目的**: ξ 注入防止模型通过观测幅值 shortcut 伪造深度信号。

#### CNN 骨干

```
输入 (B, 1, H, W)
  → Conv2d(1→c, 3×3, pad=1) → GroupNorm → GELU
  → Conv2d(c→c, 3×3, stride=2, pad=1) → GELU    ← 下采样 2×
  → Conv2d(c→c, 3×3, pad=1) → GELU
输出 (B, c, h, w)  其中 h=H/2, w=W/2
```

默认 `c=64`（全量训练），demo 时 `c=32`。

#### 关键设计约束

1. **双分支独立**: $f_g$ 不看 $m$，$f_m$不看 $g$
2. **不在观测层 early fusion**（INH D004 已淘汰）
3. **ξ 加法注入**: `f = CNN(x) + Linear(ξ)[..., None, None]`

---

### 3.2 StructureEvidence — 半显式结构证据头

**继承自**: `CMDCV.S`

#### 定义

```python
class StructureEvidence(nn.Module):
    def __init__(self, in_ch, K=16):
        self.z_head = nn.Conv2d(in_ch, K, 1)    # 深度边际头
        self.geom = nn.Conv2d(in_ch, 3, 1)      # 几何属性头

    def forward(self, f):
        Sz  = softmax(z_head(f).spatial_mean, dim=-1)   # (B,K) 深度边际分布
        geo = geom(f).spatial_mean                        # (B,3) Ω/b/c
        return Sz, geo
```

#### 输出

| 符号 | 形状 | 含义 | 用途 |
|------|------|------|------|
| $S^z$ | `(B, K)` | 深度边际分布（softmax，和=1） | C_couple 的深度证据 + L_struct 监督 |
| $S^\Omega$ | `(B,)` | 源体支撑域概率 | CouplingCost Dice 分量 |
| $S^b$ | `(B,)` | 边界概率 | 辅助几何信息 |
| $S^c$ | `(B,)` | 源中心 | 辅助几何信息 |

#### 为什么"半显式"?

| 方案 | 优点 | 缺陷 |
|------|------|------|
| 全隐式（端到端） | 无需标签 | C_couple 退化为特征对齐 |
| 全显式（完整标注） | 物理明确 | 实测无法获得完整标签 |
| **半显式（本方案）** | 合成可监督；实测弱监督 | 折中 |

**关键**: $S_g$ 完全由 $f_g$ 产生，$S_m$ 完全由 $f_m$ 产生 —— 各模态独立推断。

---

### 3.3 DepthStratifiedSensitivityVolume — 深度分层灵敏度代价体

**继承自**: `CMDCV.cost`（核心创新模块）

#### 计算流程（4 步）

**Step 1: 深度假设投影**
$$Q_g = \text{proj}_g(f_g) \in \mathbb{R}^{B \times K \times h \times w}$$
$$Q_m = \text{proj}_m(f_m) \in \mathbb{R}^{B \times K \times h \times w}$$

每个通道 $k \in \{0,...,K-1\}$ 代表一个深度假设 $z_k$。$Q_g[:,k]$ 是"如果异常源主要在第 $k$ 层深度"的重力证据图。

**Step 2: Footprint 非局部平滑**

```python
for k in range(K):
    r = 1 + k // 4          # 深度越深，平滑半径越大
    Q_smooth[:,k] = avg_pool2d(pad(Q[:,k], r), kernel=2r+1)
```

| 深度层 k | 平滑半径 r | 物理解释 |
|----------|-----------|----------|
| 0-3 | 1 | 浅部源：footprint 尖锐 |
| 4-7 | 2 | 中浅部 |
| 8-11 | 3 | 中深部 |
| 12-15 (K=16) | 4 | 深部源：footprint 宽缓 |

符合位场物理：**浅部源的地面异常窄而陡，深部源的地面异常宽而缓**。
使用 `torch.stack` out-of-place 保 autograd。

**Step 3: 深度代价计算**
$$C_g(k) = -\text{spatial\_mean}(Q_g[:, k])$$
$$C_m(k) = -\text{spatial\_mean}(Q_m[:, k])$$

直觉：证据越高 → 代价越低 → softmax 后概率高。

**Step 4: 灵敏度归一化噪声白化（双重计算修正）**

$$\tilde{C}_g = \frac{C_g - \mu}{\sigma}, \quad \tilde{C}_m = \frac{C_m - \mu}{\sigma}$$

其中 $\mu, \sigma$ 是 `register_buffer` 预注册的固定值（linspace，非样本特异）。

**为什么不用乘法衰减？**

| 方法 | 公式 | 问题 |
|------|------|------|
| ~~乘法衰减~~ | $C(z) \cdot e^{-\alpha z}$ | 与正演核 $K(z)$ 中的衰减**双重计算**→浅部偏置 |
| **归一化白化（本方案）** | $(C - \mu) / \sigma$ | 无额外深度偏好 |

这是 vFinal 的 **fatal 修复**（ChatGPT P02 发现）。

---

### 3.4 CouplingV5 — 物理耦合（峰层门控 + 1D OT）

**v9 引入，v10 修复，v11 继承**

#### 核心机制

$$C_{couple}(k) = -\lambda \cdot R(k), \quad R(k) = (1-\alpha) \cdot R_{V5}(k) + \alpha \cdot R_\theta(k)$$

**关键设计：负 reward**（一致层降 cost，非加 cost —— 修 v8-V1 方向反）。

#### V5 计算流程（纯物理，无可学参数）

```
输入: Sg_z (B,K), Sm_z (B,K)

1. 峰层门控 peak_dist():
   - 保留 top-M 局部峰且 > rel_thr·max 的层
   - 归一化成峰分布 q^P (B,K)
   - 动态 topM: K=16→3, K=32→5 (~15-19% of bins)

2. Sinkhorn 最优传输 sinkhorn_1d(qg_p, qm_p, D):
   - D(i,j) = |z_i - z_j|  (深度代价矩阵)
   - blur=0.05, n_iter=30
   - 输出: π (B,K,K) 传输 plan

3. 短距配对门控:
   short_gate = exp(-D/σ_w) · sigmoid((δ_z - D)/τ_w)
   a = π ⊙ short_gate  (短距配对权重)

4. 落点映射:
   R = einsum('bij,kij->bk', a, bump)   (B,K)
   bump: 高斯落点核，将配对能量散布到相邻层

5. Confidence gate:
   g_conf = sigmoid((pk_g - τ_p)/τ_c) · sigmoid((pk_m - τ_p)/τ_c)
           · sigmoid((δ_w - W1)/τ_w) · valid_peak
   R_V5 = clamp(g_conf · R, 0, 1)
```

#### V5 参数（K 自适应，v10 修复）

| 参数 | K=16 | K=32 | K=60 | 含义 |
|------|------|------|------|------|
| τ_p (peakness) | 0.081 | 0.096 | 0.12 | 峰锐度阈值 |
| δ_w (max W1) | 0.40 | 0.48 | 0.55 | 最大允许 Wasserstein 距离 |
| σ_w (short range) | 0.059 | 0.052 | 0.045 | 短距门控宽度 |
| δ_z (depth gate) | 0.074 | 0.088 | 0.10 | 深度距离阈值 |
| topM | 3 | 5 | 7 | 峰层保留数 |

#### v10 对 V5 的修复

1. **去 detach**: v9 中 V5 用 `Sg_z.detach()`（防 backprop 污染），v10 改为直接用 `Sg_z`（让 V5 也参与 z_head 训练）
2. **紧距离门控**: 所有阈值按 $\log(K)/\log(60)$ 自动缩放
3. **gate_disable 选项**: bootstrap 阶段可跳过 confidence gate

---

### 3.5 V3CouplingHead — 学习残差耦合

**v9 引入，v10 修复**

#### 定义

```python
class V3CouplingHead_v10(nn.Module):
    def __init__(self, K=16, hidden=16):
        # v10: 只交互特征（砍单边防 shortcut）
        self.net = nn.Sequential(
            Conv1d(2, hidden, 3, padding=1), GELU(), GroupNorm(1, hidden),
            Conv1d(hidden, hidden, 3, padding=1), GELU(),
            Conv1d(hidden, 1, 1))
        # K 自适应 bias: K=16→-2.6(R≈0.069), K=60→-4.0(R≈0.018)
        init_bias = -4.0 + 1.5 * (1 - K / 60)
        nn.init.constant_(self.net[-1].bias, init_bias)
        self.logit_lambda = nn.Parameter(torch.tensor(0.0))

    def forward(self, qg, qm, lam_max=3.0):
        x = stack([qg * qm, (qg - qm).abs()], dim=1)  # (B,2,K) 纯交互
        R = sigmoid(net(x)).squeeze(1)                  # (B,K)
        lam = lam_max * sigmoid(logit_lambda)
        return R, lam
```

#### v9 → v10 变更

| | v9 | v10 |
|---|-----|-----|
| 输入特征 | `[qg, qm, qg*qm, \|qg-qm\|]` (4 ch) | `[qg*qm, \|qg-qm\|]` (2 ch) |
| bias 初始化 | -4.0 (固定) | K 自适应 (-2.6 for K=16) |
| 目的 | 提供丰富特征 | **砍单边防 shortcut** |

**设计哲学**: bias=-4 → 初始 $R_\theta \approx \sigma(-4) \approx 0.018$ ≈ 0，开局不扰动 $C_g + C_m$。$\lambda$ 从 0.5 起步慢慢增长。

---

### 3.6 EntropyBottleneck — 信息瓶颈 (v11 新增)

**★ v11 核心创新模块**

#### 动机

"先峰后崩"根因：
- $L_{depth}$ 要求 $S_g_z$ 尖锐（低熵）→ 用于精确深度预测
- $C_{couple}$ 需要 $S_g_z$ 平坦（高熵）→ 为耦合留空间
- **目标冲突** → IG 必然在训练后期崩塌

#### 解决方案: 双路径解耦

```
路径 A (尖锐): Sg_z/Sm_z → L_struct 监督 + V5 峰检测 → 深度精度优先
路径 B (平滑): Sg_z_ib/Sm_z_ib → V3 学習耦合 → 保证最小熵 (H ≥ 60%·logK)
```

#### 定义

```python
class EntropyBottleneck(nn.Module):
    """可学习的信息瓶颈：对输入分布做温度重采样，保证输出熵 ≥ H_min"""

    def __init__(self, K=16, H_min_ratio=0.6, init_T=2.0):
        self.K = K
        self.logK = math.log(K)
        self.H_min = H_min_ratio * self.logK   # 硬约束下界
        # 可学习温度（log 空间，始终 > 0）
        self.T_ib = nn.Parameter(torch.tensor(math.log(init_T)))

    def forward(self, q_raw):
        # 温度缩放: 高温 → 更均匀
        T = exp(T_ib).clamp(min=1.01, max=10.0)
        logit = log(q_raw + ε) / T
        q_ib = softmax(logit, dim=-1)           # (B,K)

        # ★ 硬约束: 熵低于 H_min → 向 uniform 混合
        H_q = entropy(q_ib)                      # (B,)
        uniform = ones_like(q_ib) / K
        deficit = relu(H_min - H_q)              # 熵缺口
        mix = deficit / (logK - H_min + ε)        # 归一化混合系数
        q_ib = (1 - mix) * q_ib + mix * uniform

        return q_ib
```

#### 数学原理

对分布 $q$ 做温度重采样：

$$q_{ib}(k) \propto q(k)^{1/T_{ib}}, \quad T_{ib} > 1$$

当 $T_{ib} > 1$ 时，分布变得更平坦（更高熵）。$T_{ib}$ 可学习，初始值 2.0。

**硬约束**: 如果重采样后熵仍低于 $H_{min}$，向 uniform 分布混合：

$$q_{ib} = (1 - \beta) \cdot q_{ib} + \beta \cdot \frac{1}{K}\mathbf{1}$$

其中 $\beta = \frac{\max(0, H_{min} - H(q_{ib}))}{\log K - H_{min}}$。

#### 参数开销

| 组件 | 参数量 |
|------|--------|
| ib_g (重力) | 1 ($T_{ib}$) |
| ib_m (磁力) | 1 ($T_{ib}$) |
| **总计** | **2** |

零推理开销（仅前向传播时多一次 softmax + clamp）。

---

### 3.7 CMDCV_v11 主模型组装

#### 完整前向传播

```python
class CMDCV_v11(CMDCV):
    def forward(self, g, m, xi_g, xi_m, use_couple=True):
        # 1. 双分支编码
        fg, fm = self.enc(g, m, xi_g, xi_m)         # (B,c,h,w) ×2

        # 2. 结构证据（路径 A: 原始/尖锐）
        Sg_z, Sg_geo = self.S(fg)                    # (B,K), (B,3)
        Sm_z, Sm_geo = self.S(fm)                    # (B,K), (B,3)

        # 3. 深度分层代价
        Cg, Cm = self.cost(fg, fm)                   # (B,K) ×2
        C_base = Cg + Cm                              # (B,K)

        # 4. 单模态后验
        T_val = self._T()                            # 校准温度
        pg = softmax(-Cg / T_val, -1)                # (B,K)
        pm = softmax(-Cm / T_val, -1)                # (B,K)

        # ★ 5. 信息瓶颈（路径 B: 平滑版）
        Sg_z_ib = self.ib_g(Sg_z)                    # (B,K), H ≥ H_min
        Sm_z_ib = self.ib_m(Sm_z)                    # (B,K), H ≥ H_min

        # 6. 双轨耦合
        R_v5, _ = self.v5(Sg_z, Sm_z)               # V5 用尖锐版（峰检测需要清晰峰）
        R_theta, lam = self.v3(Sg_z_ib, Sm_z_ib)    # ★ V3 用平滑版（保证最小熵）
        R = clamp((1-α) * R_v5 + α * R_theta, 0, 1)
        C_couple = -lam * R                           # 负 reward

        # 7. 联合后验
        C_gm = C_base + C_couple if use_couple else C_base
        pgm = softmax(-C_gm / T_val, -1)             # (B,K)

        # 8. 信息增益
        Hg = entropy(pg); Hm = entropy(pm); Hgm = entropy(pgm)
        IG = min(Hg, Hm) - Hgm                       # ★ 核心输出 (可为负!)

        # 9. 辅助输出
        conflict = 0.5*((pg-pm)**2).sum(-1) + wasserstein_1d(pg, pm)
        feat = cat([pg, pm, pgm, Sg_geo, Sm_geo], -1)
        aux = self.aux(feat)
        reliability = aux[:,0].sigmoid()
        OOD = aux[:,1].sigmoid()

        return { ..., Sg_z_ib, Sm_z_ib, H_sg_ib, H_sm_ib }
```

#### 输出字典完整列表

| 键 | 形状 | 含义 | v11 新增? |
|----|------|------|-----------|
| `p_g` | `(B, K)` | 重力单模态深度后验 | |
| `p_m` | `(B, K)` | 磁力单模态深度后验 | |
| `p_gm` | `(B, K)` | 联合深度后验 | |
| `IG` | `(B,)` | 跨模态信息增益（**可为负**） | |
| `conflict` | `(B,)` | 模态冲突度 | |
| `reliability` | `(B,)` | 可信度 ∈[0,1] | |
| `OOD` | `(B,)` | 分布外风险 ∈[0,1] | |
| `C_base` | `(B, K)` | 基线代价 (无耦合) | |
| `C_couple` | `(B, K)` | 耦合代价 (**≤0**, negative reward) | |
| `R` | `(B, K)` | 总耦合 reward ∈[0,1] | |
| `R_v5` | `(B, K)` | V5 物理 reward | |
| `R_theta` | `(B, K)` | V3 学习 reward | |
| `lam` | `(B,)` | 耦合强度 λ | |
| `Sg_z` | `(B, K)` | 重力深度边际（尖锐版） | |
| `Sm_z` | `(B, K)` | 磁力深度边际（尖锐版） | |
| `Sg_z_ib` | `(B, K)` | **重力深度边际（IB 平滑版）** | **YES** |
| `Sm_z_ib` | `(B, K)` | **磁力深度边际（IB 平滑版）** | **YES** |
| `H_sg_ib` | `(B,)` | **IB 后重力熵** | **YES** |
| `H_sm_ib` | `(B,)` | **IB 后磁力熵** | **YES** |

---

### 3.8 MixtureDecoder3D — B'-lite 物性解码器

**v7 新增，附加输出，fail 不拖垮 headline**

#### TopM 多峰机制

```
p_gm (B,K) → topk(M=2) → idx (B,M), pi (B,M)
  ↓
对每个 mode m:
  cond = feat_proj(pool) + mode_emb[idx[:,m]]
  h = dec(cond)  (共享 MLP)
  vox = to_voxel(h).view(B, 4, dz, dy,dx)
  ρ̂_r = 800·tanh(vox[:,0])     # ρ∈[-800,800]
  κ̂_r = 0.3·sigmoid(vox[:,2])  # κ∈[0,0.3]
  ↓
ρ_mixture = Σ π_r · ρ̂_r   (加权组合, 非单 MAP)
κ_mixture = Σ π_r · κ̂_r
```

#### 7 项修复落地

| 编号 | 内容 | 代码体现 |
|------|------|---------|
| D2 | 全连接特征输入 | feat_proj 接收池化特征 |
| D3 | 物性范围硬约束 | tanh(·800) / sigmoid(·0.3) |
| D4 | L_fwd 仅评估不作损失 | 正演残差不进入主损失 |
| D5 | TopM mixture 保多峰 | topk(pgm, M) + 加权组合 |
| R1 | 编码器冻结（stop_gradient） | 外层控制 |
| D8 | 每 mode EDL logvar | 输出 (μ, logvar) |
| D6 | profile likelihood 一致性 | 共享 s 参数化 |

---

## 4. 张量形状流

以 **全量配置 B=16, K=32, H=W=64, c=64** 为例：

```
阶段                      形状                          说明
═════════════════════════════════════════════════════════════════════
输入
  g                       (16, 1, 64, 64)               重力异常
  m                       (16, 1, 64, 64)               磁异常
  ξ_g, ξ_m                (16, 10) × 2                 物理元数据

编码后
  f_g, f_m                (16, 64, 32, 32)              下采样 2×, 各模独立

结构证据 (路径 A: 尖锐)
  S_g_z, S_m_z            (16, 32)                     深度边际 (softmax, 和=1)
  S_g_geo, S_m_geo        (16, 3)                      支撑/边界/中心

信息瓶颈 (路径 B: 平滑) ★v11
  S_g_z_ib, S_m_z_ib      (16, 32)                     IB 平滑版 (H ≥ 60%·logK)

深度代价体
  Q_g, Q_m                (16, 32, 32, 32)             深度假设证据
  C̃_g, C̃_m               (16, 32)                     灵敏度归一化代价

双轨耦合
  R_v5                    (16, 32)                     V5 物理 reward
  R_theta                 (16, 32)                     V3 学习 reward
  R                       (16, 32)                     加权组合 ∈[0,1]
  λ                       (16,)                       耦合强度 (可学)
  C_couple                (16, 32)                     ≤0 (negative reward)

联合代价 & 后验
  C_base                  (16, 32)                     C̃_g + C̃_m
  C_gm                    (16, 32)                     C_base + C_couple
  p_g, p_m, p_gm          (16, 32)                     概率分布 (softmax, 和=1)

信息增益头
  IG                      (16,)                        核心输出 (可为负!)
  conflict                (16,)                        ≥ 0
  reliability             (16,)                        ∈[0,1]
  OOD                     (16,)                        ∈[0,1]

IB 统计量 ★v11
  H_sg_ib                 (16,)                        IB 后重力熵
  H_sm_ib                 (16,)                        IB 后磁力熵

参数量估算:
  Encoder (c=64):         ~200K
  StructureEvidence:      ~3K
  DepthCost:              ~3K
  V5 (无参数):            0
  V3 (hidden=16):         ~0.5K
  IB (×2):                2
  Aux head:               ~3K
  ───────────────────────────────
  Headline 总计:          ~210K

  Decoder (可选):          ~550K
  ═════════════════════════════
  全量总计:               ~760K (c=64, K=32)
  U-Net 骨干替换后:       ~10⁷⁶~10⁷⁷
```

---

## 5. 损失函数体系

### 5.1 总损失

$$\mathcal{L}_{total} = \mathcal{L}_{depth} + w_{sel} \cdot \mathcal{L}_{sel} + w_{struct} \cdot \mathcal{L}_{struct} + w_{shuf} \cdot \mathcal{L}_{shuf} + w_{inv} \cdot \mathcal{L}_{inv} + \mathcal{L}_{ig\_reg}$$

### 5.2 各分量详解

#### $\mathcal{L}_{depth}$ — 深度分布监督（主损失）

$$\mathcal{L}_{depth} = \underbrace{-\sum_k q^{soft}_k \log p_{gm,k}}_{\text{NLL}} + \underbrace{\sum_k (p_{gm,k} - q^{soft}_k)^2}_{\text{Brier Score}}$$

$q^{soft}$: 来自高斯核 Blur_R 卷积真值深度支撑的软标签。

#### $\mathcal{L}_{sel}$ — Selectivity Loss（per-domain Δh）

$$\Delta h = \frac{H[p_{drop}] - H[p_{full}]}{\log K}$$

- valid 域: 鼓励 $\Delta h > 0.03$（耦合锐化）
- noncoloc 域: 约束 $\Delta h \in [0.005, 0.02]$（弱耦合）
- zerocouple 域: 强制 $|\Delta h| < 0.01$（零耦合）
- 域排序: valid > noncoloc > zerocouple

**关键**: 使用 `C_base.detach()` 让 selectivity 只训 coupling 分支。

#### $\mathcal{L}_{struct}$ — 结构监督（z_head 监督）

$$\mathcal{L}_{struct} = CE(S_g^z, z_{true}) + CE(S_m^z, z_{true})$$

让结构估计变准 → V5 峰配对有意义。带 curriculum 控制（前 30 epoch 渐增）。

#### $\mathcal{L}_{shuf}$ — Shuffled-pair 负控

每步随机错配 g-m 对，压制 $\Delta h_{shuf} \to 0$。

**杀 shortcut**: 如果耦合学的是"结构存在"而非真跨模态，错配对也会产生高 Δh。

#### $\mathcal{L}_{inv}$ — 幅值不变性

$$\mathcal{L}_{inv} = \|C_{couple}(a_g \cdot g, a_m \cdot m) - C_{couple}(g, m)\|_1$$

只看结构不看强度。每 5 步算一次省时间。

#### $\mathcal{L}_{ig\_reg}$ — IG 正则化 (v11 新增)

$$\mathcal{L}_{ig\_reg} = w \cdot \text{mean}(\text{ReLU}(-IG)^2)$$

只惩罚负 IG，不限制上限。鼓励模型输出正的信息增益。

### 5.3 Coverage-Gate-First 评价策略

```
评价流程（非训练目标）:
  1. Coverage Gate: calibration plot / ECE < 阈值?
  2. PPC (后验预测检查): 观测落在预测区间内?
  3. SBC (秩直方图): 秩均匀?
  4. → 以上通过后才看 IG
  5. SESOI Gate: CI 下界 > δ_min?
  6. → 深部 bin 多峰检验
  7. → Masked-RMSE 分区诊断
```

---

## 6. 训练策略

### 6.1 渐进式训练日程

```
Phase 0 (ep 1-10):   Warmup
  → coupling_strength: 0 → 1 (线性 ramp)
  → w_struct: 0.15 → 1.0 (curriculum, 渐增)
  → 只训 p_g, p_m + S_g, S_m (基础表示)

Phase 1 (ep 11-50):  耦合激活
  → 解冻 C_couple, full strength
  → L_depth 主导, coupling losses 参与
  → shuffled-pair 负控开启

Phase 2 (ep 51-150): 精调
  → ReduceLROnPlateau 监控 IG
  → LR 衰减 (patience=30)
  → IB 自动平衡熵 (v11)

Phase 3 (ep 151-200): 收敛
  → 极低 LR (1e-6 ~ 1e-5)
  → 模型微调表示
  → IG 可能回升 (v11 观察到的现象)
```

### 6.2 当前最佳超参

| 超参 | 值 | 说明 |
|------|-----|------|
| 学习率 lr | **1e-4** | ★ 低 LR 缓解先峰后崩 (从 3e-4 降低) |
| Batch size | 16 | GPU 内存允许 |
| Encoder width c | 64 | 全量配置 |
| K (深度 bin 数) | **32** | log(K)=3.47 bits (vs K=16 的 2.77) |
| α (V5/V3 混合比) | 0.1 | V5 主导 |
| λ_max | 30 | 最大耦合强度 |
| warmup epochs | 20 | coupling 线性 warmup |
| w_struct | 5.0 | 结构监督 (较强) |
| w_shuf | 1.5 | shuffled-pair |
| w_sel | 1.0 | selectivity |
| T_init | 1.0 | 温度初始值 |
| T clamp | [0.5, 3.0] | 防 softmax 极端值 |
| gate_disable | True | bootstrap 阶段跳过 confidence gate |
| IB_H_min_ratio | 0.6 | v11: 最小熵 60%·logK |
| ig_reg_weight | 0.1 | v11: IG 正则化 |
| grad_clip | 1.0 | 梯度裁剪 |
| optimizer | Adam (wd=1e-5) | |
| scheduler | ReduceLROnPlateau(mode=max, patience=30) | monitor IG |

### 6.3 优化器与调度

```python
optimizer = torch.optim.Adam(model.parameters(), lr=1e-4, weight_decay=1e-5)
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, mode='max', factor=0.5, patience=30,
    verbose=True, min_lr=1e-6)
```

**关键**: scheduler 监控 **IG**（非 Loss），因为我们的目标是最大化信息增益。

---

## 7. 版本演进与关键设计决策

### 7.1 版本谱系与 fatal 修复

```
v7 (2026-06-22)
 ├─ 基线架构完成, FEASIBILITY PASS
 └─ ✗ FATAL: C_couple 返回标量(B,), broadcast 到所有 K 层
             → softmax 平移不变 → 对 p_gm 零影响
             → max|Δp| = 1.5e-8 (数值零)

v8 (2026-06-24)
 ├─ 4 种 couple_mode 探索
 ├─ 发现 V3 (逐深度 bin) 能量最好
 └─ ✗ V1 方向反: 相似层加 cost (应该降 cost)

v9 (2026-06-24)
 ├─ V3+V5 双轨, 负 reward (-λ·R)
 ├─ selectivity + amplitude_invariance loss
 └─ ⚠ shuffled-pair shortcut: Δh_shuf = +0.020
             → 耦合学"结构存在"非真跨模态

v10 (2026-06-24) ★ 规范默认
 ├─ z_head 监督 (CE → z_support_true)
 ├─ V5 主导 (α 0.3→0.1, 去detach, 紧门控)
 ├─ V3 只交互 [qg·qm, |qg−qm|]
 ├─ fresh-perm shuffled 负控
 └─ ✓ shuffled Δh: 0.020→0.011 (本机 60-scene 验证)

v11 (2026-06-26) ★ 当前最新
 ├─ EntropyBottleneck 双路径解耦
 ├─ IG 正则化 (relu(-IG)²)
 └─ ★ 先峰后崩减缓 3.5x, Δh(shuf)≈0, 晚期IG回升
```

### 7.2 关键设计决策记录

| 决策 | 选择 | 弃选方案 | 理由 |
|------|------|---------|------|
| 融合层级 | 深度假设层 cost volume | 特征层 attention (EGU) | 避撞 + 物理意义明确 |
| C_couple 度量 | Wasserstein + Dice | JS/KL 散度 | 深度有序性 + 支撑域 |
| 灵敏度处理 | 归一化白化 (C-μ)/σ | 乘法衰减 e^{-αz} | 防双重计算 (ChatGPT fatal) |
| μ/σ 来源 | register_buffer 预注册 | 样本统计 | 防标签泄漏 (DeepSeek fatal①) |
| 耦合形式 | 负 reward (-λ·R) | 正惩罚 (+λ·D) | 一致层降 cost 非加 cost |
| V5/V3 混合 | V5 主导 (α=0.1) | V3 主导 | 物理 backbone 更可解释 |
| 解码器 | B'-lite mixture | Conditional INR | INR 风险高, B' FEASIBILITY PASS |
| L_fwd | 仅评估不作损失 | 作主损失 | 防隐式命题污染 |
| 信息瓶颈 | 双路径解耦 (v11) | 无 / 单路径 | 解决 L_depth 与 coupling 目标冲突 |
| LR | 1e-4 (低) | 3e-4 (标准) | 缓解先峰后崩 8x (实验 F 发现) |

---

## 8. 与已有方法的机制区分

### 8.1 方法对比表

| 维度 | Zhou2026 PDM-Net | EGU Xi&Wang | Fang2025 | Xu2026 | **CM-DCV (本方法)** |
|------|------------------|-------------|----------|--------|---------------------|
| 任务 | 单磁反演 | 跨模态分割 | 多任务联合 | 残差学习 | **重磁联合深度不确定** |
| 深度处理 | 固定衰减掩膜 | 隐式（体素坐标） | 隐式 | 隐式 | **显式 K 假设竞争** |
| 融合位置 | N/A | 特征层 attention | 特征层 | loss 层 | **深度假设层 cost volume** |
| 不确定性 | 无 | 弱（分类置信度） | 无 | 无 | **p(z)+IG+conflict+OOD** |
| 物理约束 | 固定 e^{-z^1.5} | 弱 | 弱 | 弱 | **灵敏度归一化+footprint+Wasserstein** |
| 可证伪 | 部分 | 否 | 否 | 否 | **完整负控矩阵 (5 held-out)** |
| 耦合对象 | N/A | 特征对齐 | 物性值融合 | 残差 | **结构-only (深度Wasserstein+支撑Dice)** |
| 输出 | 二维掩膜 | 分割图 | ρ/κ 点估计 | 残差 | **p(z)+IG+3D ρ/κ mixture** |

### 8.2 与 MR01/MR02 的关系

| | MR01 c(x) | MR02 M4 | **CM-DCV (MR03)** |
|--|-----------|---------|-------------------|
| 对象 | 二维空间耦合 | claim-safety 协议 | **深度维概率分布** |
| 输出 | 连续标量场 c(x) | 评测协议 | **K 维离散 p(z) + IG** |
| 目的 | "该耦合多强" | "该归谁/在多深" | **"多出了多少信息"** |
| 关系 | 互补 | 评价骨架来源 | **headline** |

### 8.3 避撞硬约束

| # | 避撞对象 | 区分点 | 风险 |
|---|---------|--------|------|
| H1 | Zhou2026 | 数据驱动 vs 固定衰减; 跨模 vs 单磁; p(z) vs 点估计 | 低 |
| H2 | EGU Xi&Wang | 深度假设层 vs 特征层; 无 gating | 低 |
| H3 | Fang2025 | 深度 cost volume vs 特征层融合; 结构-only vs 物性融合 | 低 |
| H4 | Xu2026 | 深度代价体 vs 残差 loss | 低 (正交) |
| H5 | MR01 c(x) | 深度概率 vs 连续场; 不同对象 | 中 |

---

## 9. 可证伪框架 (Bets)

### 9.1 核心 Bet 验证结果

| Bet | 操作 | 预期 | 实际 (v11 K=32) | 状态 |
|-----|------|------|-----------------|------|
| **Bet1** | 有效域 IG > 失效域 IG | valid IG >> noncoloc/zerocouple | BestIG=+1.61, FinalIG=-0.07 | ⚠ 部分通过 |
| **Bet2** | 去 C_couple 后 p(z) 展宽 | ΔH > 0 | Final Δh = -0.007 | ⚠ 未通过 (根本问题) |
| **Bet3** | zerocouple IG ≈ 0 | ≈ 0 | Δh(shuf) ≈ -0.002 | **✅ 通过** |
| **Bet4** | shuffled pair IG ≈ 0 | ≈ 0 | **Δh(shuf) ≈ 0 (v11!)** | **✅ 通过** |
| **Bet5** | wrong-kernel IG < 0 | < 0 | 待全量验证 | 待验证 |

### 9.2 闭环关卡（全量训练后必须过）

| 关卡 | 阈值 | v11 K=32 当前值 | 状态 |
|------|------|-----------------|------|
| shuffled Δh | ≤ 0.01 | **≈ 0.002** | **✅ 通过** |
| valid SESOI Δh | ≥ 0.03 | -0.007 (best: +0.006) | ❌ 未通过 |
| 域排序 | valid > noncoloc > zerocouple | 部分满足 | ⚠ |
| coverage/ECE | < 0.05 | 待校准 | 待验证 |
| 5 held-out 负控 | 全通过 | 2/5 验证 | 进行中 |

---

## 10. 附录: 符号表与文件索引

### 10.1 关键符号表

| 符号 | 定义 | 典型形状 |
|------|------|---------|
| $g$ | 地表重力异常 | $(B, 1, H, W)$ |
| $m$ | 地表磁异常 (TMI) | $(B, 1, H, W)$ |
| $\xi_g, \xi_m$ | 物理元数据 | $(B, 10)$ |
| $f_g, f_m$ | 编码特征 | $(B, c, h, w)$ |
| $S_g, S_m$ | 结构证据 | $\{(B,K), (B,3)\}$ |
| $S_g^{ib}, S_m^{ib}$ | IB 平滑结构证据 (v11) | $(B, K)$ |
| $\tilde{C}_g, \tilde{C}_m$ | 灵敏度归一化代价 | $(B, K)$ |
| $C_{couple}$ | 结构耦合代价 | $(B, K)$, **≤ 0** |
| $C_{gm}$ | 联合深度代价 | $(B, K)$ |
| $p_g, p_m, p_{gm}$ | 深度后验概率 | $(B, K)$, $\sum = 1$ |
| $IG$ | 跨模态信息增益 | $(B,)$, **可为负** |
| $T$ | 校准温度 | 标量, $> 0.1$ |
| $K$ | 深度假设数 | 16 (demo) / 32 (全量) / 60 (目标) |
| $M$ | mixture mode 数 | 2 |
| $\rho$ | 密度 contrast | kg/m³, $[-800, 800]$ |
| $\kappa$ | 磁化率 | SI, $[0, 0.3]$ |
| $\alpha$ | V5/V3 混合比 | 0.1 (V5 主导) |
| $\lambda$ | 耦合强度 | 可学, $\in [0, \lambda_{max}]$ |
| $H_{min}$ | IB 最小熵 | $0.6 \cdot \log K$ |
| $T_{ib}$ | IB 温度 | 可学, $> 1$ |

### 10.2 文件索引

| 文件 | 内容 | 状态 |
|------|------|------|
| `code/cm_dcv_feasibility.py` | v7 基线模型 (headline + decoder) | FEASIBILITY PASS |
| `code/cm_dcv_model.py` | v7 模型骨架 | 参考 |
| `code/cm_dcv_v9_dualtrack.py` | **v9/v10/v11 模型 + losses** | **规范默认 = CMDCV_v11** |
| `code/cm_dcv_v8_couple_fix.py` | v8 探索 (4 couple_mode) | 探索完成 |
| `code/train_v2.py` | v2 训练脚本 (支持 v10/v11) | **当前使用** |
| `code/gendata_v2.py` | v2 数据生成器 (14 模板) | 已验证 |
| `code/gen_G_matrix_v3.py` | 点质量近似 G 矩阵 (corr=0.984) | 已验证 |
| `code/analyze_ep1_best.py` | ep1 best checkpoint 分析 | 已运行 |
| `code/ablation_v2.py` | 消融实验矩阵 (22 configs) | 已完成 |
| `docs/training_version_log.md` | **训练版本与策略维护日志** | **本文档的姊妹篇** |
| `docs/v2_training_experiments.md` | v2 实验详细记录 | 已完成 |

### 10.3 数据集索引

| 目录 | 场景数 | K | 网格 | 状态 |
|------|--------|---|------|------|
| `dataset_pilot_30km/` | 40 | 16 | 32×32×16 | pilot 完成 |
| `dataset_small_30km/` | 60 | 16 | 32×32×16 | v9 验证用 |
| `dataset_v2_pilot/` | 40 | 16 | 32×32×16 | v2 pilot |
| `dataset_v2_500/` | 500 | 16 | 32×32×16 | C 系列实验 |
| `dataset_v2_full/` | 2000 | 16 | 32×32×16 | D/F 系列实验 |
| `dataset_v2_K32/` | 2000 | 32 | 32×32×32 | **G/H 系列实验 (当前)** |
| `dataset_full_30km/` | ~9600 | 60 | 64×64×60 | 目标全量 (待生成) |

---

*文档版本: v1 (2026-06-26)*
*对应模型版本: CMDCV_v11 (信息瓶颈解耦耦合)*
*最后更新: 实验 H0 完成后*
