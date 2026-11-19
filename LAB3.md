# 实验 3 记录：第 4 组

评分时依据本文件和 `runs/` 中的原始数据核对。本文件中的每一个数值，都须能在某个 `runs/<名字>/` 的文件中找到：表格用 `scripts/table.sh` 生成或从 `summary.md` 中复制后粘贴，其他数值注明出处（例如“`runs/b2-cpu/summary.md` 缺口表第 1 行”）。不要只贴截图。

标有“预测”的小节须在对应的测量之前写好，写好后不要修改。预测错误不扣分，与实测不一致之处在分析中讨论。

## 1. 基本信息

| 项 | 内容 |
|---|---|
| 小组 | 第 4 组 |
| 仓库地址 | https://github.com/Icecube2017/ops-lab3-4 |
| 主机 | 华为云 2 vCPUs 2 GiB |
| CPU | `pc-i440fx-4.1  CPU @ 2.0GHz`, `2` Core |
| 内存与 swap | `1775 MB`, 无 |
| 操作系统、内核、systemd | `Ubuntu 24.04.4 LTS`, `6.8.0-136-generic`, `systemd 255 (255.4-1ubuntu8.16)` |
| cgroup | `cgroup2fs`, `cpuset cpu io memory hugetlb pids rdma misc`,  |
| Go、Prometheus、node_exporter | `go 1.27.3`, `prometheus 3.13.4`, `node_exporter 1.12.1` |
| 分核 | `enrolld: 1`, `loadgen、Prometheus、node_exporter: 0` |

## 2. 任务 A：配额，节流与 OOM

### 2.1 A1 配额算术

预测：配额 0.5 核、周期 100 ms，即每个周期有 0.5 × 100 = 50 ms 的 CPU 时间。按 L-09 第 12 页的算术，n 个线程同时忙时每个线程每周期运行 50/n ms、其余时间被停：1 个线程运行 50 ms、被停 50 ms；2 个线程时每个线程运行 25 ms、被停 75 ms（L-09 的前提：n 个线程同时在跑、核数足够）。本机 enrolld 按表 1 绑在 1 个核上（`AllowedCPUs=1`）；若 A1 的 cgroup 沿用这个绑核，2 个线程不能真正并行，墙钟上仍是各跑 25 ms、各停 75 ms，但 `throttled_usec` 只累计节流，折算出的“每线程被停”会小于 75 ms；若不绑核、两个线程各占一个核，实测应接近 75 ms。

GOMAXPROCS：Go 1.25 起默认取 CPU 核数 2、绑核数 1、cgroup 限额向上取整 ⌈0.5⌉ = 1 三者中的最小值，得 1；因主机有 2 个核（不属于“只有 1 个核”的例外），“不小于 2”的下限生效，所以 enrolld 的 GOMAXPROCS = 2。即 0.5 核的配额并不会把 GOMAXPROCS 压到 1；`-burn 2` 的两个忙 goroutine 可以各占一个 P。

两次 `-burn` 的输出：

```text
root@ecs-e0a4:/home/sample/lab3# for n in 1 2; do sudo systemd-run --quiet --scope -p CPUQuota=50% bin/enrolld -burn $n -for 10s; done
cgroup：/sys/fs/cgroup/system.slice/run-r05bb12285ba040d59baf02fbc897010b.scope
cpu.max：配额 0.50 核，周期 100ms（每个周期最多 50 ms CPU 时间）
GOMAXPROCS=2，忙线程 1 个，计数 10.0 秒
实际用核 0.50；周期 100 个，被节流 100 个（100%）；被停共 5.01 秒（按 CPU 累计）
每个线程每个周期被停：实测 50.1 ms，算术 50.0 ms
cgroup：/sys/fs/cgroup/system.slice/run-rb6a9109e8bef435f82b2ff07262f692a.scope
cpu.max：配额 0.50 核，周期 100ms（每个周期最多 50 ms CPU 时间）
GOMAXPROCS=2，忙线程 2 个，计数 10.0 秒
实际用核 0.50；周期 100 个，被节流 100 个（100%）；被停共 14.98 秒（按 CPU 累计）
每个线程每个周期被停：实测 74.9 ms，算术 75.0 ms
```

| 线程数 | 周期数 | 被节流的周期 | 实际用核 | 每线程每周期被停（实测） | 算术 |
|---|---|---|---|---|---|
| 1 | 100 | 100 | 0.50 | 50.1ms | 50.0ms |
| 2 | 100 | 100 | 0.50 | 74.9ms | 75.0ms |

### 2.2 A2 节流与 p99

本组写的节流比例查询（与 `deploy/queries-red.txt` 一致）：

```text
throttle_ratio      sum(rate(cgroup_cpu_throttled_periods_total{job="enrolld"}[15s])) / sum(rate(cgroup_cpu_periods_total{job="enrolld"}[15s]))
```

