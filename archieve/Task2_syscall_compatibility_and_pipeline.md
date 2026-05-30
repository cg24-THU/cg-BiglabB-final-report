# Task2 细粒度实验报告：Syscall 兼容性修复与 Codex Pipeline

## 1. 任务定位

Task2 关注 StarryOS 的 Linux syscall 兼容性。它的核心不是“补几个 syscall 名字”，而是把 Linux 用户态可观察契约落实到 StarryOS：flags 要严格校验，errno 优先级要正确，raw syscall ABI 参数槽位不能错，失败路径不能产生副作用，用户指针和 fd 权限要在正确边界处理。

本任务同时吸收 Exp1 的 Codex pipeline 方法，因为 syscall 修复不是一次性人工列表，而是通过“Linux oracle + StarryOS replay + Reviewer + regression”持续推进。

## 2. 合并的源报告

| 源文件 | 本报告吸收内容 |
| --- | --- |
| `big_lab_b_report.md` | Codex pipeline、Exp2 syscall 修复方法、代表案例 |
| `reports/cg24_thu_syscall_pr_fix_report.md` | `pipe2`、`preadv/pwritev2`、`ftruncate` 与 syscall harness 的 PR 细节 |
| `reports/syscall_preadv_pwritev2_modification_report.md` | vectored positioned I/O 的 raw ABI、iovec 和测试细节 |
| `docs/cg24-thu-pr-analysis-report.md` | PR 状态、review 风险、合入关系 |
| `final_pre/oscope_final_pre/outline.draft.md` | Task2 PPT 叙事：修 syscall 是修可观察契约 |

重复合并说明：`pipe2`、`preadv/pwritev2`、`ftruncate` 在多份报告中反复出现，本报告按 syscall 契约统一整理，不重复罗列相同背景。

## 3. Codex Pipeline：从偶发补丁到工程闭环

Task2 的前置工作是构建 Codex 闭环 pipeline。角色拆分如下：

```text
Syncer
  -> 同步 tgoskits 到 upstream/dev
Developer
  -> 选择一个小目标，建立 Linux/StarryOS 差分，写测试和修复
Reviewer
  -> 独立复核 Linux 语义、测试覆盖和补丁风险
Committer
  -> 在 Reviewer PASS 后确定性提交、push、记录状态
```

这个设计的核心取舍是：让模型负责理解、定位、修复、解释，让脚本负责同步、schema、日志、patch、提交和状态记忆。

### 3.1 结构化输出

每个角色使用 JSON schema 输出 `decision`、`target`、`evidence`、`validation` 等字段。orchestrator 不需要猜自然语言结论，可以稳定决定下一步。

### 3.2 状态记忆

`journal.md` 和 `passed_commits.json` 被带入下一轮 prompt，避免模型重复修复已经通过的目标，也让每一轮能继承前一轮的证据。

### 3.3 Reviewer 否决权

Reviewer 独立复核 Linux/POSIX 语义。它不是形式检查，而是会否决“当前 case 通过但语义不完整”的补丁。例如 keepalive sockopt 不能只返回成功，`wait4` invalid options 不能消费 child，`epoll_wait` 不能跨阻塞点持有用户缓冲区引用。

## 4. Syscall 修复的共性模型

Task2 的 syscall 修复都可以归纳为下面的流程：

```text
选择一个小 ABI 点
  -> 在 Linux 上写 oracle 或查清语义
  -> 在 StarryOS 上复现差异
  -> 修最小代码路径
  -> 补 C regression 或 harness probe
  -> 运行 StarryOS QEMU / test-suit
  -> Reviewer 检查 errno、flags、副作用和边界
```

最终目标不是让某个程序“看起来跑了”，而是让 StarryOS 的可观察行为与 Linux 对齐。

## 5. 案例一：`pipe2` invalid flags 严格校验

### 5.1 Linux 契约

`pipe2(fds, flags)` 只接受支持的 flags。当前路径主要涉及：

- `O_CLOEXEC`
- `O_NONBLOCK`

如果传入未知 bit，例如 `O_CREAT` 或随机高位，Linux 行为是：

- 返回 `-1`；
- `errno = EINVAL`；
- 不创建 pipe fd。

### 5.2 StarryOS 旧问题

旧实现使用：

```rust
PipeFlags::from_bits_truncate(flags)
```

这个 API 会静默丢弃未知 bit。结果是 Linux 上应失败的调用在 StarryOS 上成功，并且还产生 fd 表副作用。

### 5.3 修改方案

PR #268 将解析改为严格模式：

```rust
let flags = PipeFlags::from_bits(flags).ok_or_else(|| {
    warn!("sys_pipe2 <= unrecognized flags: {flags}");
    AxError::InvalidInput
})?;
```

行为变化集中且可 review：

