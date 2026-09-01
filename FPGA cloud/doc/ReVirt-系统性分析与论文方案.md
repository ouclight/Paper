# ReVirt：系统性分析与论文方案

## 0. 核心结论

五份专利和一份 PPT 并不是六个彼此独立的论文点，而是在不同时间分别解决同一个系统中的不同问题。把它们统一起来后，可以形成一套完整的节点内 FPGA 云虚拟化架构：

> **ReVirt 将一个硬件实现 Virtio 的 SR-IOV VF 抽象为稳定的租户级虚拟加速器。VF 后端通过多对多 Binding Context 动态连接可重构 AFU：一个 VF 可以聚合多个 AFU，一个 AFU 也可以通过独立上下文被多个 VF 非抢占式共享。PF 管理面负责部分重配置、资源分配、原子绑定和故障回滚，VF 数据面保持硬件直通。**

这一系统同时解决四类矛盾：

1. **稳定接口与可变硬件之间的矛盾：** Virtio ABI 和 PCIe VF 保持稳定，AFU 可以在线变化。
2. **功能组合与资源隔离之间的矛盾：** 一个 VF 可以包含多个 AFU，但所有 AFU 只能访问该租户的 DDR、队列和事件空间。
3. **高性能直通与高利用率共享之间的矛盾：** 虚拟机直接访问硬件 VF，同时共享 AFU 在硬件中按上下文调度。
4. **云端按需服务与 FPGA 静态部署之间的矛盾：** 云控制面按功能请求选择、装载和绑定 AFU，而不是把整张 FPGA 固定分配给用户。

最适合当前论文的定位是：

> **面向 FPGA PaaS 的节点内虚拟加速器执行层。**

论文不应宣称已经实现完整的跨节点 FPGA 云、SIOV、热迁移或通用功能链平台，除非确有对应原型与实验。

## 1. 六份材料在统一系统中的位置

| 材料 | 原始问题 | 核心机制 | 统一系统中的位置 | 是否适合作为主要贡献 |
|---|---|---|---|---|
| 动态配置 VF 功能专利 | VF 与算法静态绑定，更新需重新枚举或重启 | 静态 VF 与动态 AFU 解耦、映射表、部分重配置 | 动态后端与在线更新 | 是 |
| 功能可组合架构专利 | 一个 VF 只能绑定一个 AFU | 一个 VF 聚合多个 AFU，共享 BAR/MSI-X/DDR | 空间聚合与租户级资源命名空间 | 是 |
| Virtio 统一接口专利 | 不同设备接口和驱动不一致 | 硬件 Virtio PCI、virtqueue、统一能力发现 | 稳定北向 ABI 和硬件数据面 | 是 |
| VF 共享利用率专利 | AFU 只服务一个 VF，空闲时无法复用 | 一个 AFU 多上下文、跨 VF 非抢占调度 | 时间复用、利用率和 QoS | 是 |
| 中断扩展专利 | Virtio 队列通知无法低成本承载所有 AFU 异步事件 | 额外 MSI-X 与 Virtio/PCI 中断协同 | 事件子系统工程实现 | 否，除非实验收益突出 |
| FPGA 云基础设施 PPT | 如何以云服务形式交付 FPGA 能力 | PaaS、AFU 仓库、云调度、SIOV/池化演进 | 系统背景、控制面和未来路线 | 当前论文主要作为背景 |

因此，论文贡献不应按“每份专利一个贡献”排列。更合理的做法是将其重组成三个架构贡献和一个一致性贡献：

1. 硬件 Virtio SR-IOV 数据面；
2. 多对多 VF–AFU 虚拟化；
3. 隔离的 AFU 上下文共享与硬件调度；
4. 与部分重配置协同的事务化生命周期。

## 2. 系统的中心抽象：Virtual FPGA Accelerator

### 2.1 定义

建议把虚拟机看到的设备统一称为 `Virtual FPGA Accelerator`，简称 VFA。一个 VFA 对应一个 SR-IOV VF，但 VF 只是 PCIe 承载形式；VFA 的完整语义是：

```text
VFA = stable Virtio endpoint
    + a tenant-scoped resource namespace
    + a dynamic set of AFU binding contexts
```

VFA 对虚拟机提供稳定的：

- Virtio PCI common configuration；
- feature negotiation；
- command/control/event virtqueues；
- 设备配置 generation；
- VF requester identity；
- MSI-X 能力上限；
- 用户态 runtime/API。

