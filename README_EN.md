# Ouyezi — General-Purpose Code Performance Optimization Methodology Skill

> A performance optimization methodology skill distilled and de-identified from public performance-optimization technical documents: a hotspot-driven workflow (baseline measurement → profiling → optimization → re-test verification), cross-platform and architecture-agnostic — all chip/vendor-specific details are deliberately removed, keeping only the principles that hold on any hardware platform.

English | [中文](README.md)

[![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)] [![Dependencies](https://img.shields.io/badge/dependencies-none-blue?style=flat-square)] [![Evals](https://img.shields.io/badge/evals-5%20pos%20%2B%202%20neg-orange?style=flat-square)](#-测试与验证)

## ✨ Features

- **6-step Process** ([SKILL.md](SKILL.md)): confirm the scenario and bottleneck type → profile and locate hotspots → select and apply optimization techniques → re-test to verify and quantify gains → numerical precision and determinism checks → runtime tuning and archiving
- **21-step detailed guide** ([references/guide.md](references/guide.md)): each Process step is annotated with "Guide Steps N–M"; single-capability tasks can enter at the corresponding Step and wrap up in place, without walking all 21 steps. Step 3 carries a **symptom → framework → technique diagnostic routing** (TMA four-bucket skeleton); Steps 16–17 carry **quantified re-test criteria** (dual-threshold significance / minimum repetitions / noise floor)
- **17-chapter principles catalog** ([references/methodology.md](references/methodology.md)): workflow & profiling / data layout / false sharing & prefetching / vectorization / loops & ILP / parallel decomposition & concurrency / pipeline orchestration / compiler optimization / IO & data movement / algorithms & data structures / floating-point precision / precision-performance trade-offs / verification methodology / meta-principles / distributed & services (index-level) / frontend rendering (index-level) / continuous performance gating (CI)
- **Asset layer**: [references/commands/](references/commands/) — copy-ready profiling/benchmark/compiler/IO commands with threshold tables; [references/templates/](references/templates/) — experiment plan / retest report / archive checklist templates (tool-specific names exist only in the asset layer; the principle layer stays architecture-agnostic)
- **Topics covered**: cache-friendly design, SIMD vectorization ideas, loop transformations, branch-misprediction cost model, machine-code layout & PGO toolchain, NUMA awareness, optimization levels/inlining/LTO/PGO, microbenchmark methodology, precision optimization and determinism, CI performance gating
- **Depth map**: §1–§14 and §17 are method/criteria level; §15/§16 and guide Step 20 are index-level (directions and trade-offs only)
- **Self-contained corpus**: the skill itself works without the source corpus; the corpus (15 MB of PDFs/images) is not shipped with the package

```mermaid
flowchart LR
    S1[1 场景与瓶颈] --> S2[2 剖析定位热点] --> S3[3 应用优化手段] --> S4[4 复测量化收益] --> S5[5 精度与确定性] --> S6[6 运行时调优与归档]
    S4 -.负面优化回滚.-> S3
```

## 📦 Installation

No external dependencies, no scripts.

```bash
# Option 1: workspace sync script (auto-deploy, corpus/ excluded)
bash scripts/sync-skills.sh ouyezi

# 方式二：手动复制到技能目录（目标目录名 ouyezi）
cp -r ouyezi/ ~/.zcode/skills/ouyezi/
# Option 3: Remote install (GitHub repo)
npx skills add Kirky-X/ouyezi --agent claude-code -y
```

## 🚀 Quick Start

```bash
# Verify deployment (expect SKILL.md / references/guide.md / references/methodology.md / evals/)
ls ~/.zcode/skills/ouyezi/

# 校验 evals 用例数（→ 7 / expect 7）
python3 -c "import json; print(len(json.load(open('evals/evals.json'))['evals']))"

# 查看完整原则目录
sed -n '/^## /p' references/methodology.md
```

Trigger examples (natural language spoken to the agent):

- "Our C++ service can't push throughput higher under load — the CPU is pegged but latency is high; help me optimize performance" → walk the full 6-step Process
- "The matrix multiplication kernel is too slow; row-major is already 2.3x faster than column-major — how to continue" → enter at the guide's cache/loop-transformation Step
- "The same distributed numerical program gives different results with different process counts — is it a bug" → enter the precision & determinism Step

## ✅ Tests & Verification

This skill has no scripts and no unit tests; verification is **SKILL.md reference integrity + evals** (measured 2026-10-02):

| Check | Measured result |
| ----- | --------------- |
| [references/guide.md](references/guide.md) step count | `grep -c '^## Step'` = **21** (Step 1 – Step 21, contiguous) |
| [references/methodology.md](references/methodology.md) chapter count | **17** chapters (§1 – §17, matching the §11/§12 referenced in SKILL.md) |
| Files referenced by SKILL.md | references/guide.md, references/methodology.md, references/commands/, references/templates/ all exist, 0 missing |
| [evals/evals.json](evals/evals.json) | **7 cases: 5 positive, 2 negative** (id 4 negative — CRUD development without performance requirements should not trigger; id 7 negative — AVX-512 instruction-set-specific optimization should not trigger; id 5/6 are capability checks with file inputs) |
| Engineering lint | `python3 scripts/skill_lint.py .` passes (0 FAIL) |

## 📁 Directory Structure

```
ouyezi/
├── SKILL.md                   # 6-step Process + Setup + Mode line
├── references/
│   ├── guide.md               # 21-step detailed guide (diagnostic routing / re-test criteria / asset cross-refs)
│   ├── methodology.md         # 17-chapter principles catalog (§15/§16 index-level)
│   ├── commands/              # asset layer: profiling/benchmark/compiler/IO commands & thresholds
│   │   ├── profiling.md
│   │   ├── benchmark.md
│   │   ├── compiler.md
│   │   └── io-memory.md
│   └── templates/             # asset layer: experiment plan / retest report / archive checklist
│       ├── experiment-plan.md
│       ├── retest-report.md
│       └── archive-checklist.md
├── evals/
│   └── evals.json             # 7 cases (5 positive, 2 negative, with file inputs)
└── corpus/                    # source corpus (~15 MB, not distributed, excluded on deploy)
    └── images/
```

## 🔮 Boundaries

From the [SKILL.md](SKILL.md) trigger description, do **not** trigger in the following cases:

- **Vendor-specific optimization** bound to a specific hardware instruction set (NEON / SVE / AVX, etc.)
- Configuration of vendor-proprietary tools or SDKs
- Ordinary feature development **without performance requirements**
- Security review, dependency analysis

Methodology red lines: outputs must not be bound to any specific instruction set, chip, or proprietary tool; no optimization without a baseline, no convergence without re-testing.

## 📄 License & Attribution

- License: MIT (per the declaration in the [SKILL.md](SKILL.md) frontmatter; the [LICENSE](LICENSE) file is included in the directory)
- The methodology was distilled and de-identified from multiple public performance-optimization technical documents; it contains no vendor- or chip-specific information. Asset-layer commands and templates were absorbed from open-source projects (sources and licenses noted in each file's header: MIT/CC0/Apache-2.0; perf-ninja has no LICENSE — only design ideas were absorbed, no code copied)
- The source corpus lives only in the development workspace `corpus/`; it is excluded by the deployment script and not distributed with the skill