- 未知 flags 返回 `EINVAL`；
- 合法 `O_CLOEXEC`、`O_NONBLOCK` 路径不变；
- 失败时不创建 pipe；
- 修改点只在 `os/StarryOS/kernel/src/syscall/fs/pipe.rs`。

### 5.4 工程结论

这个案例说明 bitflags 的 `truncate` API 在 ABI 边界上很危险。对 syscall flags，未知位通常不是“忽略即可”，而是用户态探测内核能力的重要信号。

## 6. 案例二：`preadv/pwritev/preadv2/pwritev2` raw ABI 与语义

### 6.1 Linux 契约

这组 syscall 的难点不只是 vectored I/O，而是 raw syscall 参数槽位、offset 语义、flags 语义和 iovec 边界共同组成的契约。

| syscall | 参数要点 | offset 行为 | flags 行为 |
| --- | --- | --- | --- |
| `preadv` | raw ABI 带 `pos_l` / `pos_h` | positioned read，不更新 fd offset，负 offset 非法 | 无 flags |
| `pwritev` | raw ABI 带 `pos_l` / `pos_h` | positioned write，不更新 fd offset，负 offset 非法 | 无 flags |
| `preadv2` | `fd, vec, vlen, pos_l, pos_h, flags` | `offset == -1` 表示使用并更新当前 fd offset | `RWF_*` flags |
| `pwritev2` | `fd, vec, vlen, pos_l, pos_h, flags` | `offset == -1` 表示使用并更新当前 fd offset | `RWF_*` flags |

即使在 64 位实现中暂时不用 `pos_h`，dispatcher 也不能跳过这个参数槽位，否则真实 `flags` 会被读错。

### 6.2 旧问题

旧实现存在多层偏差：

1. dispatcher 没有完整传入 raw ABI 的 offset high 参数。
2. `preadv2/pwritev2` 曾把 `arg4` 当作 flags，漏读真实 `arg5`。
3. `pwritev2` positioned 写路径在原始 PR diff 中存在读写路径混用风险。
4. `offset == -1` 的 Linux 特殊语义处理不足。
5. 负 offset 缺少显式拒绝，可能经整数转换变成极大正数。
6. `O_APPEND` fd 上 positioned write 需要追加到文件尾，但不能更新当前 offset。
7. 非零 `RWF_*` flags 不能静默忽略。

### 6.3 修改方案

dispatcher 对齐 raw ABI：

```rust
Sysno::preadv => sys_preadv(..., uctx.arg3() as _, uctx.arg4())
Sysno::pwritev => sys_pwritev(..., uctx.arg3() as _, uctx.arg4())
Sysno::preadv2 => sys_preadv2(
    uctx.arg0() as _,
    uctx.arg1() as _,
    uctx.arg2() as _,
    uctx.arg3() as _,
    uctx.arg4() as _,
    uctx.arg5() as _,
)
```

语义拆分：

```text
preadv/pwritev:
  offset < 0 -> EINVAL
  offset >= 0 -> read_at/write_at

preadv2/pwritev2:
  flags != 0 -> EOPNOTSUPP
  offset < -1 -> EINVAL
  offset == -1 -> read/write 当前 fd offset
  offset >= 0 -> read_at/write_at 或 append 特殊处理
```

对 `O_APPEND`，positioned write 走 append 后端，但不更新当前 fd offset。

### 6.4 测试覆盖

新增 `test-suit/starryos/syscall/preadv_pwritev2.c`，覆盖：

| 类别 | 覆盖点 |
| --- | --- |
| 基础 positioned I/O | `pwritev(fd, ..., 5)` 写出预期内容，offset 不变 |
| positioned read | `preadv(fd, ..., 0)` 读取内容，offset 不变 |
| `preadv2/pwritev2 flags=0` | 与 `preadv/pwritev` 对齐 |
| `offset == -1` | 使用并更新当前 fd offset |
| iovec 边界 | `iovcnt=0`、零长度 iovec、坏 iov 指针、坏 base |
| fd 权限 | bad fd、readonly fd 写、writeonly fd 读 |
| flags | 未知 flags 返回 `EOPNOTSUPP` |
| `O_APPEND` | positioned write 追加，当前 offset 不变 |

### 6.5 工程结论

这个案例体现了 raw ABI 的严肃性。Rust 函数签名的直觉不能代替 syscall ABI，尤其不能在参数槽位、错误码和副作用上偷懒。

## 7. 案例三：`IoVectorBuf` copyin 前边界检查

vectored I/O 的 iovec 处理不仅要验证每个 base 指针，还要验证长度加总、地址加法和最大值边界。否则会出现：

- `base + len` 溢出；
- 总长度超过 `isize::MAX`；
- bad user pointer 在 copyin 后期才暴露；
- 部分 iovec 已产生副作用后才发现错误。

