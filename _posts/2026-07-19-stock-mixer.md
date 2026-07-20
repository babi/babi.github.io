---
layout: post
title: StockMixer — 简洁而强大的MLP架构股票预测模型
subtitle: AAAI 2024 论文解读：用纯MLP超越Transformer和GNN的股票预测新范式
date: 2026-07-19
tags: [stock-prediction, deep-learning, mlp, quant, paper]
author: babi
---

# StockMixer: 股票预测的MLP新范式

> **论文**: [StockMixer: A Simple Yet Strong MLP-Based Architecture for Stock Price Forecasting](https://ojs.aaai.org/index.php/AAAI/article/view/28681) (AAAI 2024)  
> **作者**: Jinyong Fan, Yanyan Shen — 上海交通大学  
> **代码**: [github.com/SJTU-DMTai/StockMixer](https://github.com/SJTU-DMTai/StockMixer) ⭐  
> **arXiv**: [2412.11192](https://arxiv.org/abs/2412.11192)

---

## 一、背景与动机

股票价格预测是量化金融领域的核心难题。传统的深度学习方案通常依赖复杂的混合架构——将 **RNN、GNN、Transformer** 等模块拼接在一起以捕捉时间依赖、股票间关联和指标交互。然而这类方案存在两个固有问题：

1. **优化困难** — 复杂子网络在有限的股票数据上容易过拟合，调参成本极高；
2. **效率低下** — GNN 的消息传递机制在大规模股票池上计算开销大，且静态图结构无法反映动态的股票关联。

StockMixer 的核心洞察是：**不需要花哨的混合架构，精心设计的 MLP 足以胜任**。它用三个"混合块"（Mixing Blocks）分别建模三种相关性，整个模型简洁、轻量、易优化，却在 NASDAQ、NYSE 和 S&P 500 三大基准上全面超越了 SOTA 方法。

---

## 二、模型架构

StockMixer 的输入是一个三维张量：**(股票 × 时间步 × 技术指标)**。整个管道按顺序经过三个 Mixing 块：

```
输入 → Indicator Mixing → Time Mixing → Stock Mixing → 预测输出
```

![StockMixer 架构图](https://github.com/SJTU-DMTai/StockMixer/raw/master/framework.png)

### 1. Indicator Mixing（指标混合）

在每个（股票，时间步）位置，对技术指标维度做 MLP 混合，捕捉指标间的交互关系。例如开盘价、收盘价、最高价、最低价、成交量等指标之间的关联模式。

这是最直接的 MLP Mixing，与 MLP-Mixer 的思路一致。

### 2. Time Mixing（时间混合）— 核心创新

这是与标准 MLP-Mixer 区别最大的地方，包含两个关键设计：

**Patch-based Multi-scale（基于补丁的多尺度）**  
将时间序列切分成多个尺度的补丁（Patch），在每个补丁内部和跨补丁之间交换信息。这样模型既能捕捉短期的局部趋势（如连续几天的上涨），也能感知长期的整体模式。

**Causal Masking（因果掩码）**  
时间混合层使用**上三角权重矩阵**，强制信息只能从过去流向未来。这保证了预测时的时序因果性——模型在预测 t+1 时不能"偷看"未来的数据。

### 3. Stock Mixing（股票混合）— 另一核心创新

传统方案用 GNN 或全连接直接建模股票间的关联，但 StockMixer 提出了更优雅的方式：

**Market-aware Stock Mixing（市场感知的股票混合）**

1. **Stock → Market**：用 MLP 将所有股票的信息聚合到一个隐式的"市场状态"向量中；
2. **Market → Stock**：将这个市场状态作为上下文，反向影响每只股票的表示。

这种两阶段设计避免了 O(N²) 的股票间直接交互（N 为股票数），计算复杂度大幅降低，同时能捕捉动态的市场层面的影响——例如大盘情绪如何传导到个股。

---

## 三、关键创新总结

| 模块 | 创新点 | 解决的问题 |
|------|--------|-----------|
| **Indicator Mixing** | MLP 混合指标维度 | 替代手工特征交互 |
| **Time Mixing** | 多尺度 Patch + Causal Masking | 时序建模 + 因果约束 |
| **Stock Mixing** | Stock → Market → Stock 两阶段 | 动态股票关联，避免 O(N²) |

---

## 四、实验结果

### 数据集

- **NASDAQ**：美国纳斯达克市场股票
- **NYSE**：纽约证券交易所股票
- **S&P 500**：标普500成分股

### 评价指标

- **IC / RIC**：信息系数和秩信息系数（衡量预测与真实值的相关性）
- **Precision@N**：Top-N 预测的准确率
- **Sharpe Ratio**：夏普比率（衡量收益风险调整）

### 主要结果

StockMixer 在所有基准上全面超越 SOTA，平均提升幅度：

- **IC 提升 7.6%**
- **RIC 提升 10.8%**
- **Precision@N 提升 10.9%**
- 同时大幅降低内存占用和运行时间

与当时最强模型对比（如 ATFNet），StockMixer 在保持更高预测精度的同时，参数量和推理时间均显著减少。

---

## 五、为什么 StockMixer 有效？

1. **化繁为简**：去掉了 RNN/GNN/Attention，避免了混合架构中常见的优化不稳定问题；
2. **因果设计**：Causal Masking 确保了时序建模的合理性，这是一个常被 MLP 方案忽略的关键细节；
3. **市场先验**：Stock → Market → Stock 的设计巧妙地引入了"市场环境影响个股"的金融学先验，比 GNN 更轻量且更具解释力；
4. **多尺度感知**：Patch 机制让 MLP 也能捕捉不同粒度的时序模式。

---

## 六、复现与使用

官方代码基于 **Python 3.7 + PyTorch 1.10**，结构清晰，容易上手：

```bash
git clone https://github.com/SJTU-DMTai/StockMixer.git
cd StockMixer
pip install -r requirements.txt
python train.py --dataset nasdaq
```

### 环境依赖

- Python 3.7
- PyTorch ~1.10.1
- NumPy ~1.21.5
- PyYAML, pandas, tqdm, matplotlib

---

## 七、总结与展望

StockMixer 证明了在股票预测领域，**精心设计的 MLP 架构可以完胜复杂的混合模型**。它的成功对量化研究有几点启示：

- **先让人工智能简化**：在堆叠复杂模块之前，先思考问题的核心结构能否用更简单的模块建模；
- **领域知识不可少**：Causal Masking 和 Market-aware 设计都是金融领域先验，模型架构应编码这些知识；
- **效率与效果可以兼得**：StockMixer 在提升预测精度的同时大幅降低了计算成本，这对实盘交易中的快速推理非常重要。

StockMixer 之后，社区还出现了许多扩展工作（如结合频域特征的 StockMixer-ATFNet），这一"纯 MLP"方向在时序预测领域正在持续升温。

---

## 参考资料

1. Fan, J., & Shen, Y. (2024). *StockMixer: A Simple Yet Strong MLP-Based Architecture for Stock Price Forecasting*. Proceedings of the AAAI Conference on Artificial Intelligence, 38(8), 8380-8388. [DOI: 10.1609/aaai.v38i8.28681](https://doi.org/10.1609/aaai.v38i8.28681)
2. 官方代码仓库: [https://github.com/SJTU-DMTai/StockMixer](https://github.com/SJTU-DMTai/StockMixer)
3. arXiv 预印本: [https://arxiv.org/abs/2412.11192](https://arxiv.org/abs/2412.11192)
4. Sun, W. et al. (2025). *Research on deep learning model for stock prediction by integrating frequency domain and time series features*. Scientific Reports. [DOI: 10.1038/s41598-025-14872-6](https://doi.org/10.1038/s41598-025-14872-6)
