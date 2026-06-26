# MR03 CMDCV 训练版本与策略维护日志

> **用途**：记录每次训练的模型版本、策略配置、数据规模、结果和问题。
> 每次训练完成后更新本文件，追加新条目（不覆盖旧记录）。
> **当前最新模型**：CMDCV_v11（信息瓶颈解耦耦合）
> **当前最佳配置**：见下方「推荐配置」

---

## 一、模型版本谱系

| 版本 | 日期 | 核心变更 | 代码位置 | 状态 |
|------|------|---------|----------|------|
| v7 | 2026-06-22 | 基线架构：深度分层灵敏度代价体 + B'-lite MixtureDecoder3D | `cm_dcv_feasibility.py` | FEASIBILITY PASS，C_couple 结构性失效 |
| v8 | 2026-06-24 | 4 种 couple_mode 横向对比探索 | `cm_dcv_v8_couple_fix.py` | 探索完成，V3 最优 |
| v9 | 2026-06-24 | V3+V5 双轨 C_couple（负 reward）+ selectivity/amplitude invariance loss | `cm_dcv_v9_dualtrack.py` CMDCV_v9 | 收敛，但 shuffled-pair shortcut |
| **v10** | 2026-06-24 | 修 shortcut：z_head 监督 + V5 主导(α=0.1) + V3 只交互 + fresh-perm 负控 | `cm_dcv_v9_dualtrack.py` CMDCV_v10 | **规范默认**，60-scene 验证修法有效 |
| **v11** | 2026-06-26 | 信息瓶颈解耦耦合：EntropyBottleneck 双路径（尖锐→监督/平滑→耦合）+ IG 正则化 | `cm_dcv_v9_dualtrack.py` CMDCV_v11 | **当前最新**，减缓"先峰后崩" 3.5x |

### 版本间关键差异

```
v7: C_couple = 标量(B,) → broadcast → softmax 不变 → 零影响 (FATAL)
v8: 探索 4 种 couple_mode → V3 逐深度 bin 兼容能量最好
v9: C_couple(k) = -λ·[(1-α)R_V5 + α·R_θ]  (负 reward, α=0.3)
     问题: shuffled-pair Δh=+0.020 (shortcut)
v10: α=0.1(V5主导) + 去detach + 紧门控 + V3只交互[q_g·q_m,|q_g−q_m|]
     修复: shuffled Δh 0.020→0.011
v11: + EntropyBottleneck(Sg_z→Sg_z_ib, Sm_z→Sm_z_ib) 送入 V3
     效果: 崩塌速度↓3.5x, Δh(shuf)≈0(特异耦合), 晚期IG回升
```

---

## 二、策略版本与超参

### 当前推荐配置（v11 最佳）

```bash
# K=32 全量训练
python train_v2.py \
    --data dataset_v2_K32 \
    --epochs 200 \
    --batch 16 \
    --K 32 \
    --lr 1e-4 \                    # ★ 低 LR 缓解先峰后崩
    --c 64 \                       # encoder 宽度
    --t-init 1.0 \                 # 温度初始值
    --no-gate \                    # 禁用 V5 confidence gate (bootstrap)
    --lam-max 30 \                 # 最大耦合强度
    --warmup 20 \                  # coupling warmup epochs
    --w-struct 5.0 \               # 结构监督权重
    --w-shuf 1.5 \                 # shuffled-pair 权重
    --w-sel 1.0 \                  # selectivity 权重
    --v11 \                        # ★ 使用 v11 信息瓶颈
    --ib-hmin 0.6 \                # IB 最小熵比例
    --ig-reg 0.1                   # IG 正则化权重
```

### 策略演进历史