配额 100% 时测得的每请求 CPU：4.21 ms（`runs/a2-base/red.csv` 第 2 阶段，原值 4.20664；该次 `meta.txt` 中 `work=2000000`）；据此选定的到达率：95 请求/秒，平均约 0.40 核（60 请求/秒时实测 0.266 核，按 4.21 ms/请求折算，0.4 核需约 95 请求/秒）。

预测：会变，而且 p99 会明显变大，但前提是到达率真的落在 0.4 核。a2-base 实测每请求约 4.21 ms（60 请求/秒，`runs/a2-base/red.csv` 第 2 阶段），60 请求/秒只用了约 0.27 核；按 4.21 ms/请求，0.4 核对应约 95 请求/秒，即每 100 ms 周期平均只有约 9.5 个请求。每个请求 4.2 ms，12 个请求就是 50.5 ms，超过 50 ms 的周期配额；泊松到达下每周期 ≥12 个请求的概率约 25%，被节流后的积压还会带到下一周期，因此预计 `throttle_ratio` 约 25%～35%，远高于初版按 0.5 ms/请求估的 2%～5%。p99 会从 100% 配额下的约 10 ms 量级（60 请求/秒实测 10.06 ms，95 请求/秒预计相近）升到约 60～100 ms（被节流的周期整个 cgroup 停到下一周期，在途请求最多多等约 100 ms；跨两个周期时 p999 可超过 100 ms）。平均用量仍约 0.4 核，低于 0.5 核配额。

各次实验的数据（`scripts/table.sh runs/<名字> ok_rps p50_ms p99_ms server_p99_ms gomaxprocs cg_limit_cores cg_cpu_cores throttle_ratio throttled_s_per_s`，每次一张表）：

**配额 100%（`runs/a2-q100`，95 请求/秒）。** a2-base 只压到 60 请求/秒（`runs/a2-base/meta.txt` 的 `-rates 30,60`），不能作 95 请求/秒的对照，因此按同一到达率另跑了这一次：

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.91 | 11.12 | 9.96062 | 2 | 1 | 0.41288 | 0 | 0 |
| 2 | 95 | 91.6 | 6.88 | 10.85 | 9.96576 | 2 | 1 | 0.37259 | 0 | 0 |
| all | - | 94.8 | 6.90 | 11.05 | - | - | - | - | - | - |

**配额 50%（`runs/a2-q50`，95 请求/秒）：**

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.95 | 39.90 | 41.6094 | 2 | 0.5 | 0.411741 | 0.262411 | 0.0713829 |
| 2 | 95 | 91.6 | 6.95 | 43.73 | 31.2412 | 2 | 0.5 | 0.373636 | 0.18705 | 0.0387277 |
| all | - | 94.8 | 6.95 | 42.40 | - | - | - | - | - | - |

**改法一：配额改为 60%（`runs/a2-q60`，95 请求/秒）。** 按 L-09 第 13 页的“峰值 ÷ 0.7”：`runs/a2-q100/red.csv` 第 1 阶段 `cg_cpu_cores` 为 0.41288，0.41288 ÷ 0.7 ≈ 0.59，向上取 0.6 核（60%）：

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 6.89 | 19.31 | 17 | 2 | 0.6 | 0.412476 | 0.0857143 | 0.0152754 |
| 2 | 95 | 91.6 | 6.90 | 11.10 | 9.98158 | 2 | 0.6 | 0.374049 | 0.0285714 | 0.00503714 |
| all | - | 94.8 | 6.90 | 16.04 | - | - | - | - | - | - |

**改法二：配额仍为 50%，`GOMAXPROCS=1`（`runs/a2-q50-g1`，95 请求/秒）。** 重启后 `runs/a2-q50-g1/meta.txt` 的 `/config` 中 `gomaxprocs` 为 1：

| step | target_rps | ok_rps | p50_ms | p99_ms | server_p99_ms | gomaxprocs | cg_limit_cores | cg_cpu_cores | throttle_ratio | throttled_s_per_s |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 95 | 98.0 | 8.45 | 55.96 | 45.2553 | 1 | 0.5 | 0.402811 | 0.307143 | 0.0583715 |
| 2 | 95 | 91.6 | 8.38 | 37.45 | 31.6842 | 1 | 0.5 | 0.364737 | 0.157143 | 0.0256696 |
| all | - | 94.8 | 8.41 | 48.15 | - | - | - | - | - | - |

