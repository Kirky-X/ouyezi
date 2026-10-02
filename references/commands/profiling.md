# profiling.md — 剖析命令参考（资产层）

> **资产层**：本文件收录可直接复制的剖析命令与判读阈值表。工具名为 Linux 生态专名，
> 属渐进披露层资产，不改变 SKILL.md 原则层的架构无关定位。
> 命令与事件名随内核/工具版本演化，以安装版本的 `--help` 与 `perf list` 实际输出为准。
>
> 来源：mohitmishra786/low-level-dev-skills、dendibakh/perf-book（条目经实读核验）
> 许可：MIT、CC0
> 核验日期：2026-10-02

## 1. 权限准备

```bash
# perf 默认需要 root 或 perf_event_paranoid ≤ 1
cat /proc/sys/kernel/perf_event_paranoid
# 2 = 仅用户态 CPU 统计；1 = 用户+内核；0 = 全部；-1 = 无限制

# 会话级放宽（临时）
sudo sysctl -w kernel.perf_event_paranoid=1

# 持久化
echo 'kernel.perf_event_paranoid=1' | sudo tee /etc/sysctl.d/99-perf.conf
sudo sysctl -p /etc/sysctl.d/99-perf.conf
```

## 2. 基本流程：stat → record → report → annotate

```bash
# ① 全局计数概览（IPC、缓存缺失、分支缺失一次拿全）
perf stat -e instructions,cycles,cache-misses,branch-misses ./prog

# ② 采样记录热点（-F 999：每秒 999 次采样；编译需带调试信息）
perf record -F 999 -g ./prog

# ③ 按累计开销排序看热点函数
perf report

# ④ 逐行归因到源码（定位热点内部分布）
perf annotate
```

附加在编译期的配合项：`-fno-omit-frame-pointer`（保帧指针）或带 `-g`（调试符号），
否则调用栈回溯质量差（见 §8 速查表）。

## 3. 硬件计数器判读阈值表

| 指标 | 计算式 | 健康参考 | 需要关注 |
|--------|---------|---------|-----------|
| IPC | instructions / cycles | > 2.0（现代 x86 量级） | < 1.0 |
| L1 miss 率 | L1-misses / L1-accesses | < 1% | > 5% |
| LLC miss 率 | LLC-misses / LLC-accesses | < 1% | > 10% |
| 分支 miss 率 | branch-misses / branches | < 1% | > 5% |
| MPKI | misses per 1K instructions | — | L3 MPKI > 10 ≈ 内存受限 |

```bash
# MPKI（每千条指令缺失数）计算：先取计数，再手工相除
perf stat -e instructions,LLC-load-misses ./prog
# MPKI = LLC-load-misses / (instructions / 1000)
```

阈值为经验量级参考，非普适承诺；不同微架构与负载形态差异大，以自身基线复测为准。

## 4. 调用栈回溯三种方式取舍

```bash
perf record -F 999 --call-graph=fp ./prog      # 帧指针：开销最低，需编译期保留 fp
perf record -F 999 --call-graph=dwarf ./prog   # DWARF：无需 fp，栈大时可限深 dwarf,512
perf record -F 999 --call-graph=lbr ./prog     # LBR：硬件栈回溯，低开销，平台支持有限
```

## 5. off-CPU 与调度观测

```bash
# off-CPU 采样：时间花在"等待"上（锁/IO/页错误/调度）
perf record -e sched:sched_switch -ag sleep 10

# 调度延迟：识别被延迟调度的线程（过订阅/定时器配置问题）
perf sched record -- sleep 10
perf sched latency --sort max

# strace 找阻塞点：低 CPU 利用率时按调用序列 + 单次耗时定位停在哪类调用
strace -T -tt -p <pid>
```

## 6. 伪共享检测

```bash
# 症状计数器（事件名随平台而异，先 perf list 确认存在）
perf stat -e machine_clears.memory_ordering,mem_load_l3_hit_retired.xsnp_hitm ./prog

# 专用于缓存行争用的对比工具：先按 TMA 看到争用类占比异常，再用 c2c 定位到具体缓存行与地址
perf c2c record ./prog
perf c2c report
```

配合修复写法（填充/对齐隔离）见 methodology §3 与 guide Steps 5–6。

## 7. 字段级数据布局剖析

Linux 内核 ≥ 6.8 的 perf 支持按结构体字段归因采样，用于发现"哪些字段被一起访问"
（重排/填充/打包的依据）：

```bash
perf mem record ./prog
perf annotate --data-type
```

## 8. 常见问题速查表

| 问题 | 原因 | 修复 |
|---------|------|-----|
| `Permission denied` | perf_event_paranoid 过高 | 调低 paranoid 或用 sudo |
| 调用栈大量 `[unknown]` | 缺帧指针或调试信息 | `-fno-omit-frame-pointer` 重编译，或 `--call-graph=dwarf` |
| 到处是 `[kernel]` | 内核符号不可见 | sudo 采样；安装内核 dbgsym；`kptr_restrict=0` |
| `No kallsyms` | 内核符号不可用 | `echo 0 | sudo tee /proc/sys/kernel/kptr_restrict` |
| 短程序采样报告为空 | 程序退出太快 | 提高 `-F` 或让工作负载持续更久 |
| DWARF 回溯很慢 | 栈数据量大 | 限深 `--call-graph dwarf,512` |