VFA 后端可以动态变化：

- AFU 数量；
- AFU 类型与版本；
- AFU 所在动态 slot；
- AFU 是否为该 VFA 独占；
- AFU 是否通过上下文与其他 VFA 共享；
- 分配的 DDR、队列和调度权重。

### 2.2 多对多绑定图

系统本质上不是一张简单映射表，而是一张动态二分图：

```text
VFA/VF side                         AFU side

VF-1 ── Binding-1 ───────────────→ AFU-A / context-0
  └─── Binding-2 ────────────────→ AFU-B / context-2

VF-2 ── Binding-3 ───────────────→ AFU-A / context-1
  └─── Binding-4 ────────────────→ AFU-C / context-0

VF-3 ── Binding-5 ───────────────→ AFU-A / context-3
```

两个方向分别代表：

- `VF → AFU*`：一个租户聚合多个功能，属于空间聚合；
- `AFU → VF*`：一个物理 AFU 服务多个租户，属于时间复用。

### 2.3 Binding Context

每条绑定边应对应一个独立的 `Binding Context`，而不是把同一状态复制到多张表：

```text
binding_id
vf_id
afu_slot_id
afu_context_id
afu_handle
queue_group_id
register_aperture
ddr_base / ddr_limit
event_vector_mapping
scheduler_weight / credit
generation
state
```

其中：

- `vf_id` 来自可信 PCIe VF 入口；
- `afu_slot_id` 由静态区物理连接附加；
- `afu_context_id` 是共享 AFU 内部的租户上下文；
- `afu_handle` 是 guest 可见但带 generation 的逻辑句柄；
- `generation` 防止解绑或重配置前的迟到请求进入新上下文。

## 3. 统一系统架构

