# Task1 细粒度实验报告：ArceOS Tutorial 与 OS 分层基线

## 1. 任务定位

Task1 是 Big Lab B 的工程热身，但它的价值不只是熟悉仓库命令。它把后续所有 StarryOS 兼容性修复需要的基本判断提前训练了一遍：用户态现象要沿调用链定位，修复要落在正确抽象层，最后必须用脚本和回归测试证明行为改变。

本任务实际完成 ArceOS tutorial 中 5 个 exercise：

| exercise | 训练层次 | 核心问题 |
| --- | --- | --- |
| `exercise-printcolor` | 应用输出与 console 字节流 | ANSI escape sequence 是终端协议，不是内核颜色对象 |
| `exercise-hashmap` | `no_std + alloc` 与 `axstd` 兼容层 | 用户程序看到的 `std` 可能是项目提供的兼容接口 |
| `exercise-altalloc` | 内核早期内存分配器 | 对齐、边界、页粒度与字节粒度要共同维护不变量 |
| `exercise-sysmap` | syscall、虚拟内存与用户地址空间 | `mmap` 不是“返回一个地址”，而是权限、flags、文件内容和页表映射的组合 |
| `exercise-ramfs-rename` | VFS 分发与具体文件系统语义 | `rename` 要穿过 `axstd -> axfs -> axfs_ramfs` 才能形成用户可见行为 |

最终执行官方逐题脚本和整体回归，5 个 exercise 全部通过。

## 2. 合并的源报告

| 源文件 | 本报告吸收内容 |
| --- | --- |
| `task1_os_learning_report.md` | 五个 exercise 的实现、调试路径和总体收获 |
| `big_lab_b_report.md` | Task1 对 Exp2、Exp3、Exp4 的方法论影响 |
| `final_pre/oscope_final_pre/outline.draft.md` | “五个练习不是小功能，而是 OS 层次训练”的展示结构 |
| `docs/tg-arceos-tutorial-knowledge-graph-analysis.md` | tutorial 与 OS 知识图谱节点之间的映射 |

重复内容已合并：`HashMap`、`mmap`、`rename` 等案例在多个总结文件中均出现，本报告按“现象、定位、修复层、验证、迁移价值”统一重写。

## 3. 实验环境与验证方式

Task1 的验证方式以官方 tutorial 脚本为准。每一题完成后单独运行对应 `scripts/test.sh`，最后执行整体回归脚本。这个过程带来了两个后续实验一直沿用的约束：

1. 不能只看代码认为“应该对”，必须执行真实测试入口。
2. 每个修复要对应一个可复现命令，避免只留下自然语言描述。

Task1 使用容器环境完成原始验证，本轮最终归档不重新修改源代码，只整理现有报告材料。

## 4. `exercise-printcolor`：输出路径中的协议边界

### 4.1 现象与目标

题目要求输出 `Hello, Arceos!`，并让输出包含颜色控制效果。最终实现是在应用层输出 ANSI escape sequence：

```rust
println!("\x1b[1;32m[WithColor]: Hello, Arceos!\x1b[0m");
```

### 4.2 分层判断

这题容易被低估。颜色效果并不是内核 console 主动理解“绿色”或“高亮”，而是应用把一串字节写到 stdout，console/serial 路径负责传递字节，终端解释 ANSI 控制序列。

因此最小正确修改位于应用层。如果改 `println!` 宏、console driver 或串口驱动，反而扩大了影响面。

### 4.3 迁移价值

后续 syscall 兼容性修复也反复遇到类似问题：修复必须落在正确层次。比如 invalid flags 应在 syscall 层严格解析，`/proc/net/arp` 应在 procfs 文本 ABI 层补齐，virtio-blk 性能问题则要靠 fs cache 和 driver 路径共同解释。

## 5. `exercise-hashmap`：`axstd` 兼容层不是完整 Rust std

### 5.1 现象与目标

应用需要使用：

```rust
use std::collections::HashMap;
```

初始环境中 `axstd = 0.3.0-preview.1` 没有导出 `HashMap`，编译报 `unresolved import`。

### 5.2 修改策略

正确修复不是把应用里的 `HashMap` 改成 `BTreeMap`，而是通过本地 patch 覆盖 `axstd`，在 `axstd::collections` 暴露兼容接口：

```toml
[patch.crates-io]
axstd = { path = "./axstd" }
```

再由 `hashbrown` 提供实际哈希表实现：

```rust
pub use hashbrown::{HashMap, HashSet, hash_map, hash_set};
```

### 5.3 OS 认识

在 unikernel 或内核环境中，用户看到的 `std` 不一定是官方完整标准库。它可能是项目基于 `core`、`alloc` 和平台服务实现的一组兼容接口。用户程序依赖的是路径和行为契约，内部可以由 `hashbrown`、`alloc` 或项目自有代码实现。

这个经验后来迁移到 Linux ABI 修复：不能让应用绕过内核缺陷，而应补齐内核承诺给用户态的接口。

## 6. `exercise-altalloc`：早期分配器的不变量

### 6.1 设计模型

实现采用双端 bump allocator：

```text
[ bytes-used | available area | pages-used ]
start       b_pos           p_pos        end
```

核心状态：

