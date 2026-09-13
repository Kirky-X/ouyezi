# Ouyezi（欧冶子）— 通用代码性能优化方法论技能

[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)] [![Dependencies](https://img.shields.io/badge/dependencies-none-blue?style=flat-square)] [![Evals](https://img.shields.io/badge/evals-3%20pos%20%2B%201%20neg-orange?style=flat-square)](#-测试与验证)

中文 | [English](README_EN.md)

> 从鲲鹏系列技术文档提炼并脱敏的性能优化方法论技能：热点驱动工作流（基线测量 → 剖析定位 → 优化 → 复测验证），跨平台、架构无关——刻意去除所有芯片/厂商专属细节，只保留在任何硬件平台上成立的原则。

## ✨ 功能特性

- **6 步 Process**（[SKILL.md](SKILL.md)）：确认场景与瓶颈类型 → 剖析定位热点 → 选择并应用优化手段 → 复测验证与量化收益 → 数值精度与确定性检查 → 运行时调优与归档
- **21 步详细指南**（[kp-guide.md](kp-guide.md)）：Process 各步标注"指南 Steps N–M"，单能力任务可从对应 Step 进入、就地收尾，无需走完全部 21 步
- **14 章原则目录**（[references/methodology.md](references/methodology.md)）：数据布局 / 伪共享与预取 / 向量化 / 循环与 ILP / 并行分解 / 流水线编排 / 编译器优化 / IO / 算法 / 浮点精度 / 精度-性能权衡 / 验证方法论 / 元原则
- **覆盖主题**：缓存友好性设计、SIMD 向量化思想、循环变换、NUMA 意识、优化等级/内联/LTO/PGO、精度优化与确定性
- **语料自洽**：skill 本体不依赖 `kp/` 源语料即可工作；语料（15 MB PDF/图）不随包分发

```mermaid
flowchart LR
    S1[1 场景与瓶颈] --> S2[2 剖析定位热点] --> S3[3 应用优化手段] --> S4[4 复测量化收益] --> S5[5 精度与确定性] --> S6[6 运行时调优与归档]
    S4 -.负面优化回滚.-> S3
```

## 📦 安装

无外部依赖、无脚本。

```bash
# 方式一：工作区同步脚本（自动部署，kp/ 语料被排除）
bash scripts/sync-skills.sh ouyezi

# 方式二：手动复制到技能目录
cp -r ouyezi/ ~/.zcode/skills/ouyezi/
```

## 🚀 快速开始

```bash
# 验证部署结果（应看到 SKILL.md / kp-guide.md / references/ / evals/）
ls ~/.zcode/skills/ouyezi/

# 校验 evals 用例数（→ 4）
python3 -c "import json; print(len(json.load(open('evals/evals.json'))['evals']))"

# 查看完整原则目录
sed -n '/^## /p' references/methodology.md
```

触发示例（对 agent 的自然语言）：

- 「我们的 C++ 服务压测吞吐上不去、CPU 打满但延迟高，帮我优化性能」→ 走完整 6 步 Process
- 「矩阵乘法内核太慢，行优先已比列优先快 2.3 倍，怎么继续」→ 从指南缓存/循环变换 Step 进入
- 「同一分布式数值程序不同进程数结果不一样，是 bug 吗」→ 进入精度与确定性 Step

## ✅ 测试与验证

本 skill 无脚本、无单测，验证方式为 **SKILL.md 引用完整性 + evals**（2026-09-13 实测）：

| 检查项 | 实测结果 |
| ------ | -------- |
| [kp-guide.md](kp-guide.md) 步骤数 | `grep -c '^## Step'` = **21**（Step 1 – Step 21 连续） |
| [references/methodology.md](references/methodology.md) 章节数 | **14** 章（§1 – §14，与 SKILL.md 引用的 §11/§12 对应） |
| SKILL.md 引用的文件 | kp-guide.md、references/methodology.md、kp/ 均存在，0 缺失 |
| [evals/evals.json](evals/evals.json) | **4 用例：3 正 1 负**（id 4 为负例——CRUD 接口开发不应触发性能流程） |

## 📁 目录结构

```
ouyezi/                        # 欧冶子
├── SKILL.md                   # 6 步 Process + Setup + Mode line
├── kp-guide.md                # 21 步详细指南（含基线/剖析/精度/归档全部细节）
├── references/
│   └── methodology.md         # 14 章跨平台原则目录
├── evals/
│   └── evals.json             # 4 用例（3 正 1 负）
└── kp/                        # 源语料（约 15 MB，不随包分发，部署时被排除）
    ├── 鲲鹏*.pdf|md           # 5 部文档：GPU 优化白皮书 / 十板斧 / 精度三板斧 / 编程与调优指南 / 计算精度白皮书
    ├── images/                # 文档配图
    └── convert_kp.py          # PDF→Markdown 转换脚本
```

## 🔮 边界

来自 [SKILL.md](SKILL.md) 触发描述，以下场景**不要**触发：

- 绑定特定硬件指令集（NEON / SVE / AVX 等）的**专属优化**
- 厂商专有工具或 SDK 的配置
- **不含性能诉求**的普通功能开发
- 安全审查、依赖分析

方法论红线：输出不绑定任何特定指令集、芯片或专有工具；没有基线不优化，没有复测不收敛。

## 📄 License 与归属

- License：MIT（以 [SKILL.md](SKILL.md) frontmatter 声明为准，目录内暂未附带 LICENSE 文件）
- 方法论提炼自 5 部鲲鹏公开技术文档并脱敏：GPU 应用优化白皮书、性能优化十板斧、精度优化三板斧、编程与调优指南、计算精度白皮书
- 源语料仅存于开发工作区 `kp/`，经部署脚本排除，不随 skill 分发