```text
┌──────────────────────── Cloud Control Plane ────────────────────────┐
│ Accelerator Profile / Scheduler / Artifact Registry / Cyborg/DRA   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ resource request
┌──────────────────────── Compute Node ────────────────────────────────┐
│ ReVirt Node Agent                                                   │
│   discovery · placement · bitstream verification · rollback         │
│                               │ PF management                       │
│ ┌──────────────────────── FPGA Static Region ─────────────────────┐ │
│ │ SR-IOV PF                                                     │ │
│ │   PR controller · resource allocator · shadow/active tables   │ │
│ │                                                              │ │
│ │ SR-IOV VF / VFA                                              │ │
│ │   Virtio PCI config · feature negotiation                    │ │
│ │   hardware virtqueue engine · DMA · MSI-X                    │ │
│ │   capability discovery · command/event routing               │ │
│ │                                                              │ │
│ │ Binding Context table · memory protection · AFU scheduler    │ │
│ └───────────────────────────┬───────────────────────────────────┘ │
│                             │ trusted slot/context tags           │
│ ┌──────────────── FPGA Reconfigurable Region ────────────────────┐ │
│ │ AFU-A       AFU-B       AFU-C       ...       AFU-N           │ │
│ │ context[]   context[]   context[]             context[]       │ │
│ └───────────────────────────┬────────────────────────────────────┘ │
│                     tenant-partitioned device DDR/HBM              │
└─────────────────────────────┬──────────────────────────────────────┘
                              │ VF passthrough
┌────────────────────────── Virtual Machine ──────────────────────────┐
│ Application → Runtime/API → Virtio Accelerator Driver              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.1 控制面

控制面不进入正常任务数据路径，负责：

- 发现 FPGA、shell、slot、AFU、VF 和空闲 context；
- 校验 partial bitstream 的签名和兼容性；
- 匹配 BAR、MSI-X、DDR、队列、slot 和 context；
- 必要时执行部分重配置；
- 构建影子 Binding Context；
- 原子提交或失败回滚；
- 向 VFA 触发配置变化通知；
- 排空、解绑和资源回收。

### 3.2 数据面

数据面由硬件完成：

```text
guest descriptor
→ VF notification BAR
→ hardware virtqueue engine
→ Binding Context lookup
→ buffer/address validation
→ per-context request FIFO
→ AFU scheduler
→ AFU execution
→ completion router
→ used ring update
→ VF MSI-X
```

正常数据路径不经过 QEMU/vhost，PF 也不参与每个请求的转发。

### 3.3 资源命名空间

ReVirt 实际虚拟化四类资源：

1. **身份空间：** VF、binding、slot、AFU context 和 generation；
2. **执行空间：** virtqueue、request FIFO、AFU context 和调度 credit；
3. **内存空间：** guest IOVA 与租户设备 DDR/HBM 分区；
4. **通知空间：** used-ring completion、config change 和额外 MSI-X 事件。

论文应围绕这四类命名空间解释隔离，而不应只说“VF 天然隔离”。VF 只能提供 PCIe requester 粒度的入口隔离，AFU 后端共享仍需要第二级检查。

## 4. 六份材料如何形成一条完整生命周期

### 4.1 服务申请

用户提交 Accelerator Profile：

```text
AFU function/version
AFU count or function chain
DDR capacity
queue/interrupt requirement
sharing or exclusive mode
weight/SLO
isolation requirement
```

### 4.2 资源选择

资源选择不是只查找“是否存在该 AFU”，而是多维约束匹配：

```text
Σ register_size ≤ VF register aperture
queue requirement ≤ VF queue capability
interrupt requirement ≤ VF MSI-X capability
DDR requirement ≤ free tenant memory
AFU context requirement ≤ free contexts
AFU ABI compatible with Virtio driver/runtime
bitstream compatible with shell and PR slot
```

### 4.3 配置与绑定

```text
RESERVED
→ QUIESCING affected resources
→ ISOLATED
→ RECONFIGURING if necessary
→ VALIDATING
→ shadow Binding Context setup
→ atomic COMMIT
→ ACTIVE
```

发生错误时丢弃影子配置并回滚，不能让多张表分别处于半更新状态。

### 4.4 Guest 初始化

guest 驱动完成 Virtio feature negotiation 和 virtqueue 建立，通过配置页或 controlq 查询 AFU 能力。详细 AFU 列表不宜完全塞入可变长 device-specific configuration，建议配置空间只保留固定头部和 generation，详细描述通过 controlq 或共享能力页读取。

### 4.5 任务执行

命令至少应包含：

```text
opcode
afu_handle
context_id
request_id
input/output buffer handles
flags and timeout
```

一个 context 应支持多个 outstanding requests 或明确的队列深度，不能只依赖“一组寄存器 + 一个启动位”。

### 4.6 完成与异步事件

- 正常任务完成：更新 commandq used ring；
- 动态能力变化：Virtio config change；
- 无请求对应的 AFU 异常：额外 MSI-X 或 eventq；
- 不能同时为一次正常完成产生 used-ring notification 和第二个额外 IRQ。

### 4.7 解绑和回收

```text
ACTIVE
→ QUIESCING: reject new commands
→ DRAINING: finish/cancel queued and running requests
→ invalidate old generation
→ release context/DDR/queue/event mapping
→ FREE
```

必须处理 guest reset、VM 异常退出、AFU watchdog、部分重配置失败和迟到中断。

## 5. 真正值得写进论文的研究问题

### RQ1：统一接口是否可以接近原生直通性能

假设：硬件 Virtio VF 能显著降低软件 Virtio 的延迟和 CPU 开销，同时相对原生厂商 SR-IOV 接口只有有限额外代价。

### RQ2：多对多绑定是否提高资源利用率和请求接受率

假设：一个 VF 聚合多个 AFU、一个 AFU 服务多个 VF，可以减少空闲 AFU、空闲 VF 和资源碎片。

### RQ3：共享 AFU 是否仍能提供可预测的公平性和隔离

假设：独立 context FIFO、credit/weighted scheduler、DDR 配额和最大任务时间可以限制租户间干扰。

### RQ4：后端动态变化时，VF 能否保持稳定和正确

假设：quiesce、drain、generation、影子表和原子提交可以在不重新枚举 VF 的情况下改变 AFU 后端，并避免旧请求误路由。

### RQ5：一个 VF 聚合多个 AFU 是否能减少数据移动

假设：同一 VFA 内 AFU 共享租户 DDR，可以避免两个独立 VF 之间通过主机内存中转数据。

### RQ6：直接 AFU 异步 MSI-X 是否值得增加复杂度

假设：对于低频但延迟敏感的异步异常，直接 MSI-X 比 eventq 延迟低；对于正常任务完成，used ring 更合适。

云调度、Cyborg、DRA、SIOV 和跨节点池化不应成为本论文必须回答的核心 RQ，除非已经完成对应实现。

## 6. 最有说服力的论文贡献

建议将贡献控制在四项：

1. **硬件 Virtio SR-IOV 数据面。** 在 FPGA 静态区实现 Virtio PCI、virtqueue、DMA 和 MSI-X，使 guest 通过统一驱动直接访问硬件 VF，正常路径不经过软件设备模拟。
2. **多对多 VF–AFU 虚拟化。** 使用统一 Binding Context 支持一个 VFA 聚合多个 AFU，以及一个 AFU 通过独立上下文服务多个 VFA。
3. **隔离的非抢占式硬件共享。** 通过独立队列、可信身份、租户 DDR 检查和 credit 调度，在提高 AFU 利用率的同时控制公平性和性能干扰。
4. **事务化在线重配置。** 将 virtqueue 排空、上下文生命周期、影子配置和部分重配置结合，使 VFA 的后端功能可以在线变化并支持失败回滚。

如果当前只有 round-robin，没有 credit、队列配额或 DDR QoS，第三项应改成：

> 基于独立上下文的非抢占式 AFU 共享机制。

如果没有 AFU 间同步、DAG 或自动流水线，不应把“可组合”作为主要贡献；应使用“多 AFU 聚合”。

## 7. 与现有工作的关系

单独看，ReVirt 使用的许多技术并非首次出现：

- Virtio 已定义标准化设备状态、配置、通知和 virtqueue 框架。[OASIS Virtio 1.3](https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html)
- FPGA 空间/时间共享及保护已经是系统研究问题，AMORPHOS 也讨论了非抢占时间共享的局限。[AMORPHOS](https://www.usenix.org/sites/default/files/osdi18_full-proceedings.pdf)
- FlexCSV 已结合 SR-IOV、部分重配置和动态资源管理。[FlexCSV](https://www.usenix.org/system/files/atc21-kwon.pdf)
- vFPIO 已研究虚拟 I/O 抽象和多租户 I/O 事务调度。[vFPIO](https://www.usenix.org/conference/atc24/presentation/chen-jiyang)
- SVFF 已研究 Linux/QEMU/KVM 中的 FPGA VF 在线管理。[SVFF](https://arxiv.org/abs/2406.01225)
- SIOV 已提出 ADI、PASID、VDCM、直接路径和拦截路径分离。[Intel SIOV Specification](https://cdrdv2-public.intel.com/671403/intel-scalable-io-virtualization-technical-specification.pdf)

因此，论文不能把以下内容单独宣称为创新：

- 使用 SR-IOV；
- 使用 Virtio；
- 使用部分重配置；
- 使用映射表；
- 多租户共享 FPGA；
- 简单轮询调度。

潜在的差异化组合是：

> **硬件 Virtio SR-IOV 端点保持稳定，而其后端是一张可事务化更新的多对多 VF–AFU 图；同一机制同时支持 AFU 空间聚合、跨 VF 上下文共享、租户设备内存隔离以及部分重配置。**

这是否构成足够创新，最终取决于原型是否完整、对比是否充分、机制是否比“几张映射表加轮询”更严谨。

## 8. 当前方案必须正面解决的技术风险

### 8.1 Virtio-compatible 不等于已有标准设备类型

如果使用自定义 accelerator device ID、feature bits 和 device-specific ABI，应称为 `Virtio-compatible accelerator`。除非已经正式标准化，不应称为“标准 Virtio FPGA accelerator device”。

### 8.2 原始 BAR 暴露削弱统一接口

仅统一 `bar + offset` 的发现方式，应用仍需理解 AFU 私有寄存器。应定义稳定公共命令 ABI，把 BAR MMIO 保留为可选 fast path。

### 8.3 当前调度是非抢占式

运行至完成不等于固定时间片。长任务会造成队首阻塞。必须限制最大任务时长、支持 chunking，或明确独占模式。

### 8.4 权重最大优先不是加权公平

每次选择最大 weight 会饿死低权重 context。应采用 WRR、DRR 或按执行周期扣减的 credit scheduler。

### 8.5 共享 AFU 与部分重配置冲突

共享 AFU 更新前必须排空或迁移所有租户上下文。可比较等待排空、上下文迁移和版本并存三种策略。

### 8.6 DDR 隔离不能只检查起始地址

边界检查必须防整数溢出，并覆盖 `address + length` 的完整区间。AFU 身份必须由静态区附加，不能信任动态模块自报 ID。

### 8.7 中断扩展流程需要按 Linux 真实语义实现

Linux IRQ number 不保证连续；必须用 `pci_irq_vector()` 逐项获取。不能在 Virtio handler 仍注册时直接释放 vectors，也不能把 Linux IRQ number 当作 MSI-X 表内容。优先一次性预留全部 vectors，或使用受支持的动态 MSI-X API。

### 8.8 云基础设施不能只停留在愿景

如果论文把 PaaS、Cyborg 或 DRA 列为贡献，就必须实现 Accelerator Profile、资源发现、绑定失败处理和端到端交付流程。否则它们只应作为系统上下文。

## 9. 推荐的实验矩阵

| 研究问题 | 关键基线 | 工作负载 | 核心指标 |
|---|---|---|---|
| 硬件 Virtio 性能 | 软件 Virtio、原生厂商 SR-IOV | 空命令、4 KB～大块 DMA | 往返延迟、IOPS、吞吐、CPU、FPGA 开销 |
| 多对多利用率 | VF–AFU 独占、软件共享 | 突发压缩/加密请求 | AFU busy ratio、总吞吐、接受率 |
| 公平与隔离 | RR、WRR/DRR、credit | 长短任务混合、攻击性租户 | Jain 指数、P99、最大等待、DDR 干扰 |
| 动态重配置 | 全量配置、静态绑定 | AFU 替换和版本升级 | 停机时间、回滚、未参与 VF 性能 |
| 多 AFU 聚合 | 两个 VF 主机中转 | 压缩→加密、图像→编码 | PCIe 流量、端到端延迟、CPU、DDR 带宽 |
| 异步中断 | eventq、共享向量、独立 MSI-X | fault/watchdog/ECC 事件 | P99 事件延迟、CPU、中断率、资源开销 |
| 扩展性 | 不同 VF/AFU/context 数 | 合成请求 | LUT/FF/BRAM、频率、查表和调度周期 |
| 生命周期正确性 | 无 generation/无原子表消融 | reset、解绑、失败注入 | 错误路由、丢失、重复、恢复时间 |

### 9.1 必须至少实现的三类 AFU

- 短任务 AFU：压缩或加密，用于共享调度；
- 另一独立功能 AFU：用于一个 VF 多 AFU 聚合；
- 可替换版本或第三种 AFU：用于部分重配置。

### 9.2 最有价值的两个端到端场景

**场景 A：共享热门 AFU**

多个 VM 通过各自硬件 Virtio VF 使用同一个压缩 AFU，验证利用率、公平性、DDR 隔离和完成路由。

**场景 B：租户级多 AFU 数据链**

一个 VM 的 VFA 同时绑定压缩和加密 AFU，数据保留在同一租户 DDR 分区，比较两个独立 VF 经主机中转的基线。

这两个场景分别覆盖第四份专利和第二份专利，同时复用硬件 Virtio、动态绑定、中断和隔离机制。

## 10. 推荐的论文结构

1. **Introduction**：FPGA PaaS 需要稳定接口、动态功能和高利用率共享。
2. **Background and Motivation**：Virtio、SR-IOV、部分重配置；展示固定 VF–AFU 绑定的三个具体问题。
3. **Goals and Model**：VFA、信任模型、非抢占假设和系统边界。
4. **ReVirt Overview**：控制面、数据面和多对多绑定图。
5. **Hardware Virtio VF**：PCI capabilities、virtqueue、DMA、MSI-X 和公共 ABI。
6. **Spatio-Temporal AFU Sharing**：Binding Context、一个 VF 多 AFU、一个 AFU 多 VF、调度与隔离。
7. **Transactional Reconfiguration**：状态机、排空、影子表、generation 和回滚。
8. **Implementation**：FPGA、PCIe、Linux、VM、驱动、AFU、时钟和资源规模。
9. **Evaluation**：围绕 RQ1–RQ6 组织。
10. **Related Work**：FPGA 虚拟化、Virtio/SR-IOV、调度、PR 和云管理。
11. **Discussion**：功能链限制、抢占、SIOV、迁移和远程池化。
12. **Conclusion**。

## 11. 推荐题目与摘要主张

### 首选题目

> **ReVirt：面向 FPGA 云的硬件 Virtio SR-IOV 与 AFU 时空共享架构**

> **ReVirt: A Hardware Virtio SR-IOV Architecture for Spatio-Temporal AFU Sharing in FPGA Clouds**

如果没有严格意义上的 AFU 空间组合，可以把英文中的 `Spatio-Temporal AFU Sharing` 改为：

> **Dynamic Multi-AFU Aggregation and Sharing**

### 摘要逻辑骨架

```text
问题：现有 FPGA 云接口与设备绑定，SR-IOV VF 后端固定，独占 AFU 利用率低。

