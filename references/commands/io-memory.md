# io-memory.md — IO 与内存/拓扑命令参考（资产层）

> **资产层**：拓扑观测、绑定、IO 分层计时等可复制命令。工具名为 Linux 生态专名，
> 属渐进披露层资产，不改变原则层的架构无关定位。
> 命令与 sysctl 名随内核版本演化，以安装版本实际输出为准。
>
> 来源：mohitmishra786/low-level-dev-skills、dendibakh/perf-book（条目经实读核验）
> 许可：MIT、CC0
> 核验日期：2026-10-02

## 1. 拓扑观测与绑定（NUMA / 亲和性）

```bash
# 查物理拓扑：NUMA 节点、核间缓存共享关系（绑核前必查）
lscpu -e
numactl --hardware

# 线程就近绑定：CPU 与内存都锁在本地节点
numactl --cpunodebind=0 --membind=0 ./prog

# 绑定到指定核集合
taskset -c 0-7 ./prog

# 观测线程实际落点：核间迁移说明存在跨节点访问（绑核决策依据）
top -H -p <pid>      # 按 P 排序看各线程落在哪个核
perf record -e migrations:T -a sleep 5    # 采样核间迁移事件
```

## 2. 大页（降 TLB 压力）

```bash
# 查看与预留大页
grep Huge /proc/meminfo
echo 512 | sudo tee /proc/sys/vm/nr_hugepages
# 透明大页模式（always/never 按负载实测取舍）
cat /sys/kernel/mm/transparent_hugepage/enabled
```

## 3. 中断绑定与分散

```bash
# 查中断分布，把设备中断绑到距离设备最近的核（IRQ 亲和性）
cat /proc/interrupts
echo <cpumask> | sudo tee /proc/irq/<N>/smp_affinity
# 软中断负载分散到多核（防单核热点；按内核文档开启 RPS/RFS 类机制）
```

## 4. IO 观测与分层计时

```bash
# 设备级：等待时间 vs 利用率（等待占比高→调调度/并发；利用率持续 100%→设备瓶颈）
iostat -x 1

# 分层计时：请求生成 → 调度器 → 驱动 → 硬件各阶段耗时分解
sudo blktrace -d /dev/sda -o trace -w 10
blkparse -i trace -d trace.bin && btt -i trace.bin

# 页缓存与直接 IO 的取舍观测（vhitcache 类统计随内核而异，先确认可用性）
```

## 5. 预读窗口与脏页写回

```bash
# 顺序读负载调大预读窗口（单位：512B 扇区数；按实测调整）
sudo blockdev --setra 4096 /dev/sda
blockdev --getra /dev/sda

# 脏页写回水位：在"批量写摊薄"与"突发写积压/断电丢失"间权衡
sysctl vm.dirty_ratio vm.dirty_background_ratio
# 需立即落盘的关键数据绕过页缓存：open(O_DIRECT)（应用层选项）
```

## 6. 页锁定与直连（DMA 场景，对应 guide Step 15）

```bash
# 应用层锁定内存页供 DMA 使用（mlock/分配器选项按语言生态实现）；
# 只锁真正传输的缓冲区——锁页占用不可换出内存。
```
