# Big Lab B Task 报告归档索引

> 归档日期：2026-05-30
> 归档位置：`final_pre/archieve/`
> 说明：既有总结报告 `cg24_thu_big_lab_b_report.md` 与 `final_version.pptx` 保持原样。本索引只说明本轮新增 Task1-5 细粒度报告如何合并现有素材。

## 1. 本轮新增归档

| 文件 | 主题 | 主要用途 |
| --- | --- | --- |
| `Task1_arceos_tutorial_os_baseline.md` | ArceOS tutorial 与 OS 分层训练 | 解释 Task1 五个 exercise 如何建立后续 syscall、VFS、内存、测试闭环基础 |
| `Task2_syscall_compatibility_and_pipeline.md` | syscall 兼容性修复与 Codex pipeline | 合并 Exp1 pipeline、Exp2 syscall PR、Linux oracle 和 StarryOS regression 方法 |
| `Task3_real_application_semantics_busybox_codex.md` | 真实应用兼容性：BusyBox 到 Codex CLI | 把 BusyBox applet 与 Codex CLI 压力测试统一成应用级 ABI 报告 |
| `Task4_qperf_tooling_and_virtio_optimization.md` | qperf 工具链与 virtio 性能优化 | 合并 qperf 集成、marker window、深调用栈、counters、A/B compare 与 blk 优化 |
| `Task5_oscope_harness_skill_mcp_delivery.md` | OScope harness、Skill、MCP、UI 与交付 | 细化 harness 实现、Codex 集成、知识图谱、视觉演示和最终交付边界 |

视觉素材复制到 `assets/`，来源为 `final_pre/ppt_assets/`。新增报告统一使用相对路径 `assets/*.png`，便于独立报告仓库中直接渲染。

## 2. 已保留不变的总结报告

| 文件 | 处理方式 |
| --- | --- |
| `cg24_thu_big_lab_b_report.md` | 保留原始总结报告，不改写、不重排 |
| `final_version.pptx` | 保留原始最终展示 PPT，不改写 |

## 3. 素材去重与合并规则

本轮整理按“主题事实唯一、证据来源可追溯、报告叙事分任务展开”的规则合并重复内容。

1. 相同背景段落只保留一次。例如 qperf 的定位、QEMU TCG plugin 边界、marker/window 机制在多个文档中重复出现，最终放入 Task4。
2. 相同实验数据优先采用最完整的最终 A/B 结果。例如 virtio-blk readahead 使用 `qperf-virtio-blk-deepstack-optimization-report.md` 与 `qperf_harness_work_summary.md` 中一致的最终表格。
3. 阶段性失败不删除，而是转化为方法论证据。例如 direct-only read fast path 退化，被保留在 Task4 中说明“直觉优化必须经过 A/B 验证”。
4. 既有总结报告不合并、不覆盖，只作为 Task 报告的上位背景来源。
5. 通用模块 README、CHANGELOG、自动生成 crate 文档不作为最终实验报告主体，仅在涉及具体任务时作为背景路径。

## 4. 源报告分组

### 4.1 总结与展示材料

| 源文件 | 合并去向 |
| --- | --- |
| `big_lab_b_report.md` | Task1、Task2、Task3、Task5 |
| `final_pre/archieve/cg24_thu_big_lab_b_report.md` | 保留不变，仅作为既有总结 |
| `final_pre/oscope_final_pre/outline.draft.md` | Task1-Task5 的最终叙事结构 |
| `oscope_defense_speaker_script.md` | Task1-Task5 的答辩表述和追问准备 |
| `oscope_harness_gpt_image2_prompts.md` | Task5 的视觉表达边界 |

### 4.2 Task1 基础训练

| 源文件 | 合并去向 |
| --- | --- |
| `task1_os_learning_report.md` | Task1 主体 |
| `docs/tg-arceos-tutorial-knowledge-graph-analysis.md` | Task1 的知识图谱映射和教学仓库理解 |
| `docs/starryos-local-run-and-qperf-guide.md` | Task1 的本机运行环境经验，部分也进入 Task4 |

### 4.3 syscall 兼容性与 PR 证据

| 源文件 | 合并去向 |
| --- | --- |
| `reports/cg24_thu_syscall_pr_fix_report.md` | Task2 主体 |
| `reports/syscall_preadv_pwritev2_modification_report.md` | Task2 的 vectored I/O 细节 |
| `docs/cg24-thu-pr-analysis-report.md` | Task2 的 PR 状态、范围、review 风险 |
| `docs/starry-syscall-harness.md` | Task2 的 harness 入口概览，详细实现放入 Task5 |

