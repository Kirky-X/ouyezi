# Ouyezi — General-Purpose Code Performance Optimization Methodology Skill

> A performance optimization methodology skill distilled and de-identified from the Kunpeng series of technical documents: a hotspot-driven workflow (baseline measurement → profiling → optimization → re-test verification), cross-platform and architecture-agnostic — all chip/vendor-specific details are deliberately removed, keeping only the principles that hold on any hardware platform.

English | [中文](README.md)

[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)] [![Dependencies](https://img.shields.io/badge/dependencies-none-blue?style=flat-square)] [![Evals](https://img.shields.io/badge/evals-3%20pos%20%2B%201%20neg-orange?style=flat-square)](#-测试与验证)

## ✨ Features

- **6-step Process** ([SKILL.md](SKILL.md)): confirm the scenario and bottleneck type → profile and locate hotspots → select and apply optimization techniques → re-test to verify and quantify gains → numerical precision and determinism checks → runtime tuning and archiving
- **21-step detailed guide** ([kp-guide.md](kp-guide.md)): each Process step is annotated with "Guide Steps N–M"; single-capability tasks can enter at the corresponding Step and wrap up in place, without walking all 21 steps
- **14-chapter principles catalog** ([references/methodology.md](references/methodology.md)): data layout / false sharing & prefetching / vectorization / loops & ILP / parallel decomposition / pipeline orchestration / compiler optimization / IO / algorithms / floating-point precision / precision-performance trade-offs / verification methodology / meta-principles
- **Topics covered**: cache-friendly design, SIMD vectorization ideas, loop transformations, NUMA awareness, optimization levels/inlining/LTO/PGO, precision optimization and determinism
- **Self-contained corpus**: the skill itself works without the `kp/` source corpus; the corpus (15 MB of PDFs/images) is not shipped with the package

```mermaid
flowchart LR
    S1[1 场景与瓶颈] --> S2[2 剖析定位热点] --> S3[3 应用优化手段] --> S4[4 复测量化收益] --> S5[5 精度与确定性] --> S6[6 运行时调优与归档]
    S4 -.负面优化回滚.-> S3
```

## 📦 Installation

No external dependencies, no scripts.

```bash
# 方式一：工作区同步脚本（自动部署，kp/ 语料被排除）
bash scripts/sync-skills.sh ouyezi

# 方式二：手动复制到技能目录（目标目录名 ouyezi）
cp -r ouyezi/ ~/.zcode/skills/ouyezi/
```

## 🚀 Quick Start

```bash
# 验证部署结果（应看到 SKILL.md / kp-guide.md / references/ / evals/）
ls ~/.zcode/skills/ouyezi/

# 校验 evals 用例数（→ 4）
python3 -c "import json; print(len(json.load(open('evals/evals.json'))['evals']))"

# 查看完整原则目录
sed -n '/^## /p' references/methodology.md
```

Trigger examples (natural language spoken to the agent):

- "Our C++ service can't push throughput higher under load — the CPU is pegged but latency is high; help me optimize performance" → walk the full 6-step Process
- "The matrix multiplication kernel is too slow; row-major is already 2.3x faster than column-major — how to continue" → enter at the guide's cache/loop-transformation Step
- "The same distributed numerical program gives different results with different process counts — is it a bug" → enter the precision & determinism Step

## ✅ Tests & Verification

This skill has no scripts and no unit tests; verification is **SKILL.md reference integrity + evals** (measured 2026-09-13):

| Check | Measured result |
| ----- | --------------- |
| [kp-guide.md](kp-guide.md) step count | `grep -c '^## Step'` = **21** (Step 1 – Step 21, contiguous) |
| [references/methodology.md](references/methodology.md) chapter count | **14** chapters (§1 – §14, matching the §11/§12 referenced in SKILL.md) |
| Files referenced by SKILL.md | kp-guide.md, references/methodology.md, and kp/ all exist, 0 missing |
| [evals/evals.json](evals/evals.json) | **4 cases: 3 positive, 1 negative** (id 4 is the negative case — CRUD API development should not trigger the performance workflow) |

## 📁 Directory Structure

```
ouyezi/
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

## 🔮 Boundaries

From the [SKILL.md](SKILL.md) trigger description, do **not** trigger in the following cases:

- **Vendor-specific optimization** bound to a specific hardware instruction set (NEON / SVE / AVX, etc.)
- Configuration of vendor-proprietary tools or SDKs
- Ordinary feature development **without performance requirements**
- Security review, dependency analysis

Methodology red lines: outputs must not be bound to any specific instruction set, chip, or proprietary tool; no optimization without a baseline, no convergence without re-testing.

## 📄 License & Attribution

- License: MIT (per the declaration in the [SKILL.md](SKILL.md) frontmatter; no LICENSE file is included in the directory for now)
- The methodology was distilled and de-identified from 5 public Kunpeng technical documents: GPU Application Optimization Whitepaper, Performance Optimization Ten Techniques, Precision Optimization Three Techniques, Programming and Tuning Guide, and Computational Precision Whitepaper
- The source corpus lives only in the development workspace `kp/`; it is excluded by the deployment script and not distributed with the skill