**与预测的对照。** 节流比例与预测接近：第 1 阶段 `throttle_ratio` 为 26.24%，落在预测的 25%～35%；第 2 阶段为 18.71%，偏低是因为 `-rates 95,95` 下第 1、2 阶段实际 `ok_rps` 分别为 98.0 和 91.6（`runs/a2-q50/steps.csv`），每周期平均请求数从 9.8 降到 9.16，按泊松分布超过 12 个请求的概率相应从约 28% 降到约 21%，与实测的 26.24% 和 18.71% 同向。p99 则只从 100% 时的 11.12/10.85 ms（`runs/a2-q100/steps.csv`）升到 39.90/43.73 ms（`runs/a2-q50/steps.csv`），没有到预测的 60～100 ms。原因是被节流的周期并非整段停满 100 ms：`runs/a2-q50/metrics-start.txt` 到 `metrics-end.txt` 之间，`cgroup_cpu_throttled_periods_total` 从 0 增到 93，`cgroup_cpu_periods_total` 从 2729 增到 3131，`cgroup_cpu_throttled_seconds_total` 从 0 增到 2.152476，平均每个被节流周期对应约 23 ms 的累计停时（该指标按 CPU 分别累计，见 `metrics-end.txt` 的 HELP 行，墙钟停时更短）；且 `inflight_avg` 只有 0.67～0.73（`runs/a2-q50/red.csv`），在途请求少，积压不会把等待推满一个周期。`max_ms` 69.01/76.97 ms 才接近一个周期，说明接近满周期的停落在比 p99 更靠后的尾部。按 `runs/a2-q50/meta.txt` 的 SLO 阈值 50 ms，50% 时 p99 仍在该阈值内，但 `slo_good` 已从 1.0000 降到 0.9959/0.9934（`steps.csv`），即约 0.4%～0.7% 的请求超过阈值。结论是：平均用量 0.41 核虽低于配额 0.5 核，p99 仍约为 100% 时的 3.8 倍（42.40 ÷ 11.05），代价取决于每周期需求超过 50 ms 的频率和超出量。

**改法一的效果与代价。** p99 降到 19.31/11.10 ms，`throttle_ratio` 降到 8.57%/2.86%（`runs/a2-q60/red.csv`），而实际用量仍是 0.412/0.374 核，几乎不变；把实测的阶段用量 0.41288 核当作峰值，它占 0.6 核配额的 0.41288 ÷ 0.6 ≈ 69%，正好落在 70% 容量线内。代价是配额从 0.5 核提高到 0.6 核：本机 2 核，若没有这 0.1 核的空闲，提高配额只是把节流转给同机的 loadgen 等进程。它适合峰值可预测、宿主机仍有余量的场景；按“峰值 ÷ 0.7”定配额，前提是峰值本身测得准。

**改法二的效果与代价。** 合计 p99 为 48.15 ms，反而高于 50% 的 42.40 ms（两者 `steps.csv`）；p50 从 6.95 升到 8.41 ms，`sched_wait_p99` 从 6.50/4.99 升到 36.59/15.58 ms（`red.csv`），每请求 CPU 仍为 4.03/4.05 ms（`cpu_ms_per_req`），说明 CPU 总量没有减少，变的是等待的位置：原来等 cgroup 周期结束，现在先等单 P 上的排队。本机 loadgen 绑在核 0、enrolld 允许核 0,1（`runs/a2-q50-g1/meta.txt`），单 P 更难避开 loadgen，可能放大了 `sched_wait`。两个 20 s 阶段波动不小（第 2 阶段 37.45 ms 反而低于 50% 的 43.73 ms），但合计仍不支持“GOMAXPROCS=1 能解决节流尾延迟”。

### 2.3 A3 OOM

预测（测量之前写）：`/api/report?mb=8&ms=500` 在到达率 10、20、30 时平均和最多各有几个在途请求？RSS 大约多少？在 `MemoryMax=200M` 下会在哪一个阶段被杀？

在 `-report-max=0`（不限并发）下，按 Little's Law，`L = λW`。每个请求在 `time.Sleep(500ms)` 的整段 500 ms 内都持有 8 MiB 的 `buf`，分配、逐页写和求和只占几毫秒，所以 `W ≈ 0.5 s`；GC 与请求争 CPU 时 `W` 只会更大，因此下面的 `L` 是下限。到达率 10、20、30 请求/秒 时，平均在途 `L ≈ 5、10、15`。

“最多”按泊松到达、固定 500 ms 服务时间（M/D/∞）估：30 s 阶段内瞬时在途数近似服从均值为 `L` 的泊松分布，最大值约为 13、22、30（99% 分位约 16、26、34）。`report_inflight_max` 是 1 s 采一次的最大值，实测会低于这个瞬时值；30 阶段若很快被杀，采样次数少，偏低会更多。

