# MR03 CMDCV 项目文档索引

> **项目**: MR03_跨模态深度代价体_CMDCV (Cross-Modal Depth-Stratified Sensitivity Volume)
> **核心命题**: 条件跨模态深度信息增益 IG = min{H[p_g], H[p_m]} - H[p_gm] > 0
> **当前模型版本**: **CMDCV_v11**（信息瓶颈解耦耦合）
> **最后更新**: 2026-06-26

---

## 文档导航

### 必读（按顺序）

| # | 文档 | 内容 | 何时读 |
|---|------|------|--------|
| **1** | **[CMDCV_Architecture.md](./CMDCV_Architecture.md)** | **模型架构详细技术文档** — 所有模块的数学公式、张量流、设计决策、版本演进 | 了解模型时 |
| **2** | **[training_version_log.md](./training_version_log.md)** | **训练版本与策略维护日志** — 每次训练的配置、结果、问题、checkpoint 索引 | 训练前后 |
| **3** | [v2_training_experiments.md](./v2_training_experiments.md) | v2 实验完整记录 (A-H 共 8 轮实验) | 查看历史实验细节 |

### 文档详细说明

#### 1. CMDCV_Architecture.md — 模型架构文档

**这是当前模型的完整技术规格书。**

包含：
- 科学命题与定位
- 整体架构图（ASCII）
- **10 个模块逐一详解**（含数学公式）:
  - PhysicsNormDualEncoder (物理归一化双分支编码)
  - StructureEvidence (半显式结构证据头)
  - DepthStratifiedSensitivityVolume (深度分层灵敏度代价体)
  - CouplingV5 (物理耦合: 峰层门控 + 1D OT)
  - V3CouplingHead (学习残差耦合)
  - **EntropyBottleneck (信息瓶颈, v11 新增)**
  - CMDCV_v11 主模型组装
  - MixtureDecoder3D (B'-lite 物性解码器)
- 完整张量形状流（B=16, K=32, c=64 全量配置）
- 损失函数体系（6 组 loss 的数学定义）
- 训练策略（渐进式日程 + 最佳超参表）
- 版本演进谱系（v7→v8→v9→v10→v11，每个 fatal 修复）
- 与 Zhou2026/EGU/Fang2025/Xu2026/MR01/MR02 的机制区分
- 可证伪框架（Bet1-Bet5 + 闭环关卡状态）

#### 2. training_version_log.md — 训练日志

**每次训练完成后更新此文件。**

包含：
- 模型版本谱系表（v7-v11，代码位置，状态）
- 策略版本与超参（推荐配置 `--v11 --lr 1e-4 --K 32 ...`）
- **训练实验详细记录**:
  - 实验 D0: K=16 全量 (2000 scenes, 200 ep)
  - 实验 F: 消融矩阵 (22 configs) — ★ lr=1e-4 突破
  - 实验 G0: K=32 全量 — 短暂恢复窗
  - **实验 H0: v11 IB (K=32) — 当前最佳**
- 已知问题清单（P0/P1/P2 三级）
- Checkpoint 索引（含路径和关键指标）
- 下次训练计划

#### 3. v2_training_experiments.md — 历史实验记录

v2 训练阶段的完整实验记录（A-H 共 8 轮），包含每轮的超参、时间线、发现。

---

## 代码文件索引

| 文件 | 用途 | 状态 |
|------|------|------|
| `code/cm_dcv_feasibility.py` | v7 基线模型 + 自检 | FEASIBILITY PASS |
| `code/cm_dcv_v9_dualtrack.py` | **v9/v10/v11 模型定义 + losses** | **规范默认 = CMDCV_v11** |
| `code/cm_dcv_v8_couple_fix.py` | v8 探索版 | 探索完成 |
| `code/train_v2.py` | **v2 训练脚本 (支持 v10/v11)** | **当前使用** |
| `code/gendata_v2.py` | v2 数据生成器 (14 模板) | 已验证 |
| `code/gen_G_matrix_v3.py` | 点质量近似 G 矩阵 (corr=0.984) | 已验证 |
| `code/analyze_ep1_best.py` | ep1 best checkpoint 分析 | 已运行 |
| `code/ablation_v2.py` | 消融实验矩阵 (22 configs) | 已完成 |

## 数据集索引

| 目录 | 场景数 | K | 状态 |
|------|--------|---|------|
| `dataset_v2_K32/` | 2000 | 32 | **当前全量 (G/H 实验用)** |
| `dataset_v2_full/` | 2000 | 16 | D/F 实验用 |
| `dataset_v2_500/` | 500 | 16 | C 实验用 |
| `dataset_v2_pilot/` | 40 | 16 | pilot |
| `dataset_full_30km/` | ~9600 | 60 | 目标全量 (待生成) |

## Checkpoint 快速查找

| 想要什么 | 去哪里找 |
|---------|----------|
| **最佳 IG checkpoint** | `dataset_v2_K32/cmdcv_v2_best.pt` (v11, ep1, IG=+1.61) |
| **最终收敛 checkpoint** | `dataset_v2_K32/cmdcv_v2_K32_final.pt` (v11, ep200, IG=-0.07) |
| K=16 最佳 | `dataset_v2_full/cmdcv_v2_best.pt` (v10, ep1, IG=+3.41) |
| K=16 最终 | `dataset_v2_full/cmdcv_v2_K16_final.pt` (v10, ep200) |

## 当前最佳配置速查

```bash
python train_v2.py \
    --data dataset_v2_K32 \
    --epochs 200 \
    --batch 16 \
    --K 32 \
    --lr 1e-4 \
    --c 64 \
    --t-init 1.0 \
    --no-gate \
    --lam-max 30 \
    --warmup 20 \
    --w-struct 5.0 \
    --w-shuf 1.5 \
    --v11 \
    --ib-hmin 0.6 \
    --ig-reg 0.1
```

## 关键指标速查

| 指标 | v10 (K=32) | **v11 IB (K=32)** | 目标 |
|------|-------------|-------------------|------|
| Best IG | 1.34 | **1.61** | > 0 |
| Final IG | -0.22 | **-0.07** | **> 0** ❌ |
| Final Δh(valid) | -0.010 | **-0.007** | > 0.03 ❌ |
| Δh(shuf) | -0.003 | **-0.002** | < 0.01 ✅ |
| 崩塌幅度 | 5.84 | **1.68 (3.5x↓)** | — |

## 下一步

1. **两阶段训练**: 先冻结 coupling 只训 depth → 再解冻（解决 L_depth 与 coupling 目标冲突）
2. **降低 w_struct**: 从 5.0 到 2.0（减少 Sg_z 过尖压力）
3. **增加 IB H_min_ratio**: 从 0.6 到 0.75（给耦合留更多熵空间）
4. **扩数据到 5000+ scenes**

---

*本文档由 Claude Code 自动生成于 2026-06-26*
*对应 Obsidian 文档: `Obsidian\xiang\三模型分析\rounds\MR03_跨模态深度代价体_CMDCV\`*