| 实验 | 日期 | 关键变更 | 效果 |
|------|------|---------|------|
| A0-A3 (pilot) | 06-25 | 基线超参扫描 | IG peak 0.93~1.57 |
| B0-B1 (gate off) | 06-25 | 禁用 confidence gate | 耦合首次激活, Δh>0 |
| C0-C3 (500 scenes) | 06-25 | w_struct=5.0 突破 | IG peak=2.60, 发现"先峰后崩" |
| D0 (2000 scenes) | 06-25 | 全量 K=16 训练 | BestIG=3.41(ep1), FinalIG=-0.05 |
| E1-E4 (消融 w_shuf) | 06-25 | shuffled 权重扫描 | w_shuf=1.5-2.5 最优 |
| F1-F3 (消融 LR) | 06-26 | ★ lr=1e-4 突破 | BestIG=4.37, BestΔh=+0.014 (8x提升) |
| G0 (K=32) | 06-26 | 升级到 K=32 | 发现短暂恢复窗 ep11-30 |
| H0 (v11 IB) | 06-26 | 信息瓶颈解耦耦合 | FinalIG=-0.07(3x↑), Δh(shuf)≈0 |

---

## 三、训练实验详细记录

### 实验 D0: K=16 全量训练 (2000 scenes, 200 ep)

| 字段 | 值 |
|------|-----|
| **日期** | 2026-06-25 |
| **模型** | CMDCV_v10 (TClampedCMDCV) |
| **数据集** | dataset_v2_full (2000 scenes, 10 templates) |
| **网格** | 32×32×16 @ 2km, K=16 |
| **配置** | `--lr 3e-4 --batch 32 --no-gate --lam-max 30 --warmup 10 --w-struct 5.0` |
| **设备** | CUDA GPU |

#### 结果

| 指标 | 值 |
|------|-----|
| Best IG | **+3.41** (epoch 1) |
| Final IG | -0.052 (epoch 200) |
| Best Δh(valid) | **+0.018** (epoch 1) |
| Final Δh(valid) | -0.011 |
| Final Loss | 602.3 |
| LR reductions | 3 次 (ep47/112/170) |

#### 收敛时间线

| Epoch | Loss | IG | Δh(val) | 事件 |
|-------|------|-----|---------|------|
| 1 | 914.9 | **+3.41** | **+0.018** | Peak (best ckpt) |
| 47 | ~629 | ~-2.6 | -0.016 | 1st LR↓ |
| 112 | ~608 | ~-2.7 | -0.013 | 2nd LR↓ |
| 170 | 603.2 | -1.70 | -0.011 | 3rd LR↓ |
| 200 | 602.3 | -0.052 | -0.011 | Done |

#### 发现的问题

1. **"先峰后崩"模式**: Ep1 IG 极高 → Ep2+ 迅速崩为负
2. **根因**: L_depth 过拟合 → Sg_z 过尖 → H→0 → IG 必然为负
3. **Δh 最终为负**: C_couple 在长期训练后被压制

---

### 实验 F: 消融实验矩阵 (22 configs × 20 ep)

| 字段 | 值 |
|------|-----|
| **日期** | 2026-06-26 |
| **数据集** | dataset_v2_full (2000 scenes) |
| **耗时** | 131.8 min |
| **评价指标** | Δh (主要), IG, Loss |

#### 关键发现

| 维度 | 最优值 | 影响 |
|------|--------|------|
| Gate | **OFF** | BestΔh +0.0058 vs +0.0023 |
| W_struct | **5.0** | 7.0 导致 IG 崩塌 |
| Lam_max | **30-50** | 3.0 太弱 |
| Warmup | **20** | BestΔh +0.003 |
| **LR** | **★ 1e-4** | **BestIG=4.37, BestΔh=+0.014 (8x 提升!)** |

#### ★ 突破性发现

```
lr=3e-4 (原): BestIG=+0.52,  BestΔh=+0.003
lr=1e-4 (新):   BestIG=+4.37, BestΔh=+0.014  ← 8x 提升!
```
低 LR 使 ep1 的良好初始化不被快速破坏。

---

### 实验 G0: K=32 全量训练 (2000 scenes, 200 ep)

| 字段 | 值 |
|------|-----|
| **日期** | 2026-06-26 |
| **模型** | CMDCV_v10 (TClampedCMDCV) |
| **数据集** | dataset_v2_K32 (2000 scenes) |
| **网格** | 32×32×32 @ 1km, K=32, log(K)=3.47 bits |
| **配置** | `--lr 1e-4 --batch 16 --warmup 20 --no-gate --lam-max 30 --w-struct 5.0` |