RSS 先按“基线 + 在途数 × 8 MiB”算：基线取 `runs/a2-q100/metrics-start.txt` 的 `process_resident_memory_bytes` 1.42×10⁷ B ≈ 13.5 MiB（其中 `go_heap_objects_bytes` 约 3.2 MiB；`runs/a2-q100/red.csv` 第 1 阶段的 `rss_max_mb` 为 13.58，该阶段没有 `/api/report` 流量），则平均在途时约为 54、94、134 MiB。但 GOGC=100 让下一次 GC 的堆目标约为上次 GC 结束时存活堆的 2 倍：8 MiB 的 `buf` 在 500 ms 内都存活，若 GC 在平均在途数附近结束，堆要涨到约 `2 × L × 8 MiB` 才回收，故 `heap_max_mb` 的峰值约为 80、160、240 MiB，`rss_max_mb` 再加约 14 MiB 基线，约为 95、175、255 MiB（30 请求/秒 时会在 200 MiB 处先被杀）。实测峰值通常落在“1×”和“2×”之间、更靠近 2×，因为 `rss_max_mb` 取的是阶段内最大值；但 2× 不是硬上限：GC 目标是按“上一次回收结束时的存活量”定的，若某次 GC 恰在在途峰值结束（例如 20 请求/秒 时的 22 个），目标会按 `22 × 8 MiB` 再翻倍，RSS 可以超过按平均数算的 2×；被释放的 8 MiB 也不会立刻还给操作系统，RSS 有滞后，会把前一阶段的峰值带进下一阶段。没有设 `GOMEMLIMIT` 时，Go 运行时不知道 cgroup 的 200 MiB，GC 不会为它提前，200 MiB 是内核的硬墙。

在 `MemoryMax=200M`、`MemorySwapMax=0` 下：10 请求/秒 的峰值约 95 MiB，按平均估计安全（余量约 100 MiB）；20 请求/秒 按平均估计约 175 MiB，已在边界：即使按更低的 `mem_current_max_mb` 基线（`runs/a2-q100/red.csv` 第 1 阶段的 `mem_current_max_mb` 为 7.90，该阶段没有 `/api/report` 流量，比 RSS 低约 6 MiB）算，余量也只有约 31 MiB，若 GC 在在途峰值结束就会越过 200 MiB；30 请求/秒 按 2× 估计约 255 MiB，且 30 × 8 = 240 MiB/s 的分配速率下，即使从空载基线算，涨到 200 MiB 也只要约 0.8 s（若沿用上一阶段保留的约 175 MiB，则只要约 0.1 s）。主预测是第 3 阶段（30 请求/秒）被杀，预计在该阶段开始后 1 s 内；第 2 阶段并不安全，若实测在第 2 阶段就被杀，原因就是 GC 目标按峰值而不是按平均数翻倍。

**实验一：`MemoryMax=200M`，不限并发（`runs/a3-oom`）。** `meta.txt` 的 `report_max` 为 0。阶段 1 完整；阶段 2 开始约 2 s 后进程被 OOM 杀掉（`journalctl` 的 16:10:27 一行），所以 `red.csv` 的阶段 2 为空，该行的 `ok_rps`、`error_rate` 来自 `steps.csv`。

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.42 | 4.73333 | 8 | 138.363 | 121.365 | 132.898 | 200 |
| 2 | 20 | 1.9 | 0.9066 | 505.55 | - | - | - | - | - | - |
| all | - | 5.5 | 0.6211 | 505.42 | - | - | - | - | - | - |

**实验二：`MemoryMax=300M`，不限并发（`runs/a3-limit300`）。** `report_max` 仍为 0。阶段 2 跑完，进程在阶段 3 开始约 1 s 后退出（`journalctl` 的 16:21:03 一行），`red.csv` 的阶段 3 为空。

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.35 | 4.46667 | 7 | 131.418 | 121.396 | 126.039 | 300 |
| 2 | 20 | 19.8 | 0.0000 | 505.37 | 10.8667 | 17 | 298.938 | 177.498 | 293.574 | 300 |
| 3 | 30 | 0.1 | 0.9968 | 501.95 | - | - | - | - | - | - |
| all | - | 9.7 | 0.5171 | 505.35 | - | - | - | - | - | - |

**实验三：`MemoryMax=200M`，`-report-max=11`（`runs/a3-cap11`）。** 三个阶段都跑完，没有 OOM；代价是 503：`steps.csv` 的 `codes` 在阶段 2 为 `503:69`、阶段 3 为 `503:233`。

| step | target_rps | ok_rps | error_rate | p99_ms | inflight_avg | report_inflight_max | rss_max_mb | heap_max_mb | mem_current_max_mb | mem_limit_mb |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 10 | 9.1 | 0.0000 | 505.23 | 4.2 | 9 | 134.746 | 89.3629 | 129.289 | 200 |
| 2 | 20 | 16.4 | 0.1742 | 505.05 | 7.53333 | 11 | 186.805 | 145.402 | 181.406 | 200 |
| 3 | 30 | 19.5 | 0.3740 | 504.69 | 9.8 | 11 | 186.965 | 169.436 | 181.617 | 200 |
| all | - | 15.0 | 0.2515 | 505.04 | - | - | - | - | - | - |

