# Task3 细粒度实验报告：真实应用兼容性，BusyBox 到 Codex CLI

## 1. 任务定位

Task3 关注真实应用如何把多个 Linux ABI 面组合起来。相比 Task2 直接修单个 syscall，Task3 的入口是应用失败现象：BusyBox applet、git、curl、Codex CLI 等程序会同时触发 procfs、netlink、packet socket、poll/epoll、进程生命周期、TLS、文件系统和 socket options。

本报告把原 Big Lab B 中 Exp3 BusyBox 与 Exp4 Codex CLI 合并为“真实应用兼容性”任务。这样可以避免重复讲同一方法论：应用级失败不是终点，必须继续下钻到底层可观察 ABI。

## 2. 合并的源报告

| 源文件 | 本报告吸收内容 |
| --- | --- |
| `big_lab_b_report.md` | Exp3 BusyBox、Exp4 Codex CLI 的完整叙事 |
| `final_pre/oscope_final_pre/outline.draft.md` | Task3 PPT 结构：BusyBox 语义扩展 |
| `oscope_defense_speaker_script.md` | BusyBox 与 Codex CLI 的答辩表述 |
| `docs/cg24-thu-pr-analysis-report.md` | 与 BusyBox、qperf、harness 相关 PR 背景 |

重复合并说明：BusyBox 和 Codex CLI 在总报告和讲稿中都作为“真实应用压力测试”出现，本报告统一按“应用现象、依赖 ABI、修复策略、验证风险”整理。

## 3. 从 syscall case 到应用 oracle

Task2 的 syscall probe 通常只有一个小目标，例如 `pipe2(O_CREAT)` 应返回 `EINVAL`。Task3 中，一个 BusyBox applet 可能同时依赖多个接口：

```text
busybox ip link
  -> socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)
  -> send RTM_GETLINK
  -> recv RTM_NEWLINK records
  -> parse ifinfomsg + attributes
  -> format stdout
```

应用 oracle 的优点是能发现组合语义缺口；风险是如果只匹配 stdout，容易接受假阳性。因此 Task3 采用两层验证：

1. 应用级命令输出与退出码要接近 Linux。
2. 底层 regression 要固定协议字段、errno、fd 状态或 procfs 内容。

## 4. BusyBox 兼容性成果

Exp3 以 BusyBox issue backlog 为入口，最终形成 7 个已合入 PR：

| PR | BusyBox 目标 | 主要缺口 |
| --- | --- | --- |
| #477 | `nice` | priority syscall |
| #479 | `ip link` | route netlink `RTM_GETLINK` |
| #480 | `arp` | `/proc/net/arp` |
| #481 | `ip addr` | route netlink `RTM_GETADDR` |
| #482 | `pidof` | `/proc/<pid>` 中用户可见 PID |
| #483 | `iplink` | standalone applet 的 link dump 覆盖 |
| #484 | `arping` | `AF_PACKET` 与 ARP synthetic reply |

这些 PR 的共同点是：它们都不是“加一个 syscall 表项”就能解决，而是要补齐应用可观察的 Linux 语义。

## 5. 案例一：`ip link` 与 `ip addr` 背后的 route netlink

### 5.1 应用现象

BusyBox `ip link` 和 `ip addr` 表面上只是打印接口列表和地址，但它们的输入不是普通文本文件，而是 netlink socket。

### 5.2 依赖 ABI

`ip link` 需要：

- 接收 `RTM_GETLINK` 请求；
- 返回 `RTM_NEWLINK` records；
- 合成 loopback/eth0 的接口名、MTU、MAC、operstate 等 attributes。

`ip addr` 需要：

- 接收 `RTM_GETADDR` 请求；
- 返回 `RTM_NEWADDR` records；
- 合成 loopback/eth0 的 IPv4 地址属性。

### 5.3 修复原则

StarryOS 可以先不实现完整 Linux 网络栈，但常用查询路径必须返回格式正确、字段足够的最小响应。这里的 ABI 是协议格式，而不只是 syscall 存在。

### 5.4 风险

如果只让 stdout 出现 `lo` 或 `eth0`，可能掩盖 attribute 长度、类型、对齐、顺序等错误。协议类测试应检查 netlink message header 与关键 attributes。

## 6. 案例二：`/proc/net/arp` 是文本 ABI

BusyBox `arp` 依赖 `/proc/net/arp`。这个路径说明 `/proc` 不是随意调试输出，而是稳定用户态接口。