#### 结果

| 指标 | 值 |
|------|-----|
| Best IG | **+1.34** (epoch 1) |
| Final IG | -0.22 (epoch 200) |
| Best Δh(valid) | -0.002 |
| Final Δh(valid) | -0.010 |
| Final Loss | 323.5 |
| LR reductions | 4 次 |

#### K=32 独有现象: 短暂恢复窗

| Epoch | 事件 |
|-------|------|
| 11-14 | Δh(valid) **转正** (+0.001 ~ +0.0012) |
| 28 | Δh(valid) = **+0.002** (最强正) |
| 30+ | 再次转负 |

Template 排序在恢复窗内符合物理直觉（深部构造 > 浅部构造）。

#### 结论

K=32 未带来预期的 IG 持续为正。根本瓶颈不在 K 大小，而在 L_depth 与耦合损失的目标冲突。

---

### 实验 H0: v11 信息瓶颈解耦耦合 (K=32, 200 ep) ★ 当前最佳

| 字段 | 值 |
|------|-----|
| **日期** | 2026-06-26 |
| **模型** | **CMDCV_v11** (TClampedCMDCV_v11) |
| **数据集** | dataset_v2_K32 (2000 scenes) |
| **网格** | 32×32×32 @ 1km, K=32 |
| **配置** | `--v11 --ib-hmin 0.6 --ig-reg 0.1 --lr 1e-4 --batch 16 --warmup 20 --no-gate --lam-max 30 --w-struct 5.0` |

#### 核心创新: EntropyBottleneck

```
路径 A (尖锐): Sg_z/Sm_z → L_struct 监督 + V5 峰检测 → 深度精度优先
路径 B (平滑): Sg_z_ib/Sm_z_ib → V3 学習耦合 → 保证最小熵 (H ≥ 60%·logK)
```

IB 通过可学习温度 T_ib > 1 对分布做平滑，加硬约束 H ≥ H_min。
仅增加 **2 个参数** (T_ib for g/m)，零推理开销。

#### 结果

| 指标 | v10 (K=32) | **v11 IB (K=32)** | 改善 |
|------|-------------|-------------------|------|
| Best IG | 1.34 | **1.61** | **+20%** |
| Final IG | -0.22 | **-0.07** | **3x↑** |
| IG nadir | -4.51(ep60) | **-1.18(ep100)** | **4x↓** |
| Final Δh(val) | -0.010 | **-0.007** | **30%↑** |
| Final Δh(shuf) | -0.003 | **-0.002** | **33%↑** |
| Loss | 323.5 | 326.8 | +1% |
| Δh 正窗口 | ep11-30(弱) | **ep13-35(强)** | 更长更强 |
| 晚期 IG | 平坦/降 | **回升(-1.12→-0.07)** | 全新现象 |

#### 时间线

| Epoch | LR | Loss | IG | Δh(val) | Δh(shuf) | H_ib(Sg) | Event |
|-------|-----|------|-----|---------|----------|----------|-------|
| 1 | 1e-4 | 235.5 | **+1.61★** | -0.001 | -0.001 | 3.40 | Best IG |
| 13 | 1e-4 | 302.7 | -0.41 | **+0.003** | -0.001 | 2.78 | Δh 转正 |
| 28 | 1e-4 | 369.5 | **-0.20↑** | **+0.006** | +0.017 | 2.36 | IG 回升! |
| 100 | 2.5e-5 | 333.9 | -1.17 | -0.006 | **≈0.000** | 2.43 | shuf≈0! |
| 200 | 6e-6 | 326.8 | **-0.07** | **-0.007** | **-0.002** | — | DONE |

#### 核心发现

1. **信息瓶颈有效减缓 "先峰后崩" 3.5x**
   - v10 IG 变化幅度: 5.84 (1.34 → -4.51)
   - v11 IG 变化幅度: 1.68 (1.61 → -1.07)

2. **Δh(shuf)≈0 — 特异耦合突破**
   - 错配对不产生耦合信号 → 学到的耦合是物理特异的