**与预测的对照。** 主预测“第 3 阶段被杀”没有成立：`runs/a3-oom` 在阶段 2 开始约 2 s 就被杀（阶段 2 开始于 16:10:25，`journalctl` 的 OOM 记录为 16:10:27；阶段 2 成功 37 个，约合 1.9 s 的 20 请求/秒）。偏差的原因正是预测里写到的“第 2 阶段并不安全”：20 请求/秒 阶段实测最多 17 个在途（`runs/a3-limit300/red.csv` 阶段 2；预测的 22 是按 30 s 阶段估的，实际阶段是 20 s，峰值会低一些），按 GOGC=100 的 2 倍目标，RSS 约 10 + 2 × 17 × 8 ≈ 282 MiB，已经超过 200M，不必等到 30 请求/秒 的 22～30 个在途（进程不会真长到 282 MiB：`runs/a3-oom/pidstat.log` 在 16:10:27 的最后一秒 RSS 为 190876 KB ≈ 186.4 MiB，加上 cgroup 里的页缓存等，`memory.current` 已顶到 200 MiB，硬墙先到）。预测给的 RSS 95/175/255 MiB 是按平均在途 5/10/15 算的，实测控制峰值的是阶段内最多在途，不是平均：`runs/a3-oom` 阶段 1 的 `rss_max_mb` 为 138.363（10 + 2 × 8 × 8 ≈ 138，`report_inflight_max` 为 8），`runs/a3-cap11` 阶段 3 为 186.965（10 + 2 × 11 × 8 ≈ 186，`report_inflight_max` 为 11）；`runs/a3-limit300` 阶段 2 的 298.938 比 10 + 2 × 17 × 8 ≈ 282 高约 17 MiB，差额来自 1 s 采样漏掉的瞬时峰值和 RSS 的滞后。也就是说，“基线 + 在途数 × 8 MiB”要再乘 GC 的 2 倍余量、并用最多在途，才接近实测；平均在途只适合估平均 RSS，不适合定 limit。

把 limit 提到 300M（`runs/a3-limit300`）后，20 请求/秒 阶段 396 个请求全部成功，但 `rss_max_mb` 已达 298.938，离 300M 只有约 1 MiB；30 请求/秒 阶段一开始，约 1 s 后进程退出（阶段 3 开始 16:21:02，`journalctl` 的 OOM 为 16:21:03；只成功 2 个，`net_err` 621）。提高 limit 只是把悬崖从 20 请求/秒 推迟到 30 请求/秒，并没有消除它。

`-report-max=11` 在 200M 下跑完三个阶段，`report_inflight_max` 在阶段 2、3 都恰为 11，`rss_max_mb` 为 186.805/186.965，`mem_current_max_mb` 为 181.406/181.617，systemd 的 `MemoryPeak` 为 190689280 B（≈181.9 MiB，见下面的证据），都低于 200M。代价是超出的请求立即 503：阶段 2 有 69 个、阶段 3 有 233 个（`steps.csv` 的 `codes`），全程 302/1201 = 25.15%。503 的耗时在 1 ms 以内（`runs/a3-cap11/metrics-end.txt` 中 503 的直方图桶全部落在 `le="0.001"`），所以 p99 仍由成功请求的 500 ms 保持时间决定，错误没有体现在 p99 上。

被杀的证据（`systemctl show -p ActiveState,Result,ExecMainStatus lab3-a` 和 `journalctl -u lab3-a` 的相关行，`dmesg` 或 `/var/log/kern.log` 中的 oom-kill 行）：