```rust
pub struct EarlyAllocator<const PAGE_SIZE: usize> {
    start: usize,
    end: usize,
    b_pos: usize,
    p_pos: usize,
    count: usize,
}
```

字节分配从低地址向高地址增长，页分配从高地址向低地址增长。两者不能交叉。

### 6.2 关键边界

实现中最重要的不是“返回一个可用地址”，而是维护以下不变量：

| 不变量 | 作用 |
| --- | --- |
| `b_pos <= p_pos` | 防止字节分配区与页分配区重叠 |
| `align_up` 与 `align_down` | 保证不同分配粒度满足对齐要求 |
| `checked_add` / `checked_sub` | 防止整数溢出变成错误可用区间 |
| `count` | 记录分配状态，辅助调试和释放限制 |

### 6.3 后续影响

后续 StarryOS 修复中，很多 bug 都不是功能缺失，而是不变量破坏。例如用户 iovec 长度求和溢出、`mmap` offset/length 校验顺序、`epoll_wait` 阻塞期间持有用户指针等。Task1 的 allocator 训练让这些问题有了共同的分析语言。

## 7. `exercise-sysmap`：从 `mmap` 看 Linux ABI 组合语义

### 7.1 任务本质

`mmap` 的返回值表面是一个地址，但内核真正要建立的是一段可访问的用户虚拟地址空间。关键输入包括：

- `length` 是否为 0；
- `offset` 是否按页对齐；
- `PROT_READ/WRITE/EXEC` 如何转成页表权限；
- `MAP_PRIVATE/MAP_SHARED/MAP_FIXED/MAP_ANONYMOUS` 如何组合；
- file-backed mapping 是否把文件内容填入用户地址；
- 返回地址后用户程序是否真的能读写。

### 7.2 定位方法

这题训练了两类定位：

1. syscall 参数进入内核后的边界检查。
2. 返回用户态后是否真正建立了对应映射。

只让 syscall 返回成功不够，用户态继续访问映射地址时不能 page fault，文件映射内容也必须可观察。

### 7.3 迁移价值

后续 Exp2 中 `mmap(fd=0)` 修复直接依赖这类认识。fd 0 只是文件描述符数值，不天然代表无效；`MAP_ANONYMOUS` 与 file-backed mapping 的 fd 规则也不能混在一起。这类 ABI 兼容问题必须按 Linux 可观察语义处理，而不是按直觉处理。

## 8. `exercise-ramfs-rename`：VFS 分发链路

### 8.1 任务本质

`std::fs::rename` 的调用链不是单点函数，而是：

```text
user std-like API
  -> axstd
  -> axfs
  -> VFS node dispatch
  -> axfs_ramfs directory node
```

如果只在具体 ramfs 节点实现 rename，但上层组合根目录没有正确转发，用户态仍然看不到完整行为。

### 8.2 语义细节

`rename` 不是简单修改路径字符串。它涉及：

- 源路径是否存在；
- 目标路径是否存在；
- 目标存在时覆盖语义；
- 跨目录移动；
- 目录和文件类型限制；
- 上层 mount/VFS 是否允许下发；
- 错误码是否稳定。

### 8.3 迁移价值

后续 BusyBox `/proc/net/arp`、route netlink、tmpfs cwd cleanup 等问题本质上都要求沿 VFS、procfs、socket 或文件系统实现链路定位。Task1 的 `rename` 训练说明：看见用户态 API 名字只是开始，真正修复要找到最终承担语义的系统层。

## 9. Task1 形成的方法论

Task1 的关键结论可以概括为五点。

| 结论 | 后续迁移 |
| --- | --- |
| 用户态 API 与内核实现不是一一对应 | `std::fs::rename`、BusyBox applet、Codex CLI 都会跨多层 |
| 正确性大量体现在边界条件 | flags、errno、offset、用户指针、fd 类型都必须检查 |
| 抽象层职责要清楚 | 应用层、兼容层、VFS、driver 的修复位置不同 |
| 测试脚本是事实来源 | 后续 Linux oracle、StarryOS regression、qperf A/B 都沿用这一点 |
| 小实验可以训练大系统判断 | Task1 的分层定位直接支撑 Exp2 到 Task5 |

## 10. 与 OS 知识图谱的关系

后续 OScope Knowledge Graph 对 tutorial 仓库做静态扫描时，Task1 对应到以下知识节点：

| exercise | 图谱节点 |
| --- | --- |
| `printcolor` | Unikernel Runtime、Console、User Output |
| `hashmap` | `no_std`、Allocator、Rust Compatibility Layer |
| `altalloc` | Allocator、Memory Management、Page Granularity |
| `sysmap` | User Address Space、Page Table、Syscall ABI |
| `ramfs-rename` | VFS、File I/O、Path Resolution |

这说明 Task1 不只是“做了五道题”，而是把后续报告中的 OS 子系统词汇提前落到了代码和验证路径上。

## 11. 小结

Task1 最终沉淀的是一种 OS 工程工作方式：

```text
看用户态现象
  -> 沿 syscall / runtime / VFS / driver 链路定位
  -> 在正确抽象层做最小修复
  -> 用可复现脚本确认行为
  -> 把边界条件转化为长期 regression
```

这个工作方式贯穿后续 Task2 的 syscall 兼容、Task3 的真实应用支持、Task4 的 qperf 性能优化和 Task5 的 OScope harness。
