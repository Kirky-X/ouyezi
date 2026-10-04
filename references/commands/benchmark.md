# benchmark.md — 基准与复测命令参考（资产层）

> **资产层**：可直接复制的降噪命令、基准 harness 结构与 A/B 对比流程。
> 工具名为各生态专名，属渐进披露层资产，不改变原则层的架构无关定位。
> 命令版本敏感，以安装版本实际帮助为准。
>
> 来源：google/benchmark、wieslawsoltes/Performance-Skill、dendibakh/perf-ninja
>   （perf-ninja 仅吸收 harness 设计思想，其仓库无 LICENSE，未复制任何代码）
> 许可：Apache-2.0、MIT；perf-ninja 无许可声明
> 核验日期：2026-10-02

## 1. 降噪五旋钮（基准前先锁环境）

```bash
# ① 锁定 CPU 频率到性能档（防降频引入抖动）
sudo cpupower frequency-set --governor performance

# ② 关闭睿频/boost（趋势对比时避免频率漂移；按平台工具实际提供的方式）

# ③ 绑核（防迁移冷缓存）
taskset -c 2 ./bench

# ④ 提调度优先级（防背景抢占；chrt 需要权限）
sudo chrt -f 80 ./bench

# ⑤ 关闭 ASLR（地址布局随机化是复现性噪声源）
sudo echo 0 > /proc/sys/kernel/randomize_va_space     # 全局（用后恢复）
setarch $(uname -m) -R ./bench                        # 单次
```

复测结果仍不稳定时，按十四项噪声源逐项排查：散热降频、频率调节、后台进程、
虚拟化/共享宿主、NUMA 放置、GC 模式、输入突变、隐藏 IO、计时器精度、基准时长、
分支/缓存状态、进程亲和、杀毒/索引服务、JIT 分层切换。**禁止靠加迭代次数凑出想要的结论。**

## 2. 基准 harness 通用结构（先正确性，后跑分）

任何 A/B 对比 harness 都应包含四件套（设计思想源自 perf-ninja 课程，结构通用化）：

1. **正确性门禁先于计时**：优化版输出与黄金参考逐一比对（全输入域函数级比对），
   或对输出做哈希（MD5 类）与基线输出哈希比对；不一致直接失败退出，不进入跑分。
2. **基线与优化版同文件共存**：用编译期开关（`#ifdef SOLUTION` 类）让两个版本
   在同一构建系统里可切换，保证除被测改动外一切等价。
3. **双目标统一构建**：`validate`（正确性）与 `bench`（计时，输出机器可读结果如 JSON）
   两个构建目标，CI 先跑前者再跑后者。
4. **环境元数据随结果归档**：机器、频率策略、commit、运行次数与结果一起落盘，
   否则结果不可追溯（对接 experiment-plan.md 实验前置记录与 retest-report.md 复测报告模板）。

## 3. google/benchmark 速用

```cpp
#include <benchmark/benchmark.h>

static void BM_Work(benchmark::State& state) {
  for (auto _ : state) {
    DoWork();
    benchmark::DoNotOptimize(sink);   // 防止死代码消除（DCE）
  }
}
BENCHMARK(BM_Work)->Iterations(10);   // 固定迭代数，使两次对比工作量可比
BENCHMARK(BM_Work);
BENCHMARK_MAIN();
```

```bash
# 输出 JSON 供对比工具消费
./bench --benchmark_format=json --benchmark_out=after.json

# 对比：差值 (new-old)/|old| + Mann-Whitney U 检验显著性
# 注意：U 检验需要不少于 9 次重复才有意义
./bench --benchmark_repetitions=12
compare.py baseline.json after.json

# 随机交错多次运行可显著降低方差（官方实测可降约 40%，个案值量级参考）
./bench --benchmark_random_interleaving=true
```

**防 DCE 纪律**：编译器可能整体删除"结果未被消费"的基准循环（如循环内构造对象即丢弃），
使计时失真甚至趋近于零。用 `DoNotOptimize` 类语义把结果强制落点（各语言基准框架
均有等价物，如 JMH 的 `Blackhole.consume()`），并**剖析基准本身确认目标代码确为热点**
（"Always Measure One Level Deeper"）。

## 4. 多轮 A/B 对比流程

单轮对比不可信；逐轮"构建-运行-对比"并打印每轮加速比，观察轮间散布：

```bash
for i in $(seq 1 5); do
  git checkout -q main    && make -s && ./bench > base_$i.txt
  git checkout -q feature && make -s && ./bench > cand_$i.txt
done
# 对每轮 base_i vs cand_i 计算变化率，再对轮间散布复核（脚本按项目测试框架自行实现）
```

## 5. 交替配对（BABA）与排除纪律

- 对比顺序交替进行：`B A B A B A`（B=基线，A=候选），而非 `B B B A A A`——
  漂移（温升、缓存状态、背景负载）会被均匀分摊到两侧。
- **只按预声明的规则排除数据**（跑前写明何种情况作废重跑），禁止事后挑数据。
- 禁止只报最快一轮；以中位数/分布叙述，尾部看 P99（对接 guide Step 16 判据）。