```text
root@ecs-e0a4:/home/sample/lab3# systemctl show -p ActiveState,Result,ExecMainStatus lab3-a; journalctl -u lab3-a --no-pager | tail -n 5
Result=oom-kill
ExecMainStatus=9
ActiveState=failed
Nov 19 16:09:44 ecs-e0a4 systemd[1]: Started lab3-a.service - /home/sample/lab3/bin/enrolld -addr 127.0.0.1:8081 -data /home/sample/lab3/data/a.
Nov 19 16:09:44 ecs-e0a4 enrolld[2242279]: 2026/11/19 16:09:44 enrolld v1 监听 127.0.0.1:8081；GOMAXPROCS=2；cgroup /sys/fs/cgroup/system.slice/lab3-a.service；预热 0s；读：work=200000、pool=32、db-ms=2；写：sync=each
Nov 19 16:10:27 ecs-e0a4 systemd[1]: lab3-a.service: A process of this unit has been killed by the OOM killer.
Nov 19 16:10:27 ecs-e0a4 systemd[1]: lab3-a.service: Main process exited, code=killed, status=9/KILL
Nov 19 16:10:27 ecs-e0a4 systemd[1]: lab3-a.service: Failed with result 'oom-kill'.
root@ecs-e0a4:/home/sample/lab3# sudo dmesg | grep -i oom-kill | tail -n 2
[1987194.863097] enrolld invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
[1987194.863240] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/system.slice/lab3-a.service,task_memcg=/system.slice/lab3-a.service,task=enrolld,pid=2242279,uid=0

root@ecs-e0a4:/home/sample/lab3# systemctl show -p ActiveState,Result,ExecMainStatus lab3-a
Result=oom-kill
ExecMainStatus=9
ActiveState=failed
root@ecs-e0a4:/home/sample/lab3# journalctl -u lab3-a --no-pager | tail -n 8
Nov 19 16:10:27 ecs-e0a4 systemd[1]: lab3-a.service: Main process exited, code=killed, status=9/KILL
Nov 19 16:10:27 ecs-e0a4 systemd[1]: lab3-a.service: Failed with result 'oom-kill'.
Nov 19 16:19:44 ecs-e0a4 systemd[1]: Started lab3-a.service - /home/sample/lab3/bin/enrolld -addr 127.0.0.1:8081 -data /home/sample/lab3/data/a.
Nov 19 16:19:44 ecs-e0a4 enrolld[2244338]: 2026/11/19 16:19:44 enrolld v1 监听 127.0.0.1:8081；GOMAXPROCS=2；cgroup /sys/fs/cgroup/system.slice/lab3-a.service；预热 0s；读：work=200000、pool=32、db-ms=2；写：sync=each
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: A process of this unit has been killed by the OOM killer.
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Main process exited, code=killed, status=9/KILL
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Failed with result 'oom-kill'.
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Consumed 1.221s CPU time, 3.5M memory peak, 0B memory swap peak.
root@ecs-e0a4:/home/sample/lab3# sudo dmesg -T | grep -i oom-kill | tail -n 2
[Thu Nov 19 16:20:52 2026] enrolld invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
[Thu Nov 19 16:20:52 2026] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/system.slice/lab3-a.service,task_memcg=/system.slice/lab3-a.service,task=enrolld,pid=2244338,uid=0

root@ecs-e0a4:/home/sample/lab3# systemctl show -p ActiveState,Result,ExecMainStatus,MemoryPeak lab3-a
Result=success
ExecMainStatus=0
MemoryPeak=190689280
ActiveState=active
root@ecs-e0a4:/home/sample/lab3# journalctl -u lab3-a --no-pager | tail -n 5
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Main process exited, code=killed, status=9/KILL
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Failed with result 'oom-kill'.
Nov 19 16:21:03 ecs-e0a4 systemd[1]: lab3-a.service: Consumed 1.221s CPU time, 3.5M memory peak, 0B memory swap peak.
Nov 19 16:23:27 ecs-e0a4 systemd[1]: Started lab3-a.service - /home/sample/lab3/bin/enrolld -addr 127.0.0.1:8081 -data /home/sample/lab3/data/a -report-max=11.
Nov 19 16:23:27 ecs-e0a4 enrolld[2245170]: 2026/11/19 16:23:27 enrolld v1 监听 127.0.0.1:8081；GOMAXPROCS=2；cgroup /sys/fs/cgroup/system.slice/lab3-a.service；预热 0s；读：work=200000、pool=32、db-ms=2；写：sync=each
root@ecs-e0a4:/home/sample/lab3# sudo dmesg -T | grep -i oom-kill | tail -n 2
[Thu Nov 19 16:20:52 2026] enrolld invoked oom-killer: gfp_mask=0xcc0(GFP_KERNEL), order=0, oom_score_adj=0
[Thu Nov 19 16:20:52 2026] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=(null),cpuset=/,mems_allowed=0,oom_memcg=/system.slice/lab3-a.service,task_memcg=/system.slice/lab3-a.service,task=enrolld,pid=2244338,uid=0
```

两种改法：按最坏情况定 limit，本组实测取 300 MiB（算式：实测 20 请求/秒 阶段的最多在途 17 个，2 × 17 × 8 MiB + 约 10 MiB 基线 ≈ 282 MiB，向上取 300 MiB；`runs/a3-limit300/meta.txt` 的 `mem_limit_mb` 为 300），但这次实测说明它只够 20 请求/秒；若还要覆盖 30 请求/秒，按预测的峰值约 30 个在途算，2 × 30 × 8 + 10 ≈ 490 MiB，limit 应取 512 MiB（本组未实测这个值）。限制并发时 `-report-max` 取 11（依据：内存预算 (200 − 10) / (2 × 8) ≈ 11.9，向下取 11；实测阶段 3 的 `rss_max_mb` 为 186.965，仍有约 13 MiB 余量），代价是 20、30 请求/秒 下分别有 17.42%、37.40% 的 503。