最小兼容要求：

| 项 | 要求 |
| --- | --- |
| 路径 | `/proc/net/arp` 必须存在 |
| header | 字段名和顺序应被 BusyBox 解析 |
| 内容 | IP、HW type、flags、HW address、mask、device 格式稳定 |
| 空表 | 即使没有真实 ARP cache，也应返回 Linux 兼容 header |

这个案例说明，文本格式本身也可以是 ABI。输出多一个字段、少一个空格、header 不匹配，都可能让真实应用失败。

## 7. 案例三：`pidof` 与用户可见 PID

BusyBox `pidof` 依赖 `/proc/<pid>` 枚举，并期望 init 可见为 PID 1。旧实现中部分 procfs 路径暴露内部 task/thread id，而不是用户可见 process id。

这类问题的本质是：内核内部对象标识不能直接泄漏到 Linux 用户态 ABI。

| 内核内部概念 | 用户态期望 |
| --- | --- |
| task id / thread id | process id |
| scheduler 内部状态 | `/proc` 可见进程状态 |
| zombie 内部对象 | wait 前仍可被部分 syscall 查询 |

这个认识后来迁移到 Codex CLI 触发的 zombie 可见性修复。

## 8. 案例四：`arping` 与 packet socket

`busybox arping` 会打开 `AF_PACKET` socket，绑定接口，发送 ARP request，再等待 ARP reply。

最小兼容面包括：

- packet socket 创建；
- 接口查询；
- bind / getsockname；
- packet send / recv；
- synthetic ARP reply；
- SHA、SPA、THA、TPA 等字段正确。

这个 PR 的 review 价值在于防止“stdout 包含 Received 就算过”。协议类修复必须检查底层字段，否则应用级 oracle 会变成假阳性。

## 9. BusyBox 阶段的方法论

BusyBox 阶段形成了三条规则：

1. 真实应用可以暴露 syscall probe 不容易覆盖的组合语义。
2. 应用 stdout 只能作为入口，不能替代底层 regression。
3. `/proc`、netlink、packet socket、PID 可见性都是稳定 ABI，不是实现细节。

## 10. Codex CLI：更复杂的真实应用压力测试

Exp4 的目标是让 Codex CLI 在 StarryOS x86_64 QEMU guest 中完成基础 coding-agent 工作流：

- 启动 Linux x86_64 musl 版本 Codex CLI；
- 运行 `codex --help`、`codex exec --help`、`codex login status`；
- 注入 `auth.json`、CA 证书和代理后在线请求模型；
- 在 guest 内使用 `git`、`rg`、shell 命令读写文件；
- 在 StarryOS guest 内 clone 或注入 TGOSKits 代码，让 Codex 阅读、修改和验证 StarryOS/TGOSKits。

Codex CLI 比 BusyBox 更复杂。它会触发多线程、网络、TLS、git、子进程、poll/epoll、zombie 进程、cwd、TCP sockopts 等路径。

## 11. Codex CLI 阶段拆分

真实接入按阶段推进：

| 阶段 | 目标 | 暴露的问题 |
| --- | --- | --- |
| A | 基线环境和可观测性 | QEMU、Alpine rootfs、DNS、HTTPS、CA、`/tmp`、`/dev/null` 等前置条件 |
| B | 注入 Codex 与离线 smoke | `codex --version`、`--help`、`rg --version` |
| C | auth、TLS、HTTPS、最小在线请求 | `epoll_wait` 跨阻塞点持有用户 buffer |
| D | 小 workspace 读写、`rg` 和 shell | `poll/ppoll` 用户 buffer 引用问题 |
| E | 扩大到 TGOSKits 源码子集 | git 仓库发现、输出过长、prompt 稳定性 |
| F | 按 Codex 日志补兼容缺口 | `getcwd`、keepalive、zombie 可见性等 |
| G | 无 secret smoke 工程化 | 从默认 CI 调整为 opt-in example |
| H | TGOSKits 子任务试跑 | guest 内读写源码、检查 diff |
| I | 演示整理和 failure-path 清理 | tmpfs cwd cleanup panic |

这种阶段化方式避免了一开始就把所有问题混在一起，每一步只扩大一个维度。

## 12. Codex CLI 案例一：`epoll_wait` 用户缓冲区跨阻塞点

### 12.1 现象

Codex 在线请求模型时触发 `epoll_wait`。旧实现可能在阻塞等待期间持有用户态 `events` buffer 的内核引用。