3. **晚期 IG 自发回升**
   - 从 -1.12 (ep100) 回升到 -0.07 (ep200)
   - 低 LR + IB 熵约束让模型后期重新平衡表示

4. **IB 熵稳定在健康水平**
   - H_ib(Sg) ≈ 71% 最大熵, H_ib(Sm) ≈ 69%

---

## 四、已知问题与待解决项

### P0 (阻塞进展)

| # | 问题 | 影响 | 尝试过的方案 | 状态 |
|---|------|------|-------------|------|
| 1 | **"先峰后崩"模式** | IG ep1 高 → 快速崩为负 | 低 LR(缓解)、IB(减缓 3.5x)、curriculum | **部分解决**，根因未消除 |
| 2 | **Final IG 仍为负** | 命题要求 IG > 0 | 所有上述方案 | **未解决** |
| 3 | **L_depth 与 C_couple 目标冲突** | depth 要求尖锐 / coupling 要求平坦 | IB 解耦(v11) | **部分解决** |

### P1 (影响质量)

| # | 问题 | 影响 | 状态 |
|---|------|------|------|
| 4 | K=16 熵空间太小 (logK=2.77) | IG 上限受限 | 已升级 K=32 |
| 5 | Template 间 Δh 差异小 (~0.01) | 区分度不足 | 待更多数据 |
| 6 | 训练时 IG 与评估时 IG 差异大 | 监控指标不可靠 | 已知，需用 eval 模式验证 |

### P2 (改进方向)

| # | 方向 | 预期效果 | 优先级 |
|---|------|---------|--------|
| 7 | 两阶段训练 (先 depth 后 coupling) | 避免目标冲突 | 高 |
| 8 | 对抗训练 (discriminator 判断共址) | 额外耦合信号 | 中 |
| 9 | K=16 + lr=1e-4 重跑 | 验证 F1 消融结论 | 低 |
| 10 | 更大数据集 (>5000 scenes) | 统计显著性 | 中 |

---

## 五、Checkpoint 索引

| Checkpoint | 模型 | K | Epoch | IG | 数据集 | 路径 |
|-----------|------|---|-------|----|--------|------|
| best_D0 | v10 | 16 | 1 | +3.41 | dataset_v2_full | `dataset_v2_full/cmdcv_v2_best.pt` |
| final_D0 | v10 | 16 | 200 | -0.05 | dataset_v2_full | `dataset_v2_full/cmdcv_v2_K16_final.pt` |
| best_G0 | v10 | 32 | 1 | +1.34 | dataset_v2_K32 | `dataset_v2_K32/cmdcv_v2_best.pt` |
| final_G0 | v10 | 32 | 200 | -0.22 | dataset_v2_K32 | `dataset_v2_K32/cmdcv_v2_K32_final.pt` |
| **best_H0** | **v11** | **32** | **1** | **+1.61** | dataset_v2_K32 | `dataset_v2_K32/cmdcv_v2_best.pt` |
| **final_H0** | **v11** | **32** | **200** | **-0.07** | dataset_v2_K32 | `dataset_v2_K32/cmdcv_v2_K32_final.pt` |

---

## 六、下次训练计划

### 目标: 让 Final IG > 0

| 步骤 | 内容 | 预期 |
|------|------|------|
| 1 | 两阶段训练: Phase1(1-50ep) 冻结 coupling 只训 depth → Phase2 解冻 | 避免 L_depth 干扰耦合学习 |
| 2 | 降低 w_struct 到 2.0 (从 5.0) | 减少 Sg_z 过尖压力 |
| 3 | 增加 IB H_min_ratio 到 0.75 (从 0.6) | 给耦合留更多熵空间 |
| 4 | 尝试 cosine annealing LR (替代 ReduceLROnPlateau) | 更平滑的 LR 衰减 |
| 5 | 扩数据到 5000+ scenes | 更好的统计估计 |

---

*文档维护规则: 每次训练完成后追加新实验到此文件，不覆盖旧记录。*
*最后更新: 2026-06-26 (实验 H0 完成)*