## 3. 任务 B：伸缩控制环

### 3.1 B0 单个实例的能力

<粘贴 `runs/b0-…/steps.csv` 的主要列，或 `scripts/table.sh runs/b0-… ok_rps p99_ms` 的输出>

单个实例（CPU 配额 <填写>%）的上限约 <填写> 请求/秒，按 0.7 的容量线可承担 <填写> 请求/秒；依据：<填写>。

本组的到达曲线：平时 <填写>、峰值 <填写> 请求/秒（`scripts/plan.sh` 的输出：`<填写>`）。

预测（B1～B3 之前写）：峰值需要几个实例？反应式从爬升开始到新实例就绪大约要多久，缺口大约多长？定时预热最晚何时开始？<填写>

### 3.2 三种方案的对照

| 方案 | `runs/` 目录 | 参数 | 缺口（秒） | 错误数 | 峰值阶段的 p99（ms） | 实例·分钟 |
|---|---|---|---|---|---|---|
| B1 常驻 | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |
| B2 反应式（信号一） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |
| B2 反应式（信号二） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |
| B3 定时加反应式 | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |

### 3.3 控制环各段的实测

以 B2（信号一）为例，从 `summary.md` 的“伸缩决策”和“实例”两张表读出：

| 段 | 起止 | 实测（秒） | L-11 第 10 页的设定 |
|---|---|---|---|
| 指标延迟与等待决策（爬升开始 → 第一次扩容决策） | <填写> | <填写> | 指标约 20 s + 决策 ≤ 15 s |
| 拉镜像（创建 → 启动） | <填写> | <填写> | 约 20 s |
| 启动、预热与健康检查（启动 → 就绪） | <填写> | <填写> | 约 60 s |
| 合计（爬升开始 → 新实例就绪） | <填写> | <填写> | 约 2 min |

B2 缺口中的错误：客户端超时 <填写> 个，lbd 返回的状态码 <填写>（`steps.csv` 的 `timeout`、`codes`）；这段时间 `lbd-events.csv` 中有 <填写> 次 `down`。

## 4. 任务 C：故障注入

| 注入 | `runs/` 目录 | lbd 的参数 | 判定用时（秒） | 恢复用时（秒） | 注入到判定之间的错误 | 全程错误 | 其他 |
|---|---|---|---|---|---|---|---|
| C1 停止（kill） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> | 判定是 `removed` 还是 `down`：<填写>；补建的实例何时就绪：<填写> |
| C1 卡住（pause 一台 20 秒） | 同上 | 同上 | <填写> | <填写> | <填写> | 同上 | 判定是 `removed` 还是 `down`：<填写> |
| C2 暂停 2 秒（灵敏） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |
| C2 暂停 2 秒（宽松） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> |
| C3 变慢（不摘除） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> | 全站 p99：<填写> |
| C3 变慢（离群摘除） | <填写> | <填写> | <填写> | <填写> | <填写> | <填写> | 全站 p99：<填写> |

预测（C1 之前写）：按 lbd 的参数，一个实例停止或卡住以后多久被判定？其间大约有多少请求出错？<填写>

C1 中停止与卡住的判定用时各是多少，分别是谁发现的？与“间隔 × 失败阈值”相比如何？<填写>

C3 中把配额调到了多少，依据的用量 U 是多少：<填写>

C3 中健康检查为什么全部通过？用 `runs/<名字>/lbd-metrics-end.txt` 或 Prometheus 中按实例的数据，找出是哪一台变慢，并说明依据：<填写>

## 5. 任务 D：发布与自愈

### 5.1 D1 滚动发布

预测：按 L-05 第 24 页的公式，3 个实例逐个替换要多久？直接替换时大约多少请求出错？<填写>

| 做法 | `runs/` 目录 | 发布总时长（秒） | 错误数 | 出错的时段 | 说明 |
|---|---|---|---|---|---|
| 直接替换（naive） | <填写> | <填写> | <填写> | <填写> | <填写> |
| 排空加就绪（graceful） | <填写> | <填写> | <填写> | <填写> | <填写> |

### 5.2 D2 自愈

| 项 | 数值与出处 |
|---|---|
| 参数（`-restart`、`-args`、`-memory`） | <填写> |
| 每个实例的泄漏速度、预计多久 OOM 一次 | <填写> |
| 实测的重启次数（`systemctl show -p NRestarts lab3-inst-…`、`summary.md` 的“实例”表） | <填写> |
| 各实例重启的时刻（`instances.csv` 的 restarted 行）与缺口的时刻 | <填写> |
| 全程的成功率、错误数 | <填写> |

本组写的告警查询，以及它在这次实验中的值：