多线程程序在等待期间可能由另一个线程 `munmap` 或改变 buffer 页权限。唤醒后内核继续写旧地址，会触发 kernel page fault。

### 12.2 Linux 语义

Linux 不跨阻塞点持有用户引用，而是在内核侧维护临时 event buffer，返回用户态前再逐项 copy_to_user。用户 buffer 失效时返回 `EFAULT`，不能 panic。

### 12.3 修复方向

修复拆入 PR #523，并补充 `bug-epoll-wait-user-buffer-race` 回归。`poll/ppoll` 也按类似思路处理：入口 copy-in 到内核副本，等待期间只改内核数据，返回时 checked copy-out。

## 13. Codex CLI 案例二：raw `getcwd` ABI

`git` 暴露了 raw `getcwd` ABI 问题。Linux raw syscall 成功时返回包含末尾 NUL 的长度；小 buffer 返回 `ERANGE`；只有 buffer 足够大但用户指针无效时才返回 `EFAULT`。

旧实现的问题包括：

- 返回长度不含 NUL；
- null buffer 过早返回 `EFAULT`；
- 错误优先级不符合 Linux。

这个案例说明，ABI 修复不能只覆盖成功路径。参数组合下的错误优先级同样会被真实程序观察到。

## 14. Codex CLI 案例三：TCP keepalive 不能伪兼容

Codex/curl/hyper 会设置：

- `TCP_KEEPIDLE`
- `TCP_KEEPINTVL`
- `TCP_KEEPCNT`
- `TCP_USER_TIMEOUT`

如果 `setsockopt` 只返回成功但不保存状态、不支持 `getsockopt` 读回，应用仍可观察差异。进一步地，`TCP_KEEPIDLE` 不仅要保存，还要接入 smol socket 的 keepalive idle timeout，否则底层行为仍与设置不符。

这代表一类“伪兼容”风险：返回成功不等于兼容。

## 15. Codex CLI 案例四：zombie wait 前仍可见

Codex 执行任务会创建和清理子进程。旧实现中，未被 `waitpid` 回收的 zombie child 在某些查询路径中过早不可见，导致：

- `getpgid`；
- `getsid`；
- `kill(pid, 0)`；
- `kill(pid, SIGKILL)`；

行为与 Linux 不一致。

Linux 中 zombie 虽然已退出，但在 parent wait 前仍保留进程表可见性和权限判断所需信息。修复思路是保留 unreaped zombie 的 `Process` 和凭据快照。

## 16. Codex CLI 案例五：tmpfs cwd cleanup failure path

子进程 cwd 指向 tmpfs 目录时，父进程先 `rmdir`，子进程退出清理路径可能在 atomic context 中获取睡眠锁并 panic。

修复方向：

- `MemoryNode::drop` 不再清理目录 entries；
- tmpfs inode slab/metadata 使用适合上下文的锁；
- `unlink`/`rename` 在正常 syscall 上下文提前清理被移除目录项。

这个案例说明，真实应用不仅测试主路径，还会触发资源回收和失败路径。

## 17. PR 拆分

Codex CLI 相关工作按中等粒度拆分：

| PR | 主题 | 说明 |
| --- | --- | --- |
| #523 | `poll` / `epoll_wait` 用户缓冲区安全 | 修复阻塞期间用户 buffer 被 unmap 后 kernel page fault |
| #524 | CLI 兼容缺口 | 覆盖 `getcwd`、`PR_CAPBSET_READ`、TCP keepalive、zombie 可见性 |
| #525 | tmpfs cwd cleanup panic | 修复 failure/drop path panic |
| #575 | Codex CLI example flow | 调整为 `examples/starry/codex-cli` opt-in example，不进入默认 CI |

拆分的好处是 review 可以逐个对照 Linux 语义和 regression，demo/example 也不需要把 auth、rootfs、大二进制或在线请求放进 CI。

## 18. Task3 总结

Task3 证明真实应用兼容性有三层：

```text
应用输出与退出码
  -> 协议 / procfs / fd / socket / process ABI
  -> 内核状态与 failure path
```

BusyBox 让 StarryOS 暴露出 `/proc`、netlink、packet socket 和 PID 可见性缺口；Codex CLI 进一步暴露多线程用户指针、网络 sockopt、zombie 生命周期和 tmpfs cleanup 等复杂路径。

最终形成的工程原则是：真实应用是最好的缺口发现器，但最终修复仍必须回到 Linux 可观察契约和长期 regression。