修复思路是将用户指针访问检查前置到 `IoVectorBuf::new()`，在真正读写之前形成稳定的内核侧描述。这个模式也迁移到后续 `poll/epoll_wait` 修复：跨阻塞点不能持有用户缓冲区引用。

## 8. 案例四：`ftruncate` readonly / `O_PATH` fd errno

### 8.1 Linux 契约

`ftruncate(fd, length)` 的错误语义包括：

| 场景 | Linux/POSIX 期望 |
| --- | --- |
| `length < 0` | `EINVAL` |
| fd 无效或已关闭 | `EBADF` |
| fd 未以写模式打开 | `EBADF` |
| `O_PATH` fd | `EBADF` |
| pipe/socket/目录等对象 | 通常 `EINVAL` |
| 成功 truncate | 返回 0，不改变当前 file offset |

readonly fd 返回 `EBADF` 很关键，因为它表示 fd 不具备该操作所需打开模式，而不是参数值非法。

### 8.2 修改方案

#990 中的实现将 fd mode 检查显式化：

```rust
let file = f.inner();
let flags = file.flags();
if flags.contains(FileFlags::PATH) {
    return Err(AxError::BadFileDescriptor);
}
if !flags.contains(FileFlags::WRITE) {
    return Err(AxError::BadFileDescriptor);
}
file.access(FileFlags::WRITE)?.set_len(length as _)?;
```

### 8.3 回归测试

`test-ftruncate` 的 readonly fd 场景从宽松接受 `EBADF || EINVAL` 收紧为严格 `EBADF`。测试覆盖正常缩小、扩展、截断到 0、负 length、bad fd、关闭 fd、readonly fd、pipe fd、目录 fd、超大 length 和错误优先级。

### 8.4 工程结论

`ftruncate` 说明 failure path 也是 ABI。Linux 不只规定成功时文件大小如何变化，也规定 fd 类型、权限和 length 组合下的 errno 优先级。

## 9. Syscall Harness 对 Task2 的方法论价值

#990 新增的 syscall harness 把手工差分固化为流程：

```text
编译 Linux probe
  -> 在 Linux 环境运行，得到 CASE 输出
  -> 交叉编译 StarryOS probe
  -> 注入 StarryOS rootfs
  -> QEMU 启动并运行 /root/syscall-probe
  -> 解析 Linux 与 StarryOS CASE 输出
  -> 生成 report.json，列出 differences
```

当前 `syscall_probe.c` 覆盖：

| probe | 目标语义 |
| --- | --- |
| `pipe2_invalid_flags` | invalid flags 应失败且不创建 pipe |
| `eventfd2_invalid_flags` | invalid flags 应失败且不创建 fd |
| `memfd_create_invalid_flags` | invalid flags 应失败且不创建 fd |
| `dup3_same_fd` | `oldfd == newfd` 的 Linux errno |
| `pwritev2_writes_data` | `pwritev2` 必须真的写入数据 |
| `ftruncate_readonly_fd` | readonly fd 不具备写 truncate 权限 |

probe 与 regression 的严格度可以不同：probe 用于快速发现跨环境差异，StarryOS regression 用于固定项目期望。

## 10. PR 状态与边界

| PR | 状态 | 与 Task2 的关系 |
| --- | --- | --- |
| #268 | Merged | 合入 `pipe2` invalid flags 严格校验 |
| #476 | Closed | 早期混合 PR，成熟内容后续进入 #665 |
| #665 | Merged | 合入 vectored I/O 兼容性、qperf 初版、BusyBox 测试增强 |
| #990 | Open | 包含 `ftruncate` 修复与 syscall/qperf harness，review 认可方向但整体 PR 仍受 qperf/driver 冲突影响 |

报告中需要区分“单个 syscall 修复的语义正确性”和“所在 PR 是否已整体合入”。例如 `ftruncate` 修复本身已按 review 调整为 `EBADF`，但 #990 作为大 PR 仍未合入。

## 11. 横向总结

Task2 的 syscall 修复形成了五条规则：

1. flags 必须严格校验，未知位不能静默截断。
2. errno 优先级是 ABI 的一部分。
3. raw syscall ABI 参数槽位不能用 Rust 函数直觉代替。
4. 用户指针应在 copyin/copyout 边界统一治理。
5. 成功路径和失败副作用同样重要。

## 12. 小结

Task2 最终建立的是一套 syscall 兼容性工程纪律：

```text
Linux oracle
  -> StarryOS 差分
  -> 最小修复
  -> regression / probe
  -> review
  -> 可重复 harness
```

这套纪律直接支撑 Task3 的真实应用兼容性，也为 Task5 的 OScope harness 奠定了工具基础。
