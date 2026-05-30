# Big Lab B 总结报告

> cg24-THU  
> 日期：2026-05-30  
> 本地分支：`fix/starry-syscall-harness`  
> 主要依据：当前仓库代码与文档、`task1_os_learning_report.md`、`docs/` 中 qperf 与 harness 报告、`rcore-os/tgoskits` 和 `rcore-os/virtio-drivers` 中由 `cg24-THU` 提交的 PR。

## 一、整体概述

本次 Big Lab B 的主线可以概括为：

```text
Task1 建立 OS 分层意识
  -> 修复 StarryOS Linux syscall 兼容性细节
  -> 将 qperf 接入 StarryOS 性能分析流程
  -> 建立 Linux-vs-StarryOS syscall/qperf harness
  -> 用 harness 和 qperf 定位 virtio-blk 性能瓶颈
  -> 用 A/B profile 验证真实优化效果
  -> 将可复用的 virtio 性能思路推进到上游 virtio-drivers
```

这组工作不是单独实现一个功能点，而是在 StarryOS 上逐步形成了一个“可发现、可解释、可验证、可复现”的 OS 工程闭环。前期的 syscall 修复关注 Linux ABI 的正确性，中期的 qperf 关注性能热点的可观测性，后期的 harness 则把 syscall 对拍、qperf profiling、MCP、Web UI、知识图谱和 A/B compare 串成统一工具链。最终，性能优化不再依赖主观猜测，而是可以通过 marker window、driver counters、深调用栈火焰图和 compare 报告证明。

从 PR 角度看，当前公开成果包括：