方法：ReVirt 在 FPGA 中实现硬件 Virtio SR-IOV VF，并以 Binding Context
建立多对多 VF–AFU 映射，使一个 VF 聚合多个 AFU、一个 AFU 服务多个 VF。
PF 使用事务化协议管理部分重配置，VF 数据面保持直通。

机制：硬件 virtqueue、租户 DDR 隔离、非抢占式 context 调度、generation、
影子表提交和异步事件路由。

结果：填写实测数据，分别说明相对软件 Virtio、原生 SR-IOV、独占 AFU
和全量重配置的性能、利用率、公平性及更新时间。

结论：ReVirt 在保持稳定统一接口的同时，实现可重构 FPGA 功能的高效共享。
```

摘要中不要提前写“低开销”“高公平”“无影响”等结论，直到实验得到相应数据。

## 12. 一篇还是两篇论文

### 论文一：ReVirt 设备架构

当前应优先完成，范围是：

- 硬件 Virtio SR-IOV VF；
- 多对多 VF–AFU Binding Context；
- 非抢占上下文调度和隔离；
- 多 AFU 聚合与共享 DDR；
- 部分重配置和一致生命周期；
- 节点内实机评估。

### 论文二：FPGA PaaS 基础设施

在有完整云原型后再写，范围是：

- AFU artifact registry 和供应链；
- Accelerator Profile；
- Cyborg或 Kubernetes DRA 集成；
- 多卡与跨节点放置；
- 重配置感知调度；
- 请求接受率、负载均衡和容错；
- 跨代兼容与迁移。

这样拆分能避免当前论文同时承担硬件接口、设备虚拟化、调度、重配置、云编排、池化和迁移七条主线。

## 13. 下一步工作优先级

### P0：决定论文能否成立

1. 画出统一架构图和 VF–AFU 多对多绑定图；
2. 固化 VFA 公共 ABI、Binding Context 和状态机；
3. 明确硬件 Virtio VF 实际实现模块；
4. 修正中断和 Linux IRQ 生命周期问题；
5. 实现至少两个 AFU、两个 VF 和 AFU 跨 VF 共享；
6. 建立软件 Virtio、原生 SR-IOV 和独占 AFU 基线。

### P1：形成主要实验结论

1. 硬件 Virtio 性能与资源开销；
2. AFU 共享的利用率、公平性和 P99 延迟；
3. 同一 VF 多 AFU 的设备内零拷贝收益；
4. 动态更新的端到端停机时间与未参与 VF 影响；
5. reset、解绑、异常和回滚正确性。

### P2：增强论文完整度

1. credit/DRR 调度；
2. 队列和 DDR QoS；
3. bitstream manifest 与签名；
4. 节点 agent 或 Cyborg 的最小集成；
5. trace-driven 请求接受率实验。

### P3：后续论文或未来工作

1. 抢占和 AFU 状态迁移；
2. SIOV/PASID/VDCM；
3. VM 热迁移；
4. 多卡及跨节点池化；
5. CXL/RDMA 远程 AFU；
6. 完整 DAG 功能链运行时。

## 14. 最终判断

这组材料的学术价值不在于某一张表、某一种 BAR 布局或某一个中断技巧，而在于形成了一个清晰的系统闭环：

```text
统一 Virtio 接口
→ 硬件 SR-IOV 直通
→ 多对多 VF–AFU 虚拟化
→ AFU 时空共享与租户隔离
→ 部分重配置和事务化生命周期
→ FPGA PaaS 节点内服务交付
```

如果原型能够完整跑通这一闭环，并用实验证明性能、利用率、公平性和更新连续性，ReVirt 可以成为一篇结构完整的 FPGA/系统方向论文。反之，如果只实现若干表项配置和简单轮询，论文容易被评价为工程集成。接下来的重点不是继续增加概念，而是固化抽象、补齐一致性机制、建立强基线并获得可信数据。
