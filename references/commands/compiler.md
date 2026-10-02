# compiler.md — 编译器命令参考（资产层）

> **资产层**：向量化报告、PGO/BOLT 工具链等可复制命令。编译器选项名为 GCC/Clang 专名，
> 属渐进披露层资产，不改变原则层的架构无关定位；其他工具链按等价能力映射。
> 选项随编译器版本演化，以安装版本文档为准。
>
> 来源：mohitmishra786/low-level-dev-skills、dendibakh/perf-book（条目经实读核验）
> 许可：MIT、CC0
> 核验日期：2026-10-02

## 1. 优化等级与目标

```bash
# 发布高等级、调试低等级；精度敏感处逐文件锁级（见 guide Step 13）
gcc -O3 -march=<目标架构代> -o prog prog.c    # 交叉发布时显式指定目标特性
```

## 2. 自动向量化报告：先看编译器怎么说

```bash
# GCC：报告成功/失败的向量化及原因
gcc -O3 -fopt-info-vec -fopt-info-vec-missed -c prog.c

# Clang：转换序列与缺失原因
clang -O3 -Rpass=loop-vectorize -Rpass-missed=loop-vectorize -c prog.cpp
```

常见 blocker → 修复对照：

| 报告原因 | 修复方向 |
|---------|---------|
| 循环携带依赖（本迭代写、下迭代读） | 消依赖/拆循环/重排写入 |
| 可能别名（指针可能重叠） | `restrict` 标注（确认不重叠时）、下标化访问 |
| 循环次数未知/非常量 | 常量化、2 的幂、尾部循环拆分 |
| 循环体内分支/函数调用 | 条件外提、查表、内联小函数 |
| 混合类型/填充破坏装载宽度 | 成员同类型并对齐到装载宽度（见 guide Step 7） |

## 3. LTO

```bash
gcc  -O3 -flto -c a.c && gcc -O3 -flto a.o b.o -o prog   # GCC
clang -O3 -flto=thin -c a.cpp                            # Clang ThinLTO（并行、省内存）
```

代价：编译时间与内存上升；全程序分析解锁跨文件内联。

## 4. PGO 工具链

```bash
# ── GCC 插桩 PGO ──
gcc -O3 -fprofile-generate -o prog prog.c     # ① 插桩编译
./prog <代表性负载>                            # ② 真实负载跑采样（生成 .gcda）
gcc -O3 -fprofile-use -o prog prog.c          # ③ 用 profile 重编译

# ── Clang 插桩 PGO ──
clang -O3 -fprofile-instr-generate -o prog prog.cpp
LLVM_PROFILE_FILE=app.profraw ./prog <代表性负载>
llvm-profdata merge -output=app.profdata app.profraw
clang -O3 -fprofile-instr-use=app.profdata -o prog prog.cpp

# ── 采样 PGO（AutoFDO 类，免插桩，可直接在生产环境采集）──
perf record -b -e cycles:u ./prog             # 需 LBR 支持
create_llvm_prof --binary=prog --out=code.prof   # perf 数据转换为编译器可消费格式
clang -O3 -fprofile-sample-use=code.prof -o prog2 prog.cpp

# ── BOLT：后链接优化（重排已编译二进制，指令缓存/ITLB 受限程序收益最大）──
perf record -e cycles:u -j any,u -o perf.data ./prog
llvm-bolt prog -o prog.bolt -data=perf.data -reorder-blocks=cache+ -split-functions

# ── 收益验证 ──
perf stat -e instructions,cycles ./prog       # 对比 PGO 前后 IPC
```

工程判据（数字均为量级参考）：

- 插桩版运行时开销通常 **5–10 倍**，采.profile 不能贪长；profile 随代码演化**过期**，需重采。
- 训练负载必须有代表性：编译器"盲目"信任 profile，改进一个场景可能劣化另一个；
  多场景 profile 可合并采集。
- 适用性：指令取指（Frontend）受限负载收益最高（个案最高 ~30% 量级）；
  纯计算受限负载可能**零收益**——先剖析分桶，再决定是否值得引入 PGO 流程。
- BOLT 与插桩 PGO/LTO 叠加常见再获 **+5–10%**（个案量级）。

## 5. 剖析配合项

```bash
-fno-omit-frame-pointer    # 保留帧指针，供 perf fp 回溯（见 profiling.md）
-finline-limit=<n>         # 内联预算调节（过度内联膨胀指令缓存，见 guide Step 13）
```