| 仓库 | PR | 状态 | 主题 | 说明 |
| --- | --- | --- | --- | --- |
| `rcore-os/tgoskits` | [#268](https://github.com/rcore-os/tgoskits/pull/268) | Merged | `pipe2` invalid flags | 严格拒绝未知 flags，避免静默创建 fd |
| `rcore-os/tgoskits` | [#476](https://github.com/rcore-os/tgoskits/pull/476) | Closed | qperf 与 `preadv2/pwritev2` 早期整合 | 范围过大，后续成熟部分进入 #665 |
| `rcore-os/tgoskits` | [#663](https://github.com/rcore-os/tgoskits/pull/663) | Closed | qperf 集成误提交 | 实际 diff 与标题不符，关闭合理 |
| `rcore-os/tgoskits` | [#665](https://github.com/rcore-os/tgoskits/pull/665) | Merged | qperf 初版、vectored I/O、BusyBox 测试 | 已合入，是后续 qperf/harness 的基础 |
| `rcore-os/tgoskits` | [#783](https://github.com/rcore-os/tgoskits/pull/783) | Closed | virtio-blk queue/barrier 优化 | 方向有效，但 vendoring `virtio-drivers` 不适合作为 tgoskits PR |
| `rcore-os/tgoskits` | [#940](https://github.com/rcore-os/tgoskits/pull/940) | Open | qperf hotspot enhancement | 增强 qperf 地址过滤、TB 采样、symtab fallback、flamegraph、diff |
| `rcore-os/tgoskits` | [#990](https://github.com/rcore-os/tgoskits/pull/990) | Open | syscall/qperf harness | 本次工作的核心分支，含 harness、MCP/UI、metrics、深栈、readahead 优化与知识图谱 |
| `rcore-os/virtio-drivers` | [#249](https://github.com/rcore-os/virtio-drivers/pull/249) | Open | virtio-blk queue size 与 memory barrier | 将 #783 的核心驱动优化上游化 |

截至本报告撰写时，`#940` 已经历多轮 review 并获得 approve，CI/check 多项通过；`#990` 仍处于 open 状态，主要阻塞是与最新 `dev` 的 block driver API 重构冲突、部分文档/路径细节以及 PR 规模较大。本文的“当前仓库实际情况”以本地 `fix/starry-syscall-harness` 分支和 #990 当前内容为准，因此会同时区分“已合入成果”和“当前开放 PR 中的成果”。

## 二、Task1：从教学实验建立 OS 分层意识

Task1 是后续所有工作的基础。根据 `task1_os_learning_report.md`，我完成了 ArceOS tutorial 中 5 个 exercise：

```text
exercise-printcolor
exercise-hashmap
exercise-altalloc
exercise-sysmap
exercise-ramfs-rename
```

这 5 个任务覆盖了一条完整的 OS 学习路径：

```text
应用输出字节流
  -> no_std 标准库兼容层
  -> 内核早期内存分配器
  -> syscall 与用户态虚拟内存
  -> VFS 分发与具体文件系统语义
```

### 2.1 `exercise-printcolor`：输出不是内核理解颜色

彩色输出的实现本质是向 console 写入带 ANSI escape sequence 的字节流。内核 console 或串口驱动不需要理解“绿色”“高亮”“重置”这些概念，只需要可靠传递字节；颜色效果由终端解释。这一题让我建立了一个重要判断：不是每个用户可见效果都需要下沉到内核对象，很多接口只是协议化字节流。

这个判断后来迁移到了 `/proc`、netlink 和 qperf marker 设计中。例如 BusyBox 或 harness 解析的不是内核中的“结构体对象”，而是稳定文本协议或 stdout marker。只要这些文本是 ABI，就必须像 syscall 返回值一样严肃对待。

### 2.2 `exercise-hashmap`：`std` 在 OS/unikernel 中是兼容层

`exercise-hashmap` 的关键不是实现哈希表算法，而是理解 `axstd` 这类 `no_std + alloc` 环境下的标准库兼容层。用户程序写的是：

```rust
use std::collections::HashMap;
```

但在 ArceOS 场景中，这个 `std` 实际由项目自己的 `axstd` 提供。补齐 `HashMap` 的正确位置不是应用层绕过，也不是内核里硬编码，而是在兼容层暴露 `hashbrown::HashMap` 等集合类型。

这与后续 StarryOS Linux 兼容性修复完全同构：用户态看到的是 Linux ABI 承诺的路径、errno、flags 和结构体布局；底层可以用不同实现，但对外契约不能随意改变。

### 2.3 `exercise-altalloc`：早期 allocator 要维护清晰不变量

`exercise-altalloc` 中的 early allocator 采用双端 bump 设计：

```text
[ bytes-used | available area | pages-used ]
start       b_pos           p_pos        end
```

字节分配从低地址向高地址推进，页分配从高地址向低地址推进。这个设计让我实际处理了 OS 内存管理中经常被课本抽象掉的细节：

- `align_up` / `align_down` 对齐；
- `checked_add` / `checked_sub` 防止整数溢出；
- `start <= b_pos <= p_pos <= end` 的核心不变量；
- 字节区域与页区域不能相撞；
- early boot allocator 不追求复杂回收，只服务启动阶段。

后续在 qperf metrics、virtio buffer、rsext4 readahead 中也反复出现同类问题：性能优化必须先保证对象生命周期、边界和不变量正确。

### 2.4 `exercise-sysmap`：`mmap` 连接 syscall、VFS 和虚拟内存

`exercise-sysmap` 表面是在补 `SYS_MMAP`，实际打通了：

```text
用户态程序
  -> syscall 入口
  -> fd table
  -> 用户地址空间映射
  -> 文件 read_at
  -> aspace.write 写入用户虚拟地址
```

这个实验的关键经验是：`mmap` 不是“返回一个指针”，而是内核与用户态关于地址空间的一份协议。内核必须同时处理 length、offset、prot、flags、fd、页对齐、权限转换、文件内容回填和错误路径回滚。

这个经验直接影响了后续 syscall 修复。例如 #665 中 `preadv2/pwritev2` 修复必须严格区分 raw syscall 参数槽位、`offset == -1`、RWF flags 和用户 iovec 边界；#990 中 harness 的 syscall probe 也坚持用 Linux 输出作为 oracle，而不是按 StarryOS 当前行为倒推测试。

### 2.5 `exercise-ramfs-rename`：VFS 不是一个函数名

`exercise-ramfs-rename` 让我看到 `std::fs::rename` 后面实际是一条分层调用链：

```text
std::fs::rename
  -> axstd fs API
  -> axfs RootDirectory
  -> mount routing
  -> axfs_ramfs::DirNode
  -> children BTreeMap remove/insert
```

只在 ramfs 目录节点里实现 rename 不够，根目录分发和 mount 边界也必须正确。这个经验后来在 StarryOS 中非常重要：用户态一个失败现象可能落在 syscall wrapper、VFS、具体 fs、fd table、block driver 或测试 harness 任意一层。修复前必须先定位层次，不能只搜到一个同名函数就开始改。

## 三、Syscall 兼容性修复：从 flags 到 errno 优先级

本次工作中的 syscall 修复主要集中在 #268、#665 和 #990。它们共同体现了 Linux ABI 兼容性的一个核心事实：真正难的经常不是“实现 syscall 名字”，而是对齐边界输入、错误码优先级、失败路径副作用和 raw ABI 参数布局。

### 3.1 `pipe2` invalid flags：未知位不能被静默截断

#268 修复了 `sys_pipe2()` 使用 `PipeFlags::from_bits_truncate(flags)` 的问题。旧实现会静默丢弃未知 flags，例如用户传入 `O_CREAT | O_NONBLOCK` 时只保留 `O_NONBLOCK`，继续创建 pipe。Linux 语义要求未知 flags 返回 `EINVAL`，并且不能产生 fd 表副作用。

修复后，flags 解析改为严格模式：

```rust
PipeFlags::from_bits(flags).ok_or_else(|| AxError::InvalidInput)?
```

这个改动很小，但语义很关键：

- 合法 `O_CLOEXEC` / `O_NONBLOCK` 路径保持不变；
- 非法 flags 立即失败；
- 不再创建 pipe fd；
- 用户态 feature detection 不会被假成功误导。

这类修复体现了 syscall ABI 的基本纪律：flags 不是 hint，未知 bit 不能悄悄忽略。

### 3.2 `preadv/pwritev/preadv2/pwritev2`：raw ABI 参数槽位必须对齐

#665 中的 vectored I/O 修复是更复杂的一组 ABI 对齐。Linux raw syscall 参数为：

```text
fd, iov, iovcnt, pos_l, pos_h, flags
```

在 64 位架构上，`pos_h` 通常不参与 offset 组合，但 ABI 槽位仍然存在。旧实现如果把 `arg4` 当成 flags 或忽略 `arg5`，在常见 `flags=0` 情况下可能暂时看不出问题，但一旦出现非零 flags 或 32 位 offset 组合，就会偏离 Linux。

修复后的逻辑包括：

- `offset_from_hilo(pos_l, pos_h)`：32 位上组合高低位，64 位上保留 `pos_l`；
- `preadv/pwritev` 显式拒绝负 offset；
- `preadv2/pwritev2` 支持 `offset == -1` 使用并更新当前文件偏移；
- 当前未支持的非零 `RWF_*` flags 返回 `EOPNOTSUPP`，而不是忽略；
- iovec 路径前置检查用户地址、负长度、累计长度溢出和 `isize::MAX` 上限；
- `pwritev2` 的 memfd offset write 走 seal-aware path，避免绕过 `F_SEAL_WRITE` / `F_SEAL_GROW`。

这组修复说明：Linux ABI 不是函数签名层面的“差不多”。参数槽位、offset 特例、flags 策略、用户指针访问时机都会被真实程序观察到。

### 3.3 `ftruncate`：只读 fd 与 `O_PATH` fd 应返回 `EBADF`

#990 中 `sys_ftruncate` 修复了只读普通文件 fd 和 `O_PATH` fd 的 errno。当前代码路径是：

```rust
let f = File::from_fd(fd).map_err(|e| {
    if e == AxError::IsADirectory {
        AxError::from(LinuxError::EINVAL)
    } else {
        e
    }
})?;

let flags = file.flags();
if flags.contains(FileFlags::PATH) {
    return Err(AxError::BadFileDescriptor);
}
if !flags.contains(FileFlags::WRITE) {
    return Err(AxError::BadFileDescriptor);
}
file.access(FileFlags::WRITE)?.set_len(length as _)?;
```

这里的重点不是简单地“只读不能写”，而是 errno 优先级和 fd 类型语义：

- `length < 0` 返回 `EINVAL`；
- 无效 fd、关闭 fd、只读 fd、`O_PATH` fd 返回 `EBADF`；
- 目录 fd 映射为 `EINVAL`；
- pipe fd 返回 `EINVAL`；
- 超大 length 返回 `EFBIG` 或相关空间错误；
- bad fd + huge length 应先返回 `EBADF`，不能先报 `EFBIG`。

`test-suit/starryos/normal/qemu-smp1/syscall/test-ftruncate/c/src/main.c` 中的回归测试覆盖了 16 个场景，其中只读 fd 已收紧为严格 `EBADF`。这类测试不是简单 smoke，而是在固定 Linux 失败路径的状态副作用和错误优先级。

### 3.4 Syscall probe：让 Linux oracle 自动进入开发流程

#990 的 `tools/starry-syscall-harness/probes/syscall_probe.c` 把几个典型 syscall 语义做成稳定 `CASE` 输出：

```text
CASE pipe2_invalid_flags ret=-1 errno=22 created=0
CASE pwritev2_writes_data ret=2 errno=0 read_ret=2 read_errno=0 data=5859
CASE ftruncate_readonly_fd ret=-1 not_open_for_write=1
```

probe 设计遵循几个原则：

- Linux 和 StarryOS 跑同一份 C 代码；
- 每个 case 输出一行稳定 `CASE <name> key=value ...`；
- 不输出 fd 编号、地址、时间等不稳定字段；
- Linux 输出作为默认 oracle；
- StarryOS 缺失 case 或字段不一致都会进入 `differences`；
- `--fail-on-diff` 可作为门禁。

这正是 harness 的第一层价值：把“我认为 Linux 是这样”的人工判断，改成可重复、可机器读取、可作为 CI 门禁的事实。

## 四、Task4：qperf 从采样脚本到 StarryOS 性能平台

用户明确将 qperf 作为任务 4。本次 qperf 工作经历了三个阶段：

```text
#665: 初步接入 cargo starry perf
  -> #940: 增强 TCG hotspot profiling 能力
  -> #990: 结合 harness、marker、metrics、深栈和 A/B compare 做真实优化
```

### 4.1 qperf 的定位：QEMU TCG plugin，不是 host perf

qperf 位于 `tools/qperf/`。它的工作模式是：

```text
QEMU TCG callbacks
  -> 采集 guest PC / SP / FP / callchain
  -> 写出 qperf.bin
  -> qperf-analyzer 结合 StarryOS kernel ELF 符号化
  -> 生成 stack.folded / flamegraph.svg / report.json / report.md
```

这意味着 qperf 回答的是“guest kernel 当前执行在哪里”，不是硬件 PMU 问题。它不能直接给出 guest hardware cycles、guest cache misses 或精确 retired instructions。`--host-perf` 如果启用，测量的是宿主 QEMU 进程、TCG、设备模拟和 plugin overhead，只能作为 host-side context。

这个边界在报告中必须明确，因为性能工具最容易产生误读。qperf 的强项是跨平台、可复现地定位 StarryOS guest 内核热点路径，尤其适合在 QEMU TCG 环境中做 OS 实验与回归比较。

### 4.2 #665：`cargo starry perf` 初步接入

#665 已合入，首次把 qperf 接进 StarryOS 工作流。PR 描述中的核心命令是：

```bash
cargo starry perf --arch riscv64
```

该命令串联了：

```text
构建 qperf QEMU plugin
  -> 构建 qperf analyzer
  -> 构建 StarryOS debug kernel
  -> 准备 rootfs 和 QEMU config
  -> 通过 QEMU -plugin 注入 libqperf.so
  -> 收集 qperf.bin
  -> 生成 stack.folded / summary / flamegraph
```

实现路径包括：

- `scripts/axbuild/src/starry/perf.rs`：StarryOS perf 子命令编排；
- `scripts/axbuild/src/starry/mod.rs`：CLI 参数接入；
- `tools/qperf/src/profiler.rs`：QEMU TCG plugin；
- `tools/qperf/analyzer/src/main.rs`：符号解析与 folded stack 输出；
- `docs/qperf-starryos-integration-report.md`：集成报告。

#665 的验证结果包括 `cargo test -p axbuild` 207 个测试通过、qperf 产出非空 `stack.folded` 并包含 StarryOS/ArceOS kernel symbols。BusyBox 测试也从 `PASS: 272 FAIL: 0` 增加到 `PASS: 284 FAIL: 0`。

### 4.3 #940：地址过滤、TB 采样、symtab fallback 和 flamegraph

#940 在 #665 基础上解决了“采样能跑但不够可用”的问题。主要增强包括：

- 自动检测 kernel `.text` 范围，过滤 OpenSBI firmware 样本；
- TB-level sampling，默认按 translation block 执行回调采样，降低 instruction-level callback 开销；
- symtab fallback，DWARF 缺失时用 ELF `.symtab` 解析 release kernel 符号；
- 内置 inferno flamegraph，减少对外部 `flamegraph.pl` 的依赖；
- `--top N` 输出热点函数；
- diff mode 对比两次 folded stack。

这些改动让 qperf 从“能出一个火焰图”变成了“能给出工程热点排名、可对比、可在 release kernel 上工作的工具”。#940 的 review 也体现了工程迭代过程：早期因格式、clippy、导入路径和 merge conflict 被要求修改；后续修复后多轮 review approve，最新 CI/check 显示格式、clippy、std test、多架构 QEMU 等多项通过。

### 4.4 #990：qperf 与 harness 合流

#990 进一步把 qperf 从独立 profiling 子命令扩展为 harness 的一部分：

```bash
python3 tools/starry-syscall-harness/harness.py perf-profile --arch riscv64 --timeout 20
python3 tools/starry-syscall-harness/harness.py perf-diff --baseline ... --compare ...
python3 tools/starry-syscall-harness/harness.py perf-compare --baseline ... --candidate ...
```

当前 `perf-profile` 输出包括：

```text
report.json
report.md
hotspots.csv
hotspot_categories.csv
qperf/stack.folded
qperf/flamegraph.svg
qperf/summary.txt
qperf/qperf.summary.txt
qperf/qemu.time.txt
qperf/window.json
qperf/resolve.stats.json
qperf/stack-depth-summary.csv
```

这一步的关键不是多了几个文件，而是 qperf 结果开始变成结构化 evidence。后续性能优化可以明确说明“为什么改、改哪里、改完哪些 counters 下降、哪些热点下降、吞吐是否提升”。

## 五、StarryOS Syscall And Performance Harness

本次工作最核心的工程产物是 `tools/starry-syscall-harness/`。它不是一个简单脚本，而是面向 StarryOS 兼容性和性能优化的一套工作台。

### 5.1 harness 总体架构

当前 harness 的能力可以表示为：

```text
Developer / Codex / UI / MCP
        |
        v
tools/starry-syscall-harness/harness.py
        |
        +-- doctor
        |     -> 检查 Docker、镜像、QEMU、交叉工具链
        |
        +-- discover
        |     -> Linux probe
        |     -> StarryOS probe cross build
        |     -> rootfs 注入
        |     -> QEMU 运行
        |     -> CASE 行解析
        |     -> report.json 差分
        |
        +-- perf-profile / perf-postprocess
        |     -> cargo starry perf
        |     -> qperf.bin
        |     -> stack.folded / flamegraph
        |     -> hotspots / categories / report
        |
        +-- perf-diff / perf-compare
        |     -> folded stack 或 report.json 对比
        |     -> compare.json / compare.md / compare.csv
        |
        +-- ui / MCP
        |     -> 本地交互入口和 agent tool 调用
        |
        +-- knowledge graph
              -> 静态扫描 OS 子系统、代码路径和任务焦点
```

核心文件包括：

| 路径 | 作用 |
| --- | --- |
| `tools/starry-syscall-harness/harness.py` | CLI 主入口，负责 Docker re-exec、syscall discover、qperf profile、postprocess、compare |
| `tools/starry-syscall-harness/probes/syscall_probe.c` | Linux 与 StarryOS 共用 C probe |
| `tools/starry-syscall-harness/mcp_server.py` | 暴露 MCP tools，供 Codex 直接调用 |
| `tools/starry-syscall-harness/ui_server.py` | 本地 HTTP API、job 管理、artifact 文件服务 |
| `tools/starry-syscall-harness/knowledge_graph.py` | OS 知识图谱静态扫描 |
| `tools/qperf/src/profiler.rs` | QEMU TCG plugin |
| `tools/qperf/analyzer/src/main.rs` | raw sample 解析、符号化、flamegraph、diff |
| `scripts/axbuild/src/starry/perf.rs` | `cargo starry perf` 与 QEMU/qperf 集成 |

### 5.2 Docker re-exec：让环境成为工具的一部分

StarryOS build、rootfs 注入、QEMU、qperf plugin 和交叉工具链非常依赖环境。harness 默认使用：

```text
ghcr.io/rcore-os/tgoskits-container:latest
```

host 侧调用 `harness.py` 后，脚本会检查是否已在 Docker 内；如果没有，就通过 `docker run --rm -v repo:/work -w /work ...` 重入容器。结束后还会处理 artifact owner，避免 `target/starry-syscall-harness` 和 `tools/qperf/target` 被 root-owned 文件污染。

这体现了本轮工作的工程取向：工具不是只在某台机器能跑，而是把依赖环境、输出目录、权限修复和失败报告都纳入 workflow。

### 5.3 MCP 与 UI：让 agent 和人共用同一套事实

`mcp_server.py` 暴露 5 个工具：

```text
starry_syscall_doctor
starry_syscall_discover
starry_perf_profile
starry_perf_diff
starry_harness_ui_command
```

这样 Codex 不需要手写长命令，也不需要猜输出路径。参数由 schema 约束，输出会截断到可读范围，适合自动修复循环。

本地 UI 通过：

```bash
python3 tools/starry-syscall-harness/harness.py ui --host 127.0.0.1 --port 8765 --open
```

提供 doctor、discover、perf-profile、perf-diff、report 读取和 flamegraph 展示。它默认只绑定 `127.0.0.1`，同一时刻只允许一个重型 job，artifact 读取限制在 repo 和 harness target 目录下。这个安全边界很重要，因为 UI 会启动 Docker/QEMU 和读取本地 artifact。

### 5.4 Knowledge Graph：把代码热点映射回 OS 知识

#990 新增的 `Knowledge` 页让 harness 具备解释型能力。它会扫描：

```text
os/
components/
drivers/
scripts/
tools/
test-suit/
docs/
```

并把文件、符号、关键词和当前任务映射到 OS 子系统节点，例如：

- Linux syscall compatibility；
- Task / process lifecycle；
- Virtual memory / address space；
- VFS / file I/O；
- rsext4 / block cache；
- Block layer / request queue；
- virtio-blk driver；
- Network stack / socket path；
- procfs / debug observability；
- qperf / harness / GUI；
- Build / rootfs / qemu tests。

这部分的意义是：qperf 告诉我们热点在哪，Knowledge Graph 帮我们解释这个热点属于哪个 OS 子系统、和哪些目录相关、报告里应怎样讲。它不是精确调用图，也不替代人工读代码，但对实验报告、PPT 和新任务接手非常有价值。

## 六、Harness 帮助下的性能优化闭环

这一节是本报告重点。`docs/qperf-work-handoff.md` 和 `docs/qperf-virtio-blk-deepstack-optimization-report.md` 记录了完整的性能优化过程：从 qperf 工具增强，到瓶颈定位，再到 direct-only 失败尝试，最终通过 rsext4 readahead 获得可量化提升。

### 6.1 原始问题：profile 被 boot 污染，火焰图太浅，指标不够解释优化

早期 qperf 的主要限制包括：

| 限制 | 影响 |
| --- | --- |
| profile 覆盖 boot-to-exit | PCI probe、rootfs 初始化、shell 启动会污染 workload |
| 主要是 leaf PC/TB | 看不到 syscall -> fs -> driver -> virtqueue 的纵向路径 |
| 缺少工程分类 | 火焰图宽，但不知道属于 block、net、copy、lock 还是 virtqueue |
| 缺少 workload metrics | 不能按 MB 归一化，难以比较不同输入规模 |
| 缺少 driver counters | 不知道 request 数、notify/kick 数、copy bytes、queue depth |
| 缺少 A/B compare | 优化前后只能人工翻报告，不利于 review |

harness 的性能部分正是围绕这些缺口逐项补齐。

### 6.2 Workload marker/window：只分析真正的 workload

本轮引入 guest stdout marker：

```bash
echo QPERF_BEGIN:blk-read
dd if=/usr/bin/lto-dump of=/dev/null bs=64k
echo QPERF_END:blk-read
```

`scripts/axbuild/src/starry/perf.rs` 会等待 guest shell prompt，必要时执行 `stty -echo` 避免 shell 回显误触发 marker，然后注入 workload。看到 stop marker 后优先通过 QMP `quit` 收尾，失败再退回 SIGINT/kill。harness 后处理会根据 qperf raw sample 的 `elapsed_ns` 做 timestamp window 过滤，并记录：

```text
start_time
stop_time
duration_sec
boot_samples_excluded
post_window_samples_excluded
truncated_by_timeout
warnings
method = qperf_raw_elapsed_timestamp_filter
```

这里必须如实说明：当前 window 是 postprocess 过滤，不是 plugin runtime pause/resume。但它已经足以避免 boot 样本静默混入 workload 结论。

### 6.3 `/proc/qperf_metrics`：把 driver-visible 事件带进报告

仅靠火焰图不能回答“53 MB 顺序读触发多少次 block request”。因此 #990 增加了 feature-gated `qperf-metrics`：

```text
drivers/ax-driver/src/qperf_metrics.rs
os/StarryOS/kernel/src/pseudofs/proc.rs
```

guest 中可以执行：

```bash
echo reset > /proc/qperf_metrics
cat /proc/qperf_metrics
```

输出为：

```text
QPERF_METRIC virtqueue_add_count=... virtio_notify_kick_count=...
QPERF_METRIC qperf_metrics_export=1 qperf_metrics_scope=ax_driver_virtio
```

当前 counters 包括：

- virtqueue：`virtqueue_add_count`、`virtio_notify_kick_count`、`virtqueue_pop_complete_count`、`virtqueue_add_notify_wait_pop_count`、`virtqueue_depth_max`、depth histogram；
- virtio-blk：`virtio_blk_read_requests`、`virtio_blk_read_bytes`、`virtio_blk_write_requests`、`virtio_blk_write_bytes`、`virtio_blk_direct_read_requests`、`virtio_blk_direct_read_bytes`；
- virtio-net：RX/TX packets/bytes、RX `copy_within` 次数和字节数、TX staging copy、inflight insert/remove/get。

这些是 driver-visible 近似统计，不是 ring-level 精确硬件统计。但对于 A/B 比较非常有用：同一 workload、同一参数下，request 数和 notify/kick 数是否下降，可以直接说明优化是否击中了瓶颈。

### 6.4 深调用栈火焰图：从 leaf hotspot 到 syscall/fs/driver 路径

旧 qperf 的 folded stack 基本只有一层。`docs/qperf-callchain-validation-report.md` 中对旧 blk 产物的统计为：

```text
623 1
```

即 623 条 folded stack 全是一帧。新版 `--full-stack` 通过 RISC-V frame pointer 恢复内核调用链：

- qperf raw sample 记录 `pc/sp/fp/cpu/callchain/trace`；
- kernel 构建启用 debuginfo 和 `-Cforce-frame-pointers=yes`；
- plugin 从 `s0/fp` 追踪 frame pointer 链；
- analyzer 符号化并展开 inline frame；
- 输出 `stack-depth-summary.csv` 和深栈 flamegraph。

验证数据表明：

| case | workload samples | multi-frame samples | unwind success | avg symbol depth | raw max depth |
| --- | ---: | ---: | ---: | ---: | ---: |
| blk full-stack | 579 | 578 | 578 | 43.8877 | 18 |
| net full-stack | 661 | 652 | 652 | 50.6959 | 17 |

blk 深栈中可以看到：

```text
starry_kernel::syscall::fs::io::sys_read
  -> ax_fs_ng::highlevel::file::CachedFile::read_at
  -> rsext4::cache::data_block::DataBlockCache::get_or_load
  -> DataBlockCache::load_block
  -> Jbd2Dev::read_block
  -> ax_driver::block::binding::Block::read_blocks_wait
  -> rd_block::CmdQueue::read_blocks_blocking
  -> ax_driver::virtio::block::BlockQueue::submit_request
  -> virtio_drivers::device::blk::VirtIOBlk::read_blocks
  -> virtio_drivers::queue::VirtQueue::add_notify_wait_pop
```

这条路径非常关键：它把高层 `sys_read` 与底层 `VirtQueue::add_notify_wait_pop` 连接起来，说明热点不是孤立函数，而是 VFS/rsext4/block/virtio 同步小 I/O 链路。

### 6.5 virtio-blk baseline：同步 4 KiB 小 I/O 是主瓶颈

baseline workload：

```bash
echo reset > /proc/qperf_metrics
echo QPERF_BEGIN:blk
dd if=/usr/bin/lto-dump of=/dev/null bs=64k
cat /proc/qperf_metrics
echo QPERF_END:blk
```

关键数据来自 `docs/qperf-virtio-blk-deepstack-optimization-report.md`：

| 指标 | baseline |
| --- | ---: |
| dd bytes | 53,601,104 |
| dd elapsed | 6.367021 s |
| throughput | 8,418,553 B/s |
| marker window | 6.529766005 s |
| samples | 642 |
| callchain avg symbol depth | 43.306853583 |
| raw max depth | 16 |
| `virtio_blk_read_requests` | 13,629 |
| `virtio_blk_read_bytes` | 55,813,632 |
| `virtqueue_add_count` | 14,417 |
| `virtio_notify_kick_count` | 14,417 |
| `virtqueue_add_notify_wait_pop_count` | 14,354 |
| `virtqueue_pop_complete_count` | 14,354 |

hotspot categories：

| category | percent |
| --- | ---: |
| `block_io_path` | 75.8567% |
| `virtio_notify_kick` | 32.8660% |
| `virtqueue_add_notify_wait_pop` | 31.9315% |
| `memcpy` | 28.1931% |
| `lock_mutex_wait` | 10.5919% |

由 counters 可得：

```text
55,813,632 bytes / 13,629 read requests ≈ 4,095 bytes/request
```

也就是说，用户命令使用 `bs=64k`，但下层 driver-visible request 仍接近 4 KiB 粒度。大量小 read 基本都走同步 `add_notify_wait_pop`，notify/kick 数与 request 数同量级。此时首要优化方向不是盲目改 `memcpy`，而是减少同步小 I/O 次数，或改造为真正异步 pending read。

### 6.6 失败尝试：direct-only 不减少同步次数，反而退化

第一轮尝试是 direct read fast path，修改范围包括：

- `drivers/interface/rdif-block/src/lib.rs`；
- `drivers/blk/rd-block/src/lib.rs`；
- `drivers/ax-driver/src/block/binding.rs`；
- `drivers/ax-driver/src/virtio/block.rs`；
- `drivers/ax-driver/src/qperf_metrics.rs`。

思路是当目标 buffer 物理连续时，直接调用 `VirtIOBlk::read_blocks()` DMA 到目标 buffer，减少中间 buffer copy。

但是 A/B 结果显示：

| 指标 | baseline | direct-only |
| --- | ---: | ---: |
| throughput | 8,418,553 B/s | 6,889,660 B/s |
| workload elapsed | 6.367021 s | 7.779933 s |
| `virtqueue_add_notify_wait_pop_count` | 14,354 | 14,354 |

direct-only 确实命中了 read request，但没有减少同步 virtqueue 次数，所以主瓶颈仍在。这是 harness 的价值之一：它阻止我们把一个看起来合理、但没有击中主瓶颈的优化误写成成功。

### 6.7 成功优化：rsext4 data block readahead

最终有效优化落在：

```text
components/rsext4/src/cache/data_block.rs
```

当前代码中：

```rust
const DATA_BLOCK_READAHEAD: u32 = 8;
```

`DataBlockCache::get_or_load()` 在 cache miss 时不再只读一个 block，而是通过 `load_readahead()` 最多预读 8 个连续 filesystem block：

- `readahead_count()` 根据 filesystem total blocks、cache 容量和后续 block 是否已缓存决定预读数量；
- 批量分配 `self.block_size * count` 的 buffer；
- 调用 `Jbd2Dev::read_blocks(&mut data, block_num, count)`；
- 将批量数据按 block size 拆成 `CachedBlock` 插入 BTreeMap cache；
- 仍遵守 `max_entries` 和 LRU 驱逐约束；
- 遇到后续 block 已缓存时提前停止，避免覆盖和浪费。

这个优化与 qperf 证据完全对应：

```text
原路径：每个 4 KiB miss 单独 read_block -> add_notify_wait_pop
新路径：一次 miss 触发多个连续 block read_blocks -> 显著减少同步 virtqueue 次数
```

candidate 中深栈路径变为：

```text
sys_read
  -> CachedFile::read_at
  -> DataBlockCache::get_or_load
  -> DataBlockCache::load_readahead
  -> Jbd2Dev::read_blocks
  -> Block::read_block
  -> CmdQueue::read_blocks_direct
  -> BlockQueue::read_blocks_direct
  -> VirtIOBlk::read_blocks
  -> VirtQueue::add_notify_wait_pop
```

### 6.8 A/B 结果：吞吐 +25.33%，read request -84.51%

最终 compare 结论为 `明显改善`：

| 指标 | baseline | candidate | 变化 |
| --- | ---: | ---: | ---: |
| throughput_bytes_per_second | 8,418,553 | 10,551,346 | +25.3344% |
| workload elapsed | 6.367021 s | 5.080025 s | -20.2135% |
| samples.total_samples | 642 | 528 | -17.7570% |
| `virtio_blk_read_requests` | 13,629 | 2,111 | -84.5110% |
| `virtio_notify_kick_count` | 14,417 | 2,899 | -79.8918% |
| `virtqueue_add_count` | 14,417 | 2,899 | -79.8918% |
| `virtqueue_add_notify_wait_pop_count` | 14,354 | 2,836 | -80.2424% |
| `virtqueue_pop_complete_count` | 14,354 | 2,836 | -80.2424% |

hotspot category 对比：

| category | baseline | candidate | delta |
| --- | ---: | ---: | ---: |
| `virtio_notify_kick` | 32.8660% | 14.2045% | -18.6615 pp |
| `virtqueue_add_notify_wait_pop` | 31.9315% | 13.4470% | -18.4845 pp |
| `block_io_path` | 75.8567% | 66.2879% | -9.5688 pp |
| `memcpy` | 28.1931% | 31.0606% | +2.8675 pp |
| `lock_mutex_wait` | 10.5919% | 10.4167% | -0.1752 pp |

函数层面也能看到单块读路径基本消失：

| function | baseline | candidate | 说明 |
| --- | ---: | ---: | --- |
| `DataBlockCache::load_block` | 1.1869% | 0.0048% | 单块读基本消失 |
| `DataBlockCache::load_readahead` | 0.0000% | 1.1720% | 新批量读路径出现 |
| `Block::read_blocks_wait` | 0.8884% | 0.0000% | 旧同步 wrapper 热点消失 |
| `BlockQueue::read_blocks_direct` | 0.0000% | 0.3349% | direct 批量读路径出现 |
| `VirtQueue::add_notify_wait_pop` | 0.7373% | 0.3396% | leaf 占比下降 |

candidate 的 `report.json.result` 曾标记为 `incomplete`，原因是 stop marker 后 QMP 收尾阶段被 SIGKILL 清理，影响 host elapsed 和 guest instruction/block summary。因此性能结论只使用 marker window 内的 dd 输出、qperf samples、hotspot categories 和 `/proc/qperf_metrics` counters。这一点在报告中明确说明，避免过度解释不可靠字段。

### 6.9 这个闭环体现的 OS 理解

这次优化不是“看到 virtio 就改 virtio”。它经过了明确的 OS 层次判断：

1. 用户态 `dd bs=64k` 不代表 block driver 收到 64 KiB request，VFS 和 rsext4 data block cache 可能把读拆成 4 KiB。
2. `VirtQueue::add_notify_wait_pop` 热说明同步 virtqueue 开销大，但根因可能在上层 cache miss 粒度。
3. direct DMA 只减少 copy，不减少 request 数，所以不能解决主瓶颈。
4. rsext4 readahead 在 filesystem block cache 层减少 miss 次数，才真正减少 virtio request 和 notify/kick。
5. qperf counters 与 flamegraph 必须结合：火焰图显示热点，counters 解释事件规模，compare 证明优化效果。

这正是 harness 帮助下性能优化的核心价值：它让 OS 性能分析从“凭经验猜热点”变成“沿调用链、事件计数和 A/B 数据定位问题”。

## 七、VirtIO 优化与上游化：从 #783 到 virtio-drivers #249

### 7.1 #783：方向正确，但提交位置不合适

#783 的目标是优化 StarryOS virtio-blk I/O 路径，主要思路是：

- 将 `virtio-drivers` 中 virtio-blk queue size 从 16 增加到 256；
- 将 `VirtQueue::add()` 中过强的 `SeqCst` fence 放宽为 `Release` fence；
- 增加 bench-virtio-blk case；
- 基于 qperf 和 Linux 对照写性能分析文档。

从 virtio 语义看，两个方向都合理：

- 更大的 queue size 允许更多 in-flight requests；
- `Release` fence 足以保证 descriptor table 和 avail ring 写入先于 `avail.idx` store 对设备可见，更接近 Linux `dma_wmb()` / `virtio_wmb()` 的生产者排序需求；
- 在 RISC-V 上可从 `fence rw,rw` 放宽为 `fence rw,w`。

但 #783 在 tgoskits 中 vendoring 整个 `virtio-drivers` crate，并用 `[patch.crates-io]` 指向 `third_party/virtio-drivers`，这会带来同步维护成本。维护者建议把这类改动投到上游 `rcore-os/virtio-drivers`。

这个 review 反馈很重要：性能优化不仅要技术方向正确，还要落在正确的维护边界上。

### 7.2 virtio-drivers #249：两行核心改动的上游 PR

根据 [virtio-drivers #249](https://github.com/rcore-os/virtio-drivers/pull/249)，当前上游 PR 只改两个文件：

```text
src/device/blk.rs
src/queue.rs
```

内容是：

1. virtio-blk queue size `16 -> 256`；
2. `VirtQueue::add()` memory barrier `SeqCst -> Release`。

PR 中记录的 StarryOS riscv64 QEMU TCG 10MB 顺序 I/O benchmark 为：

| workload | before | after | 变化 |
| --- | ---: | ---: | ---: |
| READ 4K | 35.74 MB/s | 42.09 MB/s | +17.8% |
| WRITE 4K | 1.11 MB/s | 1.15 MB/s | +3.6% |

CI/check 显示 `check`、`build`、`examples (aarch64)`、`examples (riscv)`、`examples (x86_64)` 均通过。review 中 `qwandor` 提出了关键问题：是否使用 `write_blocks_nb/read_blocks_nb`，以及性能提升中 queue size 与 memory barrier 各自贡献多少，queue 从 16 到 256 是否过大。

这也指出了后续需要补强的证据：

- 分离 queue size 与 barrier 的 A/B 实验；
- 区分同步 `read_blocks` 与 nonblocking queue API；
- 对不同 queue size 做梯度测试，例如 16/32/64/128/256；
- 用 qperf metrics 或更下沉的 ring-level counters 证明 queue depth 真的被利用。

## 八、架构分析：StarryOS 兼容性与性能工作的系统视角

### 8.1 Linux ABI wrapper 层需要更强纪律

`pipe2`、`preadv2`、`ftruncate` 的共同点是：bug 都不在“功能完全缺失”，而在 Linux ABI 的小边界。

这说明 StarryOS 的 syscall 层不能只是把 `Sysno` 分发到 Rust 函数。它应承担 ABI wrapper 的职责：

- raw syscall 参数槽位解析；
- flags/mode 严格校验；
- 用户指针 copy-in/copy-out；
- errno 映射和错误优先级；
- 失败路径无副作用；
- legacy syscall 与新 syscall 的差异化处理。

长期看，应该把这些规则沉淀成 syscall wrapper 层的统一规范，而不是每个 syscall 分散实现。

### 8.2 VFS、rsext4 和 block layer 是性能瓶颈的共同体

virtio-blk 优化说明，block 性能不能只看 driver。用户态一次 `read()` 会穿过：

```text
sys_read
  -> fd/FileLike
  -> ax_fs_ng CachedFile
  -> rsext4 DataBlockCache
  -> block binding
  -> virtio-blk queue
  -> virtio-drivers VirtQueue
```

真正的瓶颈可能来自任意一层。qperf 最初指出 `VirtQueue::add_notify_wait_pop` 宽，但最终有效优化落在 rsext4 data block cache 层。这是典型的 OS 性能问题：底层热点不一定意味着底层就是最合适的改动点。

### 8.3 procfs 不只是调试输出，也是观测 ABI

`/proc/qperf_metrics` 在本次工作中承担了轻量 observability 出口。它不是 Linux 标准 ABI，但对实验工作流来说，它提供了稳定文本协议：

```text
QPERF_METRIC key=value ...
```

harness 可以解析这些指标并写入 `report.json`，再用于 compare。这个设计说明：OS 实验系统需要有面向工具链的观测接口，而且接口本身也要保持稳定、可解析、可 reset。

### 8.4 性能工具必须明确指标边界

qperf 的文档反复强调：

- TB mode 的 `executed_instructions` 是 QEMU guest instruction 计数，不是硬件 retired instructions；
- `--host-perf` 测的是宿主 QEMU 进程，不是 guest PMU；
- qperf counters 是 driver-visible 近似统计，不是 virtqueue ring-level 精确统计；
- marker window 是 postprocess timestamp filter，不是 runtime pause/resume。

这些限制没有削弱工具价值，反而让结论更可信。一个成熟的性能报告必须说明“能证明什么”和“不能证明什么”。

## 九、工程方法论总结

### 9.1 小 ABI 问题要用 Linux oracle 固化

本轮 syscall 工作最重要的经验是：Linux 兼容性不能只看当前应用是否跑通。`pipe2` invalid flags、`preadv2` flags 参数槽位、`ftruncate` 只读 fd errno 都是小问题，但它们会影响 libc、BusyBox、测试框架和真实应用的 feature detection。

因此每个修复都应包含：

```text
Linux 行为
  -> StarryOS 差异
  -> 最小修复
  -> regression probe/test
  -> harness 或 QEMU 验证
```

### 9.2 PR 粒度要服务 review

#476 和 #665 说明，过大的 PR 会把 syscall、qperf、BusyBox、文档混在一起，提高 review 成本。#665 最终合入，但 review 明确建议后续拆分。#990 当前也面临类似问题：94 个文件、harness、UI、metrics、深栈、优化、PPT 和大量 docs 放在一个 PR 中，技术内容很强，但 merge conflict 和 review 成本也很高。

后续更好的拆分方式是：

1. syscall harness CLI + probe；
2. qperf analyzer/CLI 增强；
3. `/proc/qperf_metrics` 与 driver counters；
4. full-stack callchain；
5. rsext4 readahead 优化；
6. UI/MCP/Knowledge Graph；
7. 文档和 PPT 素材。

这样每个 PR 都可以独立验证、独立回滚、独立 review。

### 9.3 性能优化要允许“失败尝试”被记录

direct-only fast path 最终退化，但它是有价值的实验。没有它，报告可能会错误地把“减少 copy”当成主优化方向。通过 A/B compare，我们证明 direct-only 没有减少 `add_notify_wait_pop_count`，从而把注意力转向 readahead。

优秀的性能报告不应该只记录成功结论，还应记录被数据排除的方案。

### 9.4 Harness 的价值不是替代理解，而是强制保留证据

harness 并不会自动告诉我们所有答案。它提供的是：

- 可重复命令；
- 结构化 artifact；
- Linux-vs-StarryOS 差分；
- qperf sample 与 flamegraph；
- workload metrics；
- driver-visible counters；
- compare 报告；
- UI/MCP 入口。

真正的 OS 理解仍然来自对 syscall 语义、VFS、block cache、virtqueue、内存屏障、用户指针和错误路径的分析。但 harness 能强制这些分析落到事实证据上。

## 十、最终成果与后续工作

### 10.1 已形成的成果

本次 Big Lab B 最终形成了以下成果链：

1. **Task1 OS 基础能力**  
   通过 5 个 ArceOS tutorial exercise 建立应用、`no_std` 兼容层、allocator、虚拟内存、VFS 的分层意识。

2. **Syscall 兼容性修复**  
   合入 #268 和 #665 中的 `pipe2`、`preadv/pwritev/preadv2/pwritev2` 修复，并在 #990 中补充 `ftruncate` fd 级 errno 修复。

3. **qperf 初步集成与增强**  
   #665 已合入 `cargo starry perf` 初版；#940 提供增强 qperf hotspot profiling，包括地址过滤、TB 采样、symtab fallback、flamegraph 和 diff。

4. **StarryOS syscall/qperf harness**  
   #990 新增 Docker-first harness、Linux probe、qperf profile、MCP、Web UI、Knowledge Graph 和结构化报告。

5. **qperf-driven virtio-blk 优化**  
   使用 marker/window、deep stack、driver counters 和 A/B compare 定位同步小 I/O 瓶颈，并通过 rsext4 data block readahead 将顺序读吞吐提升 25.33%，read request 数下降 84.51%，notify/kick 数下降约 79.89%。

6. **VirtIO 上游化尝试**  
   将 virtio-blk queue size 与 memory barrier 优化从 tgoskits vendoring 方案转向 `rcore-os/virtio-drivers` #249。

### 10.2 当前未完成与风险

当前仍需继续处理：

- #990 与最新 `dev` 的 block driver API 重构冲突，尤其是 `rd-block` 删除后 direct read 和 qperf metrics 需要适配新的 `rdif-block` / segment-based API；
- #990 中部分文档和 UI 示例仍有本机路径痕迹，需要替换为占位符；
- qperf candidate 中 QMP stop 后偶发 SIGKILL，导致部分 `report.json.result=incomplete`；
- `/proc/qperf_metrics` 仍是 driver-visible 近似 counters，后续可下沉到 ring-level；
- virtio-drivers #249 需要分离 queue size 和 barrier 的独立贡献，回应 reviewer 关于 queue size 过大的问题；
- `RWF_*` flags、真正 async blk pending queue、net RX 去 `copy_within()` 等仍是后续优化方向。

### 10.3 个人总结

这次实验让我对“OS 兼容性”和“OS 性能优化”有了更具体的理解。

兼容性方面，真正困难的不只是实现 syscall 表，而是对齐 Linux 用户态可以观察到的一切细节：flags、errno、参数槽位、用户指针、路径解析、fd 生命周期和失败路径副作用。

性能方面，真正困难的也不是看到某个函数宽就改某个函数。virtio-blk 的例子说明，底层 `VirtQueue::add_notify_wait_pop` 是热点，但有效优化点在上层 rsext4 data block cache。只有把火焰图、调用链、driver counters 和 A/B compare 合起来，才能判断“改动是否真的击中了瓶颈”。

最终，这轮工作的价值不只是一批 PR 或一份报告，而是形成了一套可以继续使用的 StarryOS 工作方法：

```text
用 Linux oracle 约束 syscall 语义，
用 qperf 和 counters 解释性能瓶颈，
用 harness 固化复现与验证路径，
用 PR review 约束工程边界，
用 OS 分层理解决定真正应该修改的位置。
```