```text
<填写>
```

## 6. 分析问题

每题先列出所用数据（引用上面的表或 `runs/` 中的文件），再写结论，并说明结论的成立条件。每题 300～600 字。

**问题 1：配额。** 数据：`runs/a3-oom`、`runs/a3-limit300`、`runs/a3-cap11` 的三张表，`runs/a2-q50`、`runs/a2-q60`、`runs/a2-q50-g1` 的 `steps.csv`/`red.csv`，以及 A1 的两次输出。A1 实测 50.1/74.9 ms，算术 50.0/75.0 ms，差 0.1 ms（约 0.1%～0.2%），来自 10 s 窗口的整周期取整、线程启停的非整周期和 `throttled_usec` 按 CPU 累计；2 线程的 74.9 ms 反推 A1 的 cgroup 能用 2 个核，没有沿用 `lab3-a` 的绑核。A2 平均 0.41/0.37 核（两阶段），低于 0.5 核配额，p99 却从 100% 配额的 11.05 ms 升到 42.40 ms：95 请求/秒 在 100 ms 周期内平均 9.5 个请求、约 40 ms CPU，但泊松到达下每周期超过 50 ms 的概率约 1/4，一超就整段停到下一周期，平均低不代表每周期不超。改法一（60% 配额）把 p99 降到 16.04 ms、节流比例降到 8.57%/2.86%，代价是多占 0.1 核，且依赖峰值估计准确；改法二（`GOMAXPROCS=1`）p99 反升到 48.15 ms、p50 升到 8.41 ms，等待从周期末转到单 P 运行队列，CPU 总量没少，尾延迟更差。

A3 的 RSS 用“约 10 MiB + 2 × 最多在途 × 8 MiB”大体吻合：a3-oom 阶段 1 为 138（实测 138.363），a3-cap11 阶段 3 为 186（实测 186.965）；按平均在途算的 95/175/255 偏低，a3-limit300 阶段 2 的 298.938 就是反例。GOGC=100 让 GC 目标按上次存活堆翻倍，8 MiB 的 `buf` 在 500 ms 内都存活，所以 RSS 在 2× 附近，且释放的页不立刻还给 OS，会把上一阶段的峰值带过来。`MemoryMax` 只限内存，不同时设 `MemorySwapMax=0` 时，开了 swap 的机器可以把匿名页换出后继续分配，200M 不是硬墙；本机无 swap，两者等效，但仍应一起写。

A3 的两种改法：按最坏情况定 limit 适合峰值可预测、内存有余量且不愿丢请求的场景，代价是内存和仍可能被单次尖峰击穿；限制并发适合内存固定、能接受 503 并由上游重试/扩容的场景，代价是吞吐上限（11 × 2 = 22 请求/秒）和错误率。

**问题 2：控制环。** 各段延迟的实测值是多少，哪一段最长，能压缩多少？三种方案的缺口与实例·分钟各是多少？两种信号的差别在哪里，为什么？B2 的缺口中除了超时，还有 lbd 返回的 503：过载为什么会让健康检查也失败，这使缺口变得更严重还是更轻？你们会为“开放日”选哪个方案，结论在什么条件下成立？（可选：思考题 3 第四幕中你们的判断，与这里测到的现象是否一致？）

<填写>

**问题 3：健康检查。** 停止、卡住、暂停、变慢四次注入中，各是谁、用多久发现的，健康检查看到了什么？检出快与误判少之间怎样取舍？离群摘除解决了什么问题，又带来什么风险？为什么要限制同时摘除的比例？

<填写>

**问题 4：发布与自愈。** 直接替换时，失败的请求来自哪几步？排空加就绪消除了哪几类失败，代价是什么？D2 中三个实例的重启时刻有什么关系，为什么？这对“有冗余，一台重启不影响服务”的假设意味着什么？自愈解决了什么、掩盖了什么？应当在哪一层、用什么指标告警？

<填写>

## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 GOMEMLIMIT | <未做 / 预测、`runs/` 目录、结论> |
| O2 两层伸缩 | <未做 / 参数、`runs/` 目录、结论> |
| O3 扩错层 | <未做 / 预测、`runs/` 目录、结论> |
| O4 Kubernetes 的 HPA | <未做 / 环境、配置文件、数据与结论> |
| O5 换负载均衡器 | <未做 / 配置文件、`runs/` 目录、结论> |
| O6 两台主机 | <未做 / 拓扑、`runs/` 目录、结论> |
| O7 迁移到小组项目 | <未做 / 仓库链接、做了什么、结论> |

## 8. 成员分工

每位成员一行，只填写 GitHub 用户名，无需填写姓名；行数应与小组人数一致。

| GitHub 用户名 | 负责的部分 |
|---|---|
| <用户名> | <填写> |