### 4.4 真实应用兼容性

| 源文件 | 合并去向 |
| --- | --- |
| `big_lab_b_report.md` 中 Exp3/Exp4 | Task3 主体 |
| `oscope_defense_speaker_script.md` 中 BusyBox/Codex CLI 段落 | Task3 的答辩表述 |
| `final_pre/oscope_final_pre/outline.draft.md` 中 Task3 | Task3 的 PPT 叙事 |

### 4.5 qperf 与性能优化

| 源文件 | 合并去向 |
| --- | --- |
| `qperf_harness_work_summary.md` | Task4 主体 |
| `docs/qperf-cargo-starry-integration-report.md` | Task4 的 cargo starry 集成 |
| `docs/qperf-cargo-starry-integration.md` | Task4 的使用入口 |
| `docs/qperf-tooling-validation-report.md` | Task4 的 marker/counter/compare 验收 |
| `docs/qperf-tooling-redesign.md` | Task4 的设计目标 |
| `docs/qperf-tooling-improvement-report.md` | Task4 的字段、分类和示例 |
| `docs/qperf-callchain-flamegraph-design.md` | Task4 的深调用栈设计 |
| `docs/qperf-callchain-validation-report.md` | Task4 的深调用栈验证 |
| `docs/qperf-flamegraph-guide.md` | Task4 的火焰图使用说明 |
| `docs/qperf-current-blk-bottleneck-analysis.md` | Task4 的 blk baseline 分析 |
| `docs/qperf-virtio-performance-analysis.md` | Task4 的早期 virtio 分析 |
| `docs/qperf-virtio-drivers-performance-report.md` | Task4 的 blk/net/vsock 对照 |
| `docs/qperf-host-rerun-virtio-bottleneck-report.md` | Task4 的宿主重跑和瓶颈排序 |
| `docs/qperf-virtio-optimization-report.md` | Task4 的早期优化尝试 |
| `docs/qperf-virtio-blk-deepstack-optimization-report.md` | Task4 的最终 blk readahead A/B |
| `docs/qperf-work-handoff.md` | Task4 的交付清单和后续任务 |
| `docs/qperf-marker-and-metrics-usage.md` | Task4 的 marker/counter 使用方式 |
| `docs/qperf-sampling-debug-notes.md` | Task4 的采样可信性与局限 |
| `docs/qperf-starryos-integration-report.md` | Task4 的 qperf 初版接入 |

### 4.6 OScope、harness、MCP、Skill 与知识图谱

| 源文件 | 合并去向 |
| --- | --- |
| `final_pre/harness_implementation_skill_mcp_detail.md` | Task5 主体 |
| `qperf_harness_work_summary.md` 中 UI/MCP/Knowledge 部分 | Task5 |
| `docs/ai-harness-development-framework.md` | Task5 的自治开发框架 |
| `docs/harness-os-knowledge-graph.md` | Task5 的知识图谱实现 |
| `tools/starry-syscall-harness/README.md` | Task5 的用户入口 |
| `tools/qperf/README.md` | Task4 与 Task5 的工具说明 |

### 4.7 系统工程审计类材料

| 源文件 | 合并去向 |
| --- | --- |
| `reports/spin-no-preempt-audit.md` | Task5 的后续风险与锁治理路线 |
| `reports/external-spin-audit.md` | Task5 的工程治理附录素材 |
| `reports/external-spin-migration-plan.md` | Task5 的后续改进路线 |

这些审计报告与 Task1-5 主线不是一一对应的功能交付，但它们体现了后续维护中的锁语义、外部依赖和 migration 风险，因此只在 Task5 的“能力边界与后续工作”中吸收结论，不展开成独立 Task。

## 5. 未作为主体合并的文件类型

以下文件未纳入 Task 报告主体：

- 仓库中通用 `README.md`、`README_CN.md`、`CHANGELOG.md`。
- `docs/docs/components/crates/*` 下的自动生成 crate 文档。
- 硬件外设、驱动子模块的普通 README，除非被 qperf 或应用兼容性报告明确引用。
- `.DS_Store`、构建产物、渲染中间件。

这些文件数量很大，但多数是项目背景文档，不是本次 Big Lab B 最终报告素材。若强行逐项合并，会稀释 Task1-5 的实验结论。
