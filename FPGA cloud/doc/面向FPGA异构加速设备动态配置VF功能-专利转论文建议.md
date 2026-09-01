# 面向 FPGA 云的硬件 Virtio、动态重构与 AFU 时空共享——专利转论文建议

> **实现路线确认：** 当前原型采用“硬件实现 Virtio 的 SR-IOV VF”架构。虚拟机通过直通 VF 直接访问 FPGA 静态区实现的 Virtio PCI 接口，QEMU/vhost 不进入正常数据路径；一个 VF 可以聚合多个 AFU，一个 AFU 也可以通过独立上下文被多个 VF 分时共享。后续论文定位和实验建议均以此实现路线为准。

本文档综合参考以下五份技术交底书：

1. 《面向 FPGA 异构加速设备动态配置虚拟设备 VF 功能的方法》；
2. 《一种功能可组合的 FPGA 异构加速器架构》；
3. 《基于 Virtio 面向异构加速设备的统一虚拟化接口设计》；
4. 《一种基于虚拟功能共享的 SR-IOV FPGA 加速器利用率优化方法》；
5. 《一种面向虚拟化加速器设备的中断扩展配置方法及系统》。

同时参考整体基础设施思考材料《面向云场景的 FPGA 异构计算基础设施》。该 PPT 用于确定云服务定位、参与角色、资源申请流程和长期演进路线，不等同于当前原型已经完成的功能范围。

五份技术交底书具备联合转化为论文的基础，但目前更像“可实施的工程方案”，还缺少论文所要求的研究问题、组合执行语义、调度模型、形式化设计、对比实验和量化结论。第五份专利适合作为硬件 Virtio VF 中断子系统的关键工程实现，不建议单独拔高为整篇论文的主要架构贡献。

综合前四份架构性专利，并将第五份作为中断子系统实现支撑后，建议把论文主线凝练为：

> 将 SR-IOV VF 抽象为具有稳定 Virtio ABI 的租户级虚拟加速器容器，通过多对多 VF–AFU 绑定图同时支持一个 VF 对多个 AFU 的空间聚合和一个 AFU 对多个 VF 的时间复用，并结合动态部分重配置、租户级资源隔离及硬件任务调度，实现 FPGA 加速功能的统一访问、在线装载、高利用率共享和零拷贝协同。

## 一、论文最适合强调的创新点

不要把“VF 与算法映射表”单独作为核心创新，因为仅有映射表容易被认为是常规工程设计。建议组织成三个相互支撑的贡献。

### 1. 稳定 VF 与可变算法功能的解耦架构

核心矛盾是：

- SR-IOV 的 VF 配置空间、BAR 和 MSI-X 能力通常在设备枚举时确定；
- FPGA 算法模块却需要运行时替换；
- 如果把算法接口直接固化到 VF 中，算法变化会导致设备重新枚举甚至重启服务器。

本方案通过静态区提供稳定 VF 接口，通过动态区承载算法模块，再由映射表完成绑定。这是论文最重要的体系结构贡献。

建议给这个架构起一个简洁名称，例如：

- DVFA：Dynamic VF–Accelerator Binding Architecture
- ReVF：Reconfigurable Virtual Function Architecture
- VFA-Bind：Runtime VF-to-Accelerator Binding Framework

### 2. 面向异构算法的统一 VF 接口

将接口抽象为：

- 通用控制与 DMA BAR；
- 算法私有寄存器 BAR；
- MSI/MSI-X 中断接口；
- 统一的算法身份与能力描述接口。

论文应强调：变化的是 VF 后面的算法实现，而不是虚拟机看到的 PCIe 设备接口。

这一点需要比交底书进一步明确：

- BAR 大小必须预先固定，动态配置后不能随算法任意改变；
- VF 的 MSI-X 向量数在枚举时基本固定，算法只能选择能力足够的 VF；
- 各算法应使用同一个驱动 ABI，或者提供版本兼容机制；
- 算法寄存器空间不能直接暴露未经检查的物理地址。

### 3. 原子化的在线配置、发现与绑定协议

目前交底书中的流程是“读取比特流信息—选择 VF—部分重配置—扫描算法 ID—更新映射表”。论文需要把它提升为完整的运行时协议：

1. 暂停目标 VF 接收新任务；
2. 排空在途 DMA 和中断；
3. 将映射条目标记为无效；
4. 隔离可重构区域；
5. 加载部分比特流；
6. 校验算法身份、接口版本和资源需求；
7. 选择满足 BAR、中断和队列需求的 VF；
8. 原子更新映射关系；
9. 恢复 VF 服务；
10. 失败时回滚到旧映射或安全状态。

这个状态机比“更新映射表”更有论文价值，也能回答审稿人必然会问的并发、安全和故障恢复问题。

## 二、现有方案中需要补强的技术问题

### 1. 算法 ID 不宜直接承担全部路由功能

交底书中寄存器请求通过“VF 号 → 片选 ID”路由，而中断通过“算法 ID → VF”反向查询。这里存在几个问题：

- 算法 ID 是否全局唯一；
- 同一种算法能否部署多个实例；
- 算法伪造或错误上报 ID 如何处理；
- 反向查询是遍历还是 CAM，扩展到几十个 VF 后代价如何。

建议把运行时唯一的 `slot_id/instance_id` 作为硬件路由键，把 `algorithm_id` 作为算法类型信息：

```text
VF ID → slot ID → algorithm instance
slot ID → VF ID → MSI-X vector
```

映射项可以扩展为：

```text
<VF_ID, slot_ID, algorithm_ID, ABI_version,
 interrupt_count, queue_count, generation, state>
```

其中 `generation` 可防止重配置前的延迟请求误入新算法实例。

### 2. 补充 DMA 隔离与地址安全

现文档主要描述寄存器和中断映射，但论文必须解释 DMA：

- 每个 VF 是否有独立 DMA 队列；
- Requester ID/PASID/IOMMU 如何保证地址隔离；
- 算法模块能否自行产生 DMA 地址；
- 一个恶意或故障算法是否可能访问其他 VF 的队列或数据；
- 重配置前如何确认 DMA 已排空。

否则“多租户隔离”这一核心论点不完整。

### 3. 增加重配置隔离和异常处理

需要说明动态区重配置过程中：

- 静态区如何阻断 AXI/寄存器访问；
- VF 在此期间返回重试、错误还是忙状态；
- 部分比特流校验失败如何处理；
- 算法迟迟不停止时是否超时；
- 重配置是否会干扰其他 VF；
- PF 是否是唯一有权修改映射表的实体。

建议加入 decoupler/isolation 模块，以及以下状态机：

```text
ACTIVE → QUIESCING → ISOLATED → CONFIGURING
       → VALIDATING → ACTIVE/ERROR
```

### 4. 明确“动态”的边界

论文必须区分三种不同操作：

- 动态创建或删除 VF；
- 已有 VF 在不同虚拟机之间迁移；
- VF 不变，仅动态更换其后端 FPGA 算法。

这项专利主要解决第三种。论文中不要泛化成“动态配置 VF 本身”，否则容易产生概念歧义。

## 三、实验是论文能否成立的关键

至少设置以下基线：

- 全 FPGA 重配置并重启/重新枚举；
- 全 FPGA 重配置但通过设备复位和重新扫描恢复；
- VF 与算法静态绑定；
- 本文提出的部分重配置加动态绑定方案。

建议回答五组研究问题。

### RQ1：动态更新有多快

测量：

- 部分比特流加载时间；
- 算法发现时间；
- VF 选择及映射更新时间；
- 从停止接收请求到重新可用的总时间；
- 与全量重配置相比的加速倍数。

不要只测纯 bitstream 下载时间，应报告端到端服务不可用时间。

### RQ2：是否真正不影响其他租户

在一个 VF 动态换算法时，让其他 VF 持续运行 DMA 工作负载，记录：

- 吞吐量下降比例；
- 平均及 P95/P99 延迟；
- 最大服务中断时间；
- 丢失或重复请求数；
- 中断抖动。

“不需要重启”只是定性描述；“未参与重配置的 VF 吞吐下降不超过多少、P99 延迟增加多少”才是论文证据。

### RQ3：数据通路开销多大

比较启用和绕过映射/仲裁逻辑时的：

- DMA 吞吐率；
- 小包及大块传输延迟；
- MMIO 访问延迟；
- 中断触发到主机响应的时间；
- FPGA LUT、FF、BRAM、URAM 消耗；
- 时钟频率和功耗。

### RQ4：能否扩展

改变以下参数：

- VF 数量；
- 动态分区数量；
- 并发算法实例数；
- 每个 VF 的中断和 DMA 队列数；
- 映射表规模。

观察映射查找延迟、资源开销和最大吞吐量。尤其需要说明线性扫描是否会成为瓶颈。

### RQ5：异常情况下是否安全

至少测试：

- 加载非法或损坏的部分比特流；
- 算法 ID/ABI 版本不匹配；
- 没有满足中断数量要求的 VF；
- 重配置时仍有 DMA 请求；
- 重配置失败后的回滚；
- 旧中断或旧请求延迟到达。

## 四、建议的论文结构

### 推荐题目

中文：

> 面向多租户 FPGA 加速器的 SR-IOV 虚拟功能与可重构算法动态绑定机制

英文：

> *Runtime Binding of SR-IOV Virtual Functions to Reconfigurable Accelerators on Multi-Tenant FPGAs*

### 正文结构

1. **引言**：云端 FPGA 的共享需求、静态 VF 绑定的限制、研究问题及三项贡献。
2. **背景与动机**：SR-IOV、部分重配置、传统静态架构，以及一个具体的失效场景。
3. **系统目标与威胁模型**：接口稳定、业务连续、低开销、租户隔离；明确可信 PF 和不可信 VF/算法边界。
4. **总体架构**：静态区、动态区、DMA、寄存器路由、中断路由、映射管理器和隔离模块。
5. **动态绑定机制**：能力描述、VF 选择、算法发现、映射更新、状态机、异常回滚。
6. **实现**：FPGA 型号、PCIe 代际、开发工具、虚拟化平台、驱动和比特流规模。
7. **实验评估**：围绕前述五个研究问题组织，不要按“测试项目列表”堆砌。
8. **相关工作**：FPGA 虚拟化、SR-IOV、部分重配置、运行时资源管理。
9. **局限性与未来工作**：状态迁移、异构动态区大小、bitstream 安全、多实例 QoS 等。
10. **结论**。

## 五、与已有工作的区别要谨慎定位

近年来已有工作覆盖了部分相邻方向。例如，FlexCSV 同时使用 SR-IOV、算子重命名和部分重配置，说明“SR-IOV + 部分重配置”本身不足以单独宣称创新；SVFF 则研究了 Linux/QEMU/KVM 环境中 VF 的在线管理和透明重配置。

- [FlexCSV: A Flexible Computational Storage and Virtualization System](https://www.usenix.org/system/files/atc21-kwon.pdf)
- [SVFF: An Automated Framework for SR-IOV Virtual Function Management in FPGA Accelerated Virtualized Environments](https://arxiv.org/abs/2406.01225)

因此建议把差异集中在：

- PCIe VF 接口保持稳定时，后端异构算法实例如何改变；
- 寄存器、DMA 和 MSI-X 三条路径如何一致地完成动态重绑定；
- 如何原子切换且不影响未参与更新的 VF；
- 如何处理算法能力不匹配、在途请求和失败回滚。

相关工作中也应讨论软件可配置虚拟功能、FPGA 多租户隔离和动态部分重配置，而不能只介绍 SR-IOV 标准。早期研究已经指出固定设备配置与可编程设备灵活性之间存在矛盾，可将 [Standardized but Flexible I/O for Self-Virtualizing Devices](https://www.usenix.org/legacy/event/wiov08/tech/full_papers/levasseur/levasseur_html/) 作为问题背景；FCSV-Engine 则可作为较强的系统级对比对象。

## 六、专利与论文发表顺序

首先确认这份技术方案是否已经正式提交专利申请，以及论文中是否会增加尚未申请的新方案。国家知识产权局明确说明：提出专利申请后再公开论文通常不影响该申请的新颖性；如果先公开论文，则可能破坏后续申请的新颖性。

- [国家知识产权局：申请专利以后是否可以发表相关论文](https://www.cnipa.gov.cn/jact/front/mailpubdetail.do?sysid=6&transactId=278835)
- [中华人民共和国专利法（2020 年修正）](https://www.cnipa.gov.cn/art/2020/11/23/art_2197_155169.html)

如果论文新增了原子切换、DMA 隔离、回滚机制或能力匹配算法等内容，建议先判断这些新增内容是否需要补充申请或另行申请，再投稿。此处最好让专利代理人结合具体申请状态确认。

## 七、总体判断

这个选题可以写，但需要把论文从“架构描述型”升级为“系统机制 + 原型实现 + 定量评估型”。最值得投入的两个部分，是完整的在线切换状态机，以及证明其他 VF 业务不中断的端到端实验。

## 八、第二份专利带来的论文增强点

### 1. 两份专利的关系

两份专利不是简单并列关系，而是可以组成一个递进的研究体系：

| 层次 | 第一份专利 | 第二份专利 | 合并后的论文问题 |
|---|---|---|---|
| VF 与 AFU 关系 | 一个 VF 动态绑定一个算法模块 | 一个 VF 动态绑定一个或多个 AFU | VF 如何成为可动态组装的虚拟加速器容器 |
| 功能更新 | 通过部分重配置替换动态区算法 | 从已有 AFU 中选择并组合 | AFU 的部署与组合如何统一管理 |
| 寄存器 | VF BAR 路由到单个算法 | 在 VF BAR 内为多个 AFU分配子区间 | 如何虚拟化并动态分配寄存器命名空间 |
| 中断 | 算法中断映射到 VF | 多个 AFU 共享 VF 的 MSI-X 向量空间 | 如何分配、路由并回收中断向量 |
| 数据 | 主要描述主机与设备间 DMA | 同一 VF 的多个 AFU 共享租户级 DDR 区间 | 如何实现 AFU 间零拷贝协同并保持隔离 |
| 资源管理 | 依据中断能力选择 VF | 同时匹配 BAR、MSI-X、DDR 和 AFU | 如何完成多维资源匹配与动态装配 |

因此，合并后的论文不应只描述“动态配置 VF 功能”，而应回答一个更完整的问题：

> 如何在一个稳定的 SR-IOV VF 接口后面，安全地装配一组可重构 AFU，使它们共享该 VF 的寄存器、中断和设备内存资源，并在不重新枚举 PCIe 设备的情况下改变虚拟加速器的功能构成？

### 2. 更合适的论文定位

第二份专利使论文可以从“动态映射机制”提升为“可组合虚拟 FPGA 加速器架构”。建议将系统抽象成三层：

```text
虚拟机/应用
    │
稳定的 SR-IOV VF（租户级虚拟加速器容器）
    │
虚拟资源层：BAR 子空间 + MSI-X 子空间 + DDR 地址空间
    │
AFU 组合层：AFU-1 + AFU-2 + ... + AFU-n
    │
动态 FPGA 区域与设备 DDR
```

这个抽象比“四张表支持绑定”的表述更适合论文。四张表是实现手段，虚拟加速器容器和三类资源命名空间才是核心设计。

## 九、必须区分“功能聚合”与“功能组合”

第二份专利目前已经充分描述了以下能力：

- 一个 VF 可以发现并访问多个 AFU；
- 多个 AFU 可以共享同一租户的设备 DDR 区间；
- 多个 AFU 的寄存器和中断被映射到同一个 VF；
- 避免数据经主机内存从一个 VF 搬运到另一个 VF。

这些内容能够证明“多 AFU 聚合”和“共享内存下的潜在零拷贝协同”，但还不能自动证明“功能组合”。如果论文题目使用 *composable* 或“可组合”，审稿人通常会继续追问：

- AFU A 的输出怎样成为 AFU B 的输入；
- 谁定义 A→B 的执行顺序和依赖关系；
- AFU 间通过 DDR、片上流接口还是消息队列传递数据；
- 如何通知下游 AFU 数据已经就绪；
- 多个 AFU 并发访问共享 DDR 时如何仲裁；
- 组合中的某个 AFU 失败时，整个任务如何恢复；
- 是否能把编码→加密、压缩→加密等流程作为一个端到端任务提交。

建议从以下两种论文定位中选择一种。

### 定位 A：多 AFU 聚合型虚拟加速器

如果当前硬件只支持多个 AFU 共享 VF 资源和 DDR，则使用更准确的表述：

> 一个面向多租户 FPGA 的多 AFU 聚合与动态绑定架构。

这种定位不要求实现完整任务图，但必须证明共享 DDR 能减少 VF 之间的主机中转和数据复制。

### 定位 B：可组合 AFU 流水线

如果能够继续扩展原型，则增加：

- 由 AFU 节点和数据依赖边组成的 DAG 描述；
- 共享 DDR 缓冲区句柄或片上流通道；
- AFU 间完成事件、doorbell 或队列；
- 组合任务的启动、同步、异常传播和结束协议；
- 从用户提交功能链到硬件装配的运行时系统。

此时才适合把“可组合”作为论文的主要创新声明。

## 十、第二份专利中需要修正或深化的设计

### 1. 四张表存在重复状态与一致性风险

当前包括：

- VF 状态信息表；
- AFU 状态信息表；
- VF 与 AFU 映射表；
- AFU 配置表。

同一个绑定状态分散在多张表中。配置或解绑过程中一旦失败，可能出现 VF 显示已绑定、AFU 却显示未绑定，或者正反向映射不一致。

建议采用以下机制：

- PF 是唯一写入者，VF 只能读取自己的只读能力页；
- 使用活动表和影子表双缓冲；
- 先在影子表中完成全部资源分配和合法性校验；
- 通过一次原子 `commit` 切换表版本；
- 每个表项带 `generation` 和 `state`；
- 失败时丢弃影子表，不改变当前活动配置。

这可以成为论文中的“事务化绑定协议”。

### 2. VF 资源选择是多维装箱问题

第二份专利中的 VF 选择同时受到以下资源约束：

```text
Σ AFU.reg_size ≤ VF.BAR_aperture
Σ AFU.msix_count ≤ VF.msix_capacity
requested_DDR ≤ available_DDR_partition
AFU_interface_version ∈ VF.supported_versions
```

如果还要同时选择 FPGA 卡、动态分区和 AFU 实例，该问题接近多维装箱或资源匹配问题。论文可以设计一个简单但可复现的算法，例如：

1. 过滤不包含所需 AFU 的 FPGA；
2. 过滤没有足够连续 BAR/MSI-X/DDR 空间的 VF；
3. 使用 best-fit 选择资源剩余量最小的可行 VF；
4. 对寄存器和 MSI-X 子空间分别执行对齐分配；
5. 生成影子映射表并原子提交。

实验中应与 first-fit、随机选择和静态分配比较资源利用率、请求接受率及分配延迟。

### 3. BAR 能力发现需要版本与并发协议

第二份专利让虚拟机通过只读 BAR 读取 VF 所包含的 AFU、寄存器区间和中断信息，这一设计很适合作为论文中的自描述接口，但还需增加：

- 固定的 magic、ABI version 和总长度；
- AFU 数量上限及表项长度；
- `generation` 字段；
- 完整性校验或读取前后版本校验；
- 更新期间的 `BUSY/INVALID` 状态；
- 驱动在配置变化后的通知与重新发现机制。

否则驱动可能在 PF 更新表项的中间状态读取到不一致内容。

### 4. 寄存器区间和 MSI-X 分配需要处理碎片

建议明确：

- `reg_start` 按页或固定粒度对齐；
- 区间采用半开形式 `[reg_start, reg_end)`，减少边界歧义；
- 检查任意两个 AFU 的寄存器区间不重叠；
- MSI-X 向量区间也采用 `[msix_base, msix_base + count)`；
- 解绑后的空洞如何合并；
- 是否要求连续向量，如果要求，碎片是否会导致明明总量足够却无法分配。

### 5. 不能信任 AFU 自报的索引号

专利描述中断和 DDR 请求携带 `afu_index`。如果该字段由动态区 AFU 自行产生，一个故障或恶意 AFU 可以伪造其他 AFU 的身份。

更安全的实现是：

- 在静态区的每个动态槽位入口硬连线附加 `slot_id`；
- 动态 AFU 只能提交地址、长度、中断局部编号等请求内容；
- 静态区根据物理入口产生可信 `slot_id`；
- 再用 `slot_id` 查询 AFU 配置上下文。

这样多租户隔离不依赖动态模块诚实上报身份。

### 6. DDR 边界检查需要防止整数溢出

不能只检查起始地址，应使用等价于以下条件的安全逻辑：

```text
length > 0
address >= ddr_start
address - ddr_start <= ddr_length
length <= ddr_length - (address - ddr_start)
```

应避免直接计算 `address + length` 后比较，因为固定宽度硬件加法可能溢出。还要明确：

- 主机 DMA 与 AFU 访问使用同一租户地址空间还是不同地址表示；
- DDR 分区的分配粒度与对齐要求；
- 多 AFU 并发访问的带宽仲裁和 QoS；
- 数据生产者与消费者之间的内存顺序和完成通知；
- 解绑前如何保证没有未完成的 DDR 请求。

## 十一、综合两份专利后的推荐贡献表述

论文引言中可将贡献写成以下三项，但最终必须有实现和实验支撑：

1. **可组合虚拟加速器抽象。** 将稳定的 SR-IOV VF 抽象为租户级容器，使其能够在不改变 PCIe 枚举状态的情况下绑定一个或多个可重构 AFU。
2. **统一的三类资源虚拟化机制。** 在 VF 内动态划分 BAR 寄存器子空间、MSI-X 向量子空间和设备 DDR 地址空间，并通过可信静态区完成双向路由与访问检查。
3. **事务化在线装配协议。** 通过静默、隔离、能力校验、影子表配置、原子提交和失败回滚，实现 AFU 组合的在线变更，同时维持其他租户的服务连续性。

如果没有实现 DAG、同步或自动数据流编排，第一项中的“可组合”建议改成“多 AFU 聚合”，避免贡献表述超过实际系统能力。

## 十二、综合实验方案

第二份专利加入后，除前述 RQ1—RQ5 外，建议增加以下研究问题。

### RQ6：多 AFU 共享 DDR 是否真正减少数据搬移

选择至少两个功能链：

- 压缩 → 加密；
- 视频编码 → 加密，或图像变换 → 压缩。

比较：

1. 两个 AFU 分别绑定两个 VF，数据经设备 DDR→主机内存→设备 DDR 中转；
2. 两个 AFU 绑定同一个 VF，共享设备 DDR，但由软件顺序启动；
3. 如果实现流水线，再比较自动组合执行。

测量：

- 端到端吞吐率和延迟；
- PCIe 上下行流量；
- 主机 CPU 占用率；
- 主机内存带宽；
- 设备 DDR 带宽；
- 每处理 1 GB 数据的能耗。

这是第二份专利最容易形成有说服力结果的实验。

### RQ7：资源组合的灵活性如何

构造不同大小、不同中断需求的 AFU 集合，比较：

- 一个 VF 只能绑定一个 AFU；
- 一个 VF 可以绑定多个 AFU；
- 不同资源选择算法。

测量：

- VF 消耗数量；
- BAR、MSI-X 和 DDR 利用率；
- 可接受的租户请求比例；
- 资源碎片率；
- 绑定和解绑时间。

### RQ8：事务化更新能否保证一致性

在持续发起 MMIO、DMA 和中断请求时，反复执行绑定、重配置和解绑，验证：

- 不出现跨 VF 地址访问；
- 不出现错误 AFU 中断路由；
- 不出现半更新表项；
- 旧 generation 请求不会进入新 AFU；
- 配置失败后原有 VF 能继续工作。

## 十三、更新后的题目建议

如果只实现多 AFU 聚合：

> 面向多租户 FPGA 云的多加速单元聚合与动态 VF 绑定架构

> *Dynamic Multi-AFU Aggregation behind SR-IOV Virtual Functions for Multi-Tenant FPGA Clouds*

如果实现组合执行和零拷贝流水线：

> 面向 FPGA 云的可动态重构与组合的 SR-IOV 虚拟加速器架构

> *A Dynamically Reconfigurable and Composable SR-IOV Accelerator Architecture for FPGA Clouds*

仅从前两份技术交底书的完成度看，建议先采用“多 AFU 聚合”作为稳妥定位；如果能够补充任务链、同步协议和组合运行时，再升级为“功能可组合”定位。第三份专利加入后的整体选题取舍见第十九至二十一节。

## 十四、第三份专利在整体架构中的位置

第三份专利补充的是前两份专利尚未充分覆盖的“北向软件接口层”。三份专利可以组成以下完整技术栈：

```text
虚拟机应用与统一用户态 API
              │
通用 Virtio 加速器前端驱动
              │
Virtio PCI 传输、特性协商和 virtqueue
              │
SR-IOV VF：租户级虚拟加速器容器
              │
BAR / MSI-X / DDR 三类虚拟资源命名空间
              │
VF-AFU 事务化绑定与双向路由
              │
一个或多个动态 AFU + 共享租户 DDR
              │
FPGA 部分重配置区域

PF 管理平面：发现、选择、装载、绑定、提交、回滚
```

三份专利分别回答：

| 专利 | 主要问题 | 在统一论文中的角色 |
|---|---|---|
| 动态配置 VF 功能 | VF 如何与变化的 FPGA 算法解耦 | 在线重配置与动态绑定 |
| 功能可组合架构 | 一个 VF 如何包含多个 AFU | 多 AFU 聚合和租户级资源共享 |
| Virtio 统一接口 | 应用如何摆脱具体设备接口 | 稳定 ABI、统一驱动和跨后端兼容 |

第三份专利使论文有机会形成“接口—虚拟化—重配置”三层联合设计，而不再只是 FPGA 内部的硬件路由机制。

## 十五、必须明确 Virtio 与 SR-IOV 的关系

Virtio 和 SR-IOV 解决的问题不同：

- Virtio 定义虚拟设备的标准化软件接口和队列语义；
- SR-IOV 让一个 PCIe 设备提供多个可直接分配的硬件 VF；
- 前者不是后者的自然上层，也不能在论文中不加说明地混为一种机制。

存在三种实现路线。

### 路线 A：软件中介 Virtio

```text
Guest Virtio driver → QEMU/vhost backend → host PF/VF driver → FPGA
```

优点是实现容易、适配传统 Virtio 软件栈；缺点是数据路径存在主机软件中介，无法充分体现 SR-IOV 直通价值。

### 路线 B：原生 SR-IOV 厂商接口

```text
Guest vendor driver → passthrough VF → FPGA
```

优点是性能高；缺点是应用和驱动仍与具体设备接口绑定，无法体现第三份专利的统一接口价值。

### 路线 C：硬件实现 Virtio 的 SR-IOV VF

```text
Guest Virtio accelerator driver → passthrough VF
                              → hardware Virtio queues → AFUs
```

**当前实现已确认采用此路线。** 每个 VF 同时是：

- 一个可被虚拟机直通访问的 SR-IOV VF；
- 一个符合 Virtio PCI 传输模型的虚拟加速器设备；
- 一个可动态绑定多个 AFU 的租户级硬件容器。

论文应明确说明：Virtio PCI common configuration、通知结构、设备配置空间、virtqueue 处理及 MSI-X 通知均由 FPGA 静态区硬件实现；虚拟机获得 VF 后直接建立 virtqueue，正常命令和数据传输不经过 QEMU/vhost 软件后端。PF 仅承担管理控制面功能，不进入正常 VF 数据路径。

## 十六、“复用 Virtio 驱动”需要准确表述

第三份专利中“可复用现有 Virtio 驱动框架，避免额外开发”的表述需要收敛。Virtio 提供的是：

- 设备发现和状态机；
- 特性协商；
- virtqueue 描述符和通知机制；
- PCI/MMIO 等传输层；
- 配置变化与复位框架。

但如果定义一种新的异构加速器设备类型，仍然需要开发：

- 设备类型专用的前端驱动；
- FPGA 或软件后端；
- AFU 发现、内存管理、命令提交和事件处理逻辑；
- 用户态 API 或运行时库。

因此论文宜表述为：

> 复用 Virtio 的传输、队列、通知和特性协商框架，只需维护一个设备类型专用驱动，即可支持具有不同 AFU 实现的后端。

而不宜声称“无需开发驱动”。截至目前的 Virtio 标准已经定义多种设备类型和通用机制，但没有可直接等同于本专利设计的通用 FPGA/异构 AFU 设备类型。因此，如果使用私有 device ID 或 vendor-specific 机制，论文应称为“Virtio-compatible prototype”，不能直接称为已标准化接口。

参考：

- [OASIS Virtual I/O Device (VIRTIO) Version 1.3](https://docs.oasis-open.org/virtio/virtio/v1.3/virtio-v1.3.html)
- [OASIS Virtual I/O Device (VIRTIO) Version 1.4](https://docs.oasis-open.org/virtio/virtio/v1.4/)

## 十七、第三份专利中的接口设计需要深化

### 1. 不宜把可变长 AFU 清单全部放入设备配置空间

Virtio 设备配置空间更适合较少变化、主要在初始化阶段读取的参数。AFU 列表可能较长，并且会随部分重配置和动态绑定改变。如果把整个列表直接放入配置空间，会产生：

- 配置空间容量和 AFU 数量上限问题；
- 多字段读取期间发生配置变化的一致性问题；
- 增加新字段时的 ABI 兼容问题；
- 动态添加 AFU 后 virtqueue 数量无法同步变化的问题。

建议使用两级发现接口：

```text
Device-specific configuration：
    ABI version
    feature bits
    max_afu
    active_afu
    config_generation
    control queue index

controlq：
    QUERY_AFU_LIST
    QUERY_AFU_CAPABILITY
    CREATE_CONTEXT
    DESTROY_CONTEXT
    RESET_AFU
```

详细 AFU 描述通过控制队列或共享能力页获取。读取前后检查 `config_generation`，确保没有读到跨版本数据。Virtio 规范本身已经提供配置 generation 和配置变化通知，可直接利用，而无需另造一套不兼容机制。

### 2. 动态 AFU 不宜对应动态数量的 virtqueue

专利为每个 AFU 定义一对 `requestq + eventq`。如果 VF 动态绑定的 AFU 数量改变，virtqueue 数量和索引也随之变化，但 Virtio 队列通常在设备初始化阶段发现并配置，不能假设运行期间可任意增加或删除。

建议从两种方案中选择：

**固定队列池：**

- VF 预先暴露固定上限的队列对；
- AFU 绑定后从池中分配队列；
- 适合追求 AFU 间性能隔离，但空闲队列会消耗资源。

**共享队列：**

- 一个或多个共享 `commandq`；
- 一个共享 `eventq`；
- 每条命令和事件携带由静态区分配的 `afu_handle/context_id`；
- 可通过多队列或 queue group 扩展并发。

对于 FPGA 静态区资源有限的场景，推荐共享队列方案。它还能减少 virtqueue 和 MSI-X 向量消耗。

### 3. 需要定义真正统一的命令语义

第三份专利仍通过 `bar + offset` 暴露 AFU 私有寄存器。这样虽然统一了“发现形式”，应用仍必须理解每个 AFU 的私有寄存器布局，并未完全解除应用与设备的绑定。

可以将接口分成两层：

- **稳定公共 ABI：** 枚举、内存分配、数据传输、命令提交、等待事件、取消和复位；
- **AFU 专用 ABI：** 算法参数和操作码，由 `afu_id + abi_version` 标识。

公共命令头建议至少包含：

```text
opcode
flags
afu_handle
context_id
request_id
input_count
output_count
timeout
```

数据缓冲区建议采用句柄或 scatter-gather 描述，而不是让应用直接依赖设备物理 DDR 地址。BAR 直接访问可以保留为可选的低延迟 fast path，但不应成为统一接口唯一的控制方式。

### 4. DMA 请求格式要与 Virtio 描述符语义分清

Virtqueue 描述符本身描述的是 guest memory 中的输入和输出缓冲区；专利请求头中的 `ddr_address` 则表示设备侧 DDR 地址。论文需要明确一次请求究竟执行：

- Host-to-Device；
- Device-to-Host；
- Device-to-Device；
- 内存注册或释放；
- AFU 命令提交。

建议定义明确的操作码，并为设备 DDR 缓冲区使用 PF/VF 管理器分配的 `buffer_handle`。后端根据 handle 查询租户地址范围，避免虚拟机直接构造任意设备地址。

### 5. eventq 必须预先提供可写缓冲区

“AFU 将事件经 eventq 发送给主机”在实现上意味着：

- 驱动预先向 eventq 放入一批可写描述符；
- 设备选择一个可用描述符写入事件；
- 设备更新 used ring 并按需通知驱动；
- 驱动消费事件后补充新的接收缓冲区。

需要定义事件格式，例如：

```text
event_type
afu_handle
context_id
request_id
status
device_timestamp
payload_length
```

DMA 完成通常可以通过 request/command queue 的 used buffer 表示；eventq 更适合承载异步 AFU 事件、故障、温度告警或配置变化，避免一个请求产生两套相互竞争的完成通知。

### 6. 需要利用 Virtio 特性协商和复位语义

建议定义 feature bits，用于协商：

- 多 AFU；
- 动态 AFU 热更新；
- 共享 DDR；
- eventq；
- 零拷贝设备内存；
- 每 AFU 复位；
- 有序执行或乱序完成；
- 安全 buffer handle；
- 状态保存与恢复。

Virtio 的设备级 reset 会停止整个设备的队列。如果一个 VF 内有多个 AFU，不能用整设备 reset 代替单 AFU 故障恢复，否则会影响同一租户内其他 AFU。因此还需要通过 controlq 定义 `RESET_AFU/RESET_CONTEXT`，并规定如何取消在途请求。

## 十八、三份专利整合后的控制面和数据面

建议在论文中明确区分两条路径。

### PF 控制面

负责：

- 读取和认证部分比特流；
- 发现 AFU 能力及接口版本；
- 分配 VF、AFU、BAR、MSI-X 和 DDR 资源；
- 构造影子配置表；
- 原子提交或失败回滚；
- 向 VF 触发 Virtio 配置变化通知；
- 解绑、排空和资源回收。

### VF 数据面

负责：

- Virtio feature negotiation 和队列建立；
- AFU 枚举与能力查询；
- 命令和 DMA 请求提交；
- MMIO fast path；
- virtqueue 完成通知和异步 eventq；
- 设备 DDR 地址检查；
- AFU 到 VF 的中断和事件路由。

这种划分能解释为什么管理操作由可信 PF 完成，而虚拟机可以通过 VF 直接访问数据面。

## 十九、三份专利是否应写成一篇论文

### 合并为一篇论文的条件

只有在具备以下原型时，才建议把三份专利写成一篇完整系统论文：

- FPGA VF 原生实现 Virtio PCI 接口或具有清晰的软件后端；
- 至少两个可动态加载或动态绑定的 AFU；
- 同一个 VF 能发现并访问多个 AFU；
- AFU 能共享租户级设备 DDR；
- 配置更新具有一致性和失败恢复机制；
- 同一个 guest 驱动和应用能够在至少两个不同 AFU 后端上不修改运行。

合并后的论文故事是：

> 一个具有统一 Virtio ABI、硬件直通性能和运行时 AFU 可重构能力的 FPGA 云虚拟加速器。

### 拆成两篇论文的建议

如果原型尚不能覆盖上述全链路，拆分通常更稳妥：

**论文 A：硬件架构与资源管理**

- SR-IOV VF 与 AFU 解耦；
- 多 AFU 聚合；
- BAR/MSI-X/DDR 虚拟化；
- 部分重配置和事务化绑定；
- 多租户隔离与零拷贝协同。

**论文 B：统一软件接口**

- Virtio-compatible accelerator device；
- AFU 能力发现；
- commandq/eventq 协议；
- 公共 ABI 与 AFU 专用 ABI；
- 跨后端可移植性和驱动复用。

从技术范围和审稿风险看，如果当前只有专利方案而缺少覆盖三层的完整实现，优先撰写论文 A；第三份专利可作为论文 A 的软件接口设计，但不要把“跨设备统一接口”列为未经实验验证的主要贡献。

## 二十、第三份专利加入后的新增实验

### RQ9：统一接口的性能代价

比较：

1. 原生厂商 MMIO/DMA 接口；
2. 软件中介 Virtio；
3. 硬件 Virtio over SR-IOV VF。

测量：

- 单次命令提交延迟；
- 4 KB 至大块 DMA 的吞吐率和延迟；
- virtqueue 通知与 MSI-X 中断延迟；
- 每秒命令数；
- guest 和 host CPU 占用率；
- FPGA 队列处理逻辑资源开销。

### RQ10：应用是否真正可移植

让相同的 guest 镜像、前端驱动和测试应用运行在：

- 不同 FPGA 型号；
- 同一功能的不同 AFU 实现；
- FPGA 后端与软件模拟后端；
- 如果条件允许，再加入 ASIC 或另一类加速卡。

记录：

- 无需修改的代码比例；
- 后端适配代码量；
- 驱动和应用是否需要重新编译；
- 能力差异如何通过 feature bits 处理；
- 性能是否仍接近原生接口。

仅证明“接口格式相同”不够；同一二进制或同一源代码不修改运行，才是跨设备统一性的直接证据。

### RQ11：动态更新时 Virtio 状态是否一致

在 guest 持续提交队列请求时动态改变 AFU 组合，验证：

- `config_generation` 正确变化；
- guest 收到配置变化通知；
- 旧 AFU handle 被拒绝；
- 在途请求得到完成、取消或明确错误；
- 队列不会访问已回收内存；
- 未变化 AFU 的请求不受影响。

## 二十一、综合三份专利后的题目建议

如果弱化统一接口、重点描述硬件资源管理：

> 面向 FPGA 云的动态多 AFU 聚合与 SR-IOV 虚拟加速器架构

> *A Dynamically Reconfigurable Multi-AFU SR-IOV Accelerator Architecture for FPGA Clouds*

根据当前已经确认的硬件 Virtio VF 实现，优先推荐：

> 面向 FPGA 云的 Virtio 兼容可重构组合式虚拟加速器

> *A Virtio-Compatible Reconfigurable and Composable Virtual Accelerator for FPGA Clouds*

如果只实现软件 Virtio 后端：

> 面向异构加速设备的 Virtio 统一接口与动态 AFU 管理机制

> *A Unified Virtio Interface with Dynamic AFU Management for Heterogeneous Accelerators*

由于当前已经采用硬件 Virtio SR-IOV VF，Virtio 可以进入论文主标题并列为核心体系结构贡献。但“统一”和“可组合”仍需分别由跨后端兼容性实验、AFU 间协同或功能链实验支撑；如果这些实验尚未完成，可先使用“多 AFU 聚合”替代“可组合”。

## 二十二、硬件 Virtio SR-IOV VF 路线下的论文收敛建议

### 1. 推荐的系统定位

论文不再定位为一般性的 Virtio 软件接口，而应定位为：

> 一种在 FPGA 静态区直接实现 Virtio PCI 数据面的 SR-IOV 虚拟加速器架构。每个硬件 VF 向虚拟机提供稳定的 Virtio ABI，并通过可事务化更新的资源映射层动态连接一个或多个可重构 AFU。

其核心优势是同时获得：

- Virtio 前端接口的稳定性和驱动复用能力；
- SR-IOV VF 直通的数据路径；
- FPGA 部分重配置带来的功能可变性；
- 一个 VF 聚合多个 AFU 的灵活性；
- 租户级 DDR 共享带来的设备内零拷贝协同。

### 2. 推荐的论文体系结构图

```text
┌──────────────────── Virtual Machine ────────────────────┐
│ Application → Runtime/API → Virtio Accelerator Driver  │
└──────────────────────────┬───────────────────────────────┘
                           │ VF passthrough
┌──────────────────────────┴───────────────────────────────┐
│ FPGA Static Region                                      │
│                                                         │
│ SR-IOV PF                                               │
│   └─ Management: PR, allocation, shadow-table commit    │
│                                                         │
│ SR-IOV VF                                               │
│   ├─ Virtio PCI common/device/notify configuration      │
│   ├─ Hardware virtqueue engine                          │
│   ├─ MSI-X notification                                 │
│   ├─ AFU discovery and command/event queues             │
│   └─ BAR / interrupt / DDR namespace and access checks  │
│                           │ trusted slot/context ID      │
│   VF–AFU transactional binding and routing              │
├───────────────────────────┼─────────────────────────────┤
│ FPGA Reconfigurable Region│                             │
│       AFU-1        AFU-2  │ ... AFU-n                  │
│           └──── Tenant-scoped shared DDR ────┘          │
└─────────────────────────────────────────────────────────┘
```

图中应使用粗实线标出 VF 直通数据路径，使用虚线标出 PF 配置路径，以直观证明 QEMU/vhost 不在正常数据路径中。

### 3. 论文中必须交代的硬件实现证据

不能只写“硬件支持 Virtio”，应逐项说明以下模块是否由 RTL/硬件实现：

- 每个 VF 的 Virtio PCI capability 暴露方式；
- device status 和 feature negotiation 状态机；
- virtqueue 配置、descriptor 获取、available/used ring 处理；
- queue notify 捕获和 doorbell 解析；
- DMA 读取描述符及访问 guest buffer 的路径；
- MSI-X 向量选择、合并和通知产生；
- device-specific configuration 与 `config_generation`；
- AFU 发现、命令分发和异步事件返回；
- VF reset、queue reset 及 AFU/context reset；
- Requester ID/IOMMU 对 guest 内存的隔离；
- VF 到 AFU/DDR 的第二级租户访问检查。

最好给出一次命令的完整时序：

```text
guest posts descriptor
→ writes VF notification BAR
→ hardware virtqueue engine fetches descriptor
→ validates VF/context/buffer
→ routes command to AFU
→ AFU completes
→ hardware updates used ring
→ generates VF MSI-X
→ guest driver consumes completion
```

### 4. 建议使用现代、非过渡式 Virtio PCI 接口

论文应说明实现遵循的 Virtio 版本以及兼容范围，并明确是 modern/non-transitional 还是 transitional 设备。对于新设计，建议采用 modern Virtio PCI capability 布局，避免把 legacy BAR 布局与现代接口混在一起。

由于异构/FPGA accelerator 不是可直接引用的成熟标准 Virtio 设备类型，论文中宜使用：

> hardware Virtio-compatible accelerator device

除非已经获得正式的设备 ID、规范采纳或完成严格一致性验证，否则不要写成“完全符合某个标准 Virtio accelerator device specification”。应在附录或公开材料中给出自定义 device-specific ABI、feature bits、队列编号和消息格式。

### 5. 最有辨识度的三项论文贡献

在当前实现路线下，推荐将贡献收敛为：

1. **硬件 Virtio over SR-IOV 数据面。** 在 FPGA 静态区为每个 VF 实现 Virtio PCI 和 virtqueue，使 guest 通过统一前端驱动直接访问硬件加速器，同时避免软件设备模拟进入正常数据路径。
2. **动态多 AFU 虚拟加速器。** 将 VF 与 AFU 解耦，在保持 Virtio ABI 和 PCIe 枚举状态稳定的情况下，将一个或多个动态 AFU 装配到 VF 后端，并统一虚拟化寄存器、事件和租户 DDR 资源。
3. **一致且隔离的在线更新。** 通过静默、排空、区域隔离、可信槽位身份、影子映射表、原子提交和 generation 校验，使 AFU 后端可在线变化且不破坏在途 virtqueue 请求及其他 VF 的服务。

其中第一项是第三份专利带来的核心差异，第二项来自前两份专利的结合，第三项是把专利工程方案提升为系统论文所需的完整机制。

### 6. 最关键的基线实验

硬件 Virtio 的价值必须通过以下三组对照体现：

| 基线 | 数据路径 | 用途 |
|---|---|---|
| 软件 Virtio | guest → QEMU/vhost/backend → FPGA | 量化消除软件中介后的收益 |
| 原生 SR-IOV 厂商接口 | guest vendor driver → VF → FPGA | 量化统一接口相对原生接口的代价 |
| 硬件 Virtio SR-IOV VF | guest Virtio driver → VF → FPGA | 本文方案 |

核心指标包括：

- 空操作或最小命令的往返延迟；
- 每秒 virtqueue 操作数；
- 不同块大小下的 DMA 吞吐率和延迟；
- P50/P95/P99 完成延迟；
- guest/host CPU 使用率；
- MSI-X 中断率和中断合并效果；
- Virtio 队列引擎的 LUT、FF、BRAM 和时钟开销；
- VF 数量增加时的资源增长和吞吐扩展性。

理想结论不是笼统地宣称“兼顾兼容性和性能”，而是用数据说明：硬件 Virtio 相比软件 Virtio减少多少延迟和 CPU 开销，相比原生厂商接口增加多少硬件资源或性能损失。

### 7. 动态重配置与 virtqueue 的交叉实验

这是三份专利真正结合的关键实验。建议在 VF 持续提交请求时替换或重新组合后端 AFU，并记录：

- 停止接收新请求到恢复服务的时间；
- 在途 descriptor 的完成、取消和错误数量；
- `config_generation` 变化与 guest 重新发现耗时；
- 未参与更新 VF 的吞吐和 P99 延迟变化；
- 旧 AFU handle、旧 buffer handle 和旧 generation 请求是否全部被拒绝；
- 重配置失败后是否能回滚并继续使用原 AFU；
- PCIe VF 是否始终保持已枚举状态，guest 驱动是否无需卸载和重新加载。

最后一项尤其重要，它直接验证“稳定虚拟设备接口后端动态变化”的核心主张。

### 8. 更新后的首选题目

如果已经实现多 AFU 绑定，但尚未实现完整功能链：

> **ReVirt：面向 FPGA 云的硬件 Virtio SR-IOV 与动态多 AFU 虚拟加速器架构**

> **ReVirt: A Hardware Virtio SR-IOV Architecture for Dynamic Multi-AFU Virtual Accelerators in FPGA Clouds**

如果还实现了 AFU 流水线、同步和零拷贝功能链：

> **ReVirt：面向 FPGA 云的硬件 Virtio SR-IOV 可重构组合式加速器**

> **ReVirt: A Hardware Virtio SR-IOV Architecture for Reconfigurable and Composable FPGA Accelerators**

在没有完整功能链执行证据前，首选第一个题目，使用“动态多 AFU”而不是“可组合”。

## 二十三、第四份专利带来的核心变化

第四份专利不是简单增加一种调度策略，而是把前三份专利中的 VF–AFU 关系从一对一或一对多扩展为多对多：

```text
VF-1 ─┬─ AFU-A/context-0
      └─ AFU-B/context-2

VF-2 ─┬─ AFU-A/context-1
      └─ AFU-C/context-0

VF-3 ─── AFU-A/context-3
```

这形成两种互补的资源共享方式：

- **空间聚合：** 一个 VF 同时绑定多个不同 AFU，为一个租户组成多功能虚拟加速器；
- **时间复用：** 一个 AFU 维护多个租户上下文，按调度策略依次执行来自多个 VF 的任务。

四份专利由此可以组成以下统一研究问题：

> 如何以硬件 Virtio SR-IOV VF 作为稳定租户接口，在可重构 FPGA 上实现 AFU 的空间聚合和跨 VF 时间复用，同时保证队列语义、地址隔离、性能公平性以及动态更新的一致性？

“提高利用率”应作为实验结果，而不是唯一的论文贡献。论文真正的机制贡献是：多对多绑定、上下文隔离、硬件调度以及与 Virtio 队列和动态重配置的一致协同。

## 二十四、用统一的 VF–AFU 绑定上下文替代分散表项

前述专利分别设计了多张 VF 表、AFU 表和正反向映射表。加入 AFU 多 VF 共享后，同一关系会同时出现在更多位置，一致性风险进一步增大。

建议在论文中抽象出统一的 `Binding Context`：

```text
binding_id
vf_id
afu_slot_id
afu_context_id
afu_handle
queue_or_group_id
register_base / register_size
ddr_base / ddr_limit
event_or_msix_mapping
weight / credit
generation
state
```

其语义是：每一条 VF–AFU 边对应一个独立且可验证的执行上下文。

```text
VF → binding_id → AFU slot/context       // 命令、MMIO 和 DDR 路由
AFU slot/context → binding_id → VF/queue // 完成和事件反向路由
```

为了防止动态 AFU 伪造身份，`afu_slot_id` 应由静态区物理入口附加；`vf_id` 应来自 PCIe Requester ID 或 VF 硬件端口；两者均不能由不可信请求报文自行声明。

绑定更新继续使用影子表和原子提交，并为每个上下文维护以下状态：

```text
FREE → ALLOCATED → ACTIVE → QUIESCING → DRAINED → FREE
                               └──────→ ERROR
```

解绑前必须同时排空：

- 该 VF virtqueue 中尚未获取的命令；
- 已进入 AFU 调度队列的命令；
- 正在执行的任务；
- 未写回 used ring 的完成项；
- 未被 guest 消费的异步事件；
- 未完成的 DDR 读写请求。

## 二十五、第四份专利的“分时”应准确称为非抢占式复用

专利描述的调度过程是：选择一个就绪上下文，让 AFU 执行任务，等待任务自然结束后再选择下一个上下文。除非已经实现 AFU 内部状态保存、暂停和恢复，否则这属于：

> non-preemptive run-to-completion multiplexing

它不是传统操作系统意义上具有固定时间片的抢占式调度。论文应避免直接声称：

- 支持任务抢占；
- 支持任意粒度时间片；
- 能为长任务提供严格延迟上界。

非抢占式方案仍然有价值，优势是：

- 上下文切换开销低；
- 不要求捕获 AFU 内部流水线和片上存储状态；
- 容易适配不同类型 AFU；
- 适合短任务或天然可分块的任务。

但需要明确其局限：长任务会造成队首阻塞，其他 VF 的尾延迟可能显著增加。可以采用以下缓解方法：

- 为 AFU 命令规定最大工作量或最大执行时间；
- 将大任务拆成可独立完成的 tile/chunk；
- 在 chunk 边界重新调度；
- 设置 watchdog，超时后复位对应 context；
- 对延迟敏感任务采用独占 AFU 或专用队列。

如果未来真正实现抢占，应额外描述需要保存和恢复的寄存器、片上 RAM、流水线状态以及未完成 DDR 事务，并测量上下文保存、切换和恢复开销。

## 二十六、现有调度策略需要修正

### 1. 固定优先级可能饥饿

持续有高优先级任务时，低优先级 VF 可能永远不能运行。除非论文明确把它设计为实时优先级调度，否则需要 aging、预算或最大连续服务次数。

### 2. 轮询适合等长任务，不保证时间公平

如果各任务运行时间差异很大，每个 VF 轮流执行一个任务并不等于获得相同的 AFU 时间份额。实验应同时覆盖等长任务和长短任务混合场景。

### 3. “每次选择最大 weight”不等于加权轮询

如果静态选择当前所有请求中权重最大的上下文，低权重 VF 会长期饥饿。推荐实现以下一种：

- Weighted Round Robin；
- Deficit Round Robin；
- 基于 credit 的 proportional-share 调度；
- 如果有任务成本估计，使用按预计执行周期扣减的 weighted fair scheduling。

对于执行时间差异明显的 AFU，建议按实际或估计的 AFU busy cycles 扣减 credit，而不是简单按“任务个数”扣减。

一个可实现的硬件 credit 调度框架为：

```text
每个 context 维护 credit_i
每个调度周期：credit_i += weight_i
选择 ready 且 credit 足够的 context
任务完成：credit_i -= measured_or_estimated_cost
若无 context 可执行：归一化或补充 credit
```

论文应报告调度器的选择延迟、资源开销、最大上下文数和时钟频率。

## 二十七、Virtio 队列与 AFU 上下文的衔接

第四份专利基于“每组上下文寄存器只有一个启动位”的模型，实际上每个 VF/AFU 上下文通常只能保留一个待执行任务。这与 virtqueue 支持多个 outstanding descriptors 的语义并不完全匹配。

建议采用以下数据路径：

```text
VF commandq descriptor
    → hardware virtqueue engine
    → extract afu_handle, request_id and buffer handles
    → resolve Binding Context
    → enqueue into per-AFU/per-context request FIFO
    → AFU scheduler selects context
    → load selected command into execution registers
    → AFU executes
    → completion router uses binding_id + request_id
    → update the correct VF used ring
    → generate that VF's MSI-X notification
```

必须保证：

- 一个 context 可以排队多个请求，或通过 credit 明确限制队列深度；
- 每个请求都有唯一 `request_id`，完成路由不能只依赖 context number；
- VF reset 时只取消该 VF 的请求，不错误清空其他 VF 的共享 AFU 任务；
- AFU reset 的影响范围被明确通知给所有共享该 AFU 的 VF；
- 一个 VF 不能通过填满共享 FIFO 阻塞其他 VF。

推荐采用分 context FIFO 或保留队列配额，再由 AFU 调度器从非空 context 中选择。单一共享 FIFO 容易产生队首阻塞，也难以实现权重和隔离。

## 二十八、共享 AFU 的隔离与服务质量

地址空间隔离只是多租户安全的一部分。共享 AFU 后还需考虑：

- **寄存器隔离：** 每个 context 只能访问自己的配置状态；
- **DDR 隔离：** 每个请求都使用该 Binding Context 的 DDR 边界检查；
- **队列隔离：** 一个 VF 的积压不能耗尽所有 FIFO 项；
- **中断隔离：** 完成必须返回原 VF 的 virtqueue/MSI-X；
- **复位隔离：** context reset 与 AFU-wide reset 的影响范围不同；
- **性能隔离：** 权重、最大队列深度、最大任务时间和 DDR 带宽配额；
- **信息泄露：** 上下文切换时清除私有寄存器、片上缓存和残留错误状态；
- **拒绝服务：** 恶意 VF 不能提交无限长任务或持续占满设备内存带宽。

建议为每个 Binding Context 增加：

```text
max_outstanding
max_task_cycles
ddr_bandwidth_credit
scheduler_weight
error_count
reset_generation
```

共享设备 DDR 可能成为性能隔离的主要瓶颈。即使 AFU 调度公平，如果某个 VF 的任务大量占用 DDR，其他 AFU 和 VF 仍会受到干扰。因此实验必须同时测 AFU 执行份额和 DDR 带宽份额。

## 二十九、AFU 共享与动态部分重配置存在生命周期冲突

一个 AFU 被多个 VF 共享后，不能在任意时刻直接部分重配置，否则会同时破坏多个租户的上下文和在途任务。需要为 AFU 维护引用计数和重配置状态机：

```text
ACTIVE_SHARED
    → QUIESCING：拒绝新绑定和新任务
    → DRAINING：完成或取消所有 context 请求
    → REVOKED：通知所有相关 VF，旧 handle 失效
    → ISOLATED
    → RECONFIGURING
    → VALIDATING
    → REBINDING / ACTIVE
```

可选择三种更新策略：

1. **等待排空：** 实现简单，但更新时间取决于最长任务；
2. **迁移上下文：** 若另一相同 AFU 有容量，将待执行请求迁移后再更新；
3. **版本并存：** 在另一动态槽位加载新版本，新请求切到新 AFU，旧 AFU 完成存量请求后下线。

第三种类似蓝绿更新，最适合展示“业务连续性”，但会临时占用更多 FPGA 资源。论文可比较三种策略的更新时间、额外资源和尾延迟。

## 三十、第四份专利加入后的实验设计

### RQ12：AFU 共享能提高多少有效利用率

比较：

1. VF–AFU 独占绑定；
2. 多个 VF 共享 AFU，但由软件调度；
3. 本文的硬件上下文与 AFU 调度器。

使用具有突发和空闲阶段的请求流，改变 VF 数量、到达率和任务长度。测量：

- AFU utilization：`busy_cycles / observation_cycles`；
- 总吞吐率；
- 单位时间完成任务数；
- 空闲间隙比例；
- guest/host CPU 开销；
- 每瓦吞吐率。

必须保证各方案使用相同请求集合和观察窗口，避免仅通过增加排队量得到表面上的高利用率。

### RQ13：共享后的公平性和尾延迟如何

改变共享同一 AFU 的 VF 数量，例如 1、2、4、8，并比较 round-robin、错误的 max-weight 选择、改进后的 weighted/credit scheduler。测量：

- 每个 VF 的吞吐份额；
- 权重目标与实际份额误差；
- Jain's fairness index；
- 平均、P95、P99 和最大排队延迟；
- 长短任务混合时的队首阻塞；
- 低权重 VF 的最大等待时间。

### RQ14：上下文机制的硬件代价与扩展性如何

改变每个 AFU 支持的 context 数量，测量：

- 每个 context 增加的 LUT、FF、BRAM；
- 调度决策周期数；
- 最大工作频率；
- 请求 FIFO 存储开销；
- context switch 间隙；
- VF 和 AFU 数量同时扩展时映射表查找延迟。

如果当前每个 context 都复制一整组配置寄存器，应与“命令描述符存放在 BRAM/DDR、执行时加载”的方案比较资源增长。

### RQ15：租户能否得到性能隔离

设置一个攻击性或高负载 VF，持续提交：

- 最大长度计算任务；
- 高带宽 DDR 请求；
- 大量短任务；
- 非法地址和错误命令。

观察其他 VF 的吞吐、P99 延迟和错误率，验证队列配额、任务 watchdog、DDR credit 和上下文复位机制。

### RQ16：共享 AFU 能否安全在线更新

在多个 VF 持续使用同一 AFU 时执行部分重配置，比较等待排空、迁移和版本并存策略，记录：

- 停止接收新任务后的排空时间；
- 被完成、取消或迁移的请求数；
- 各 VF 的服务中断时间；
- 未参与该 AFU 的其他 VF/AFU 的影响；
- 旧上下文和旧 generation 请求的拒绝情况；
- 更新失败后的回滚时间。

## 三十一、相关工作定位的进一步调整

相关工作应从“FPGA 虚拟化”进一步细分为：

- FPGA 空间共享与动态分区；
- FPGA 时间共享、上下文切换和任务调度；
- 多租户隔离与 QoS；
- SR-IOV/Virtio 硬件虚拟化；
- 动态部分重配置与在线更新。

AMORPHOS 讨论了 FPGA 的空间与时间共享，并明确指出不保存硬件状态时，时间共享通常只能采用非抢占方式或强制撤销，因此可作为本文“非抢占、运行至完成”定位的重要参照。[AMORPHOS](https://www.usenix.org/sites/default/files/osdi18_full-proceedings.pdf)

已有 FPGA 多租户架构也已经展示空间共享能够提高利用率，因此本文不能只以“多个租户共享 FPGA”作为创新点，而应突出硬件 Virtio SR-IOV VF、多对多 VF–AFU 绑定、每 AFU 上下文调度以及在线重配置的一体化。[Architecture Support for FPGA Multi-tenancy in the Cloud](https://arxiv.org/abs/2006.08026)

近期 vFPIO 已包含面向多租户并发 I/O 的抢占式事务调度，所以若论文讨论调度，还需明确本文调度对象是“多个直通 VF 对共享计算 AFU 的任务调度”，而不是一般 I/O transaction scheduling。[vFPIO](https://www.usenix.org/conference/atc24/presentation/chen-jiyang)

## 三十二、综合四份专利后的论文贡献

建议最终贡献收敛为四项：

1. **硬件 Virtio SR-IOV 接口。** 在 FPGA 静态区实现 Virtio PCI 和 virtqueue 数据面，使虚拟机通过统一驱动直接访问 VF，避免软件设备模拟进入正常数据路径。
2. **多对多 VF–AFU 虚拟化。** 一个 VF 可聚合多个 AFU，一个 AFU 可通过独立 Binding Context 被多个 VF 共享，从而同时支持空间聚合与时间复用。
3. **隔离且公平的硬件任务调度。** 通过可信上下文标识、独立请求队列、租户 DDR 边界、加权 credit 调度和精确完成路由，提高 AFU 利用率并控制租户间干扰。
4. **事务化动态更新。** 将 virtqueue 排空、AFU 上下文生命周期、影子映射表和部分重配置状态机结合，在后端 AFU 变化时保持 VF 接口稳定并支持失败回滚。

如果当前没有实现 credit/weighted scheduler 或性能隔离，第三项应改成：

> 基于独立上下文的非抢占式 AFU 共享机制。

不要在实现只有简单轮询时宣称“公平且支持 QoS”。

## 三十三、更新后的首选题目

如果当前实现为非抢占式 AFU 共享，首选：

> **ReVirt：面向 FPGA 云的硬件 Virtio SR-IOV 与 AFU 时空共享架构**

> **ReVirt: A Hardware Virtio SR-IOV Architecture for Spatio-Temporal AFU Sharing in FPGA Clouds**

如果已实现权重公平、队列配额和 DDR QoS，可使用：

> **ReVirt：面向多租户 FPGA 云的硬件 Virtio SR-IOV 与高效隔离的 AFU 共享机制**

> **ReVirt: Efficient and Isolated AFU Sharing with Hardware Virtio SR-IOV for Multi-Tenant FPGA Clouds**

如果论文篇幅有限，建议把“多 AFU 功能链组合”作为扩展能力，将主线集中在：

```text
硬件 Virtio VF
→ 多对多 VF–AFU 绑定
→ 非抢占式上下文调度
→ 利用率、公平性与隔离
→ 共享 AFU 的在线重配置
```

这条主线比同时强调接口统一、功能组合、资源分配、调度和所有动态能力更集中，也最能体现第四份专利带来的增量价值。

## 三十四、第五份专利在论文中的定位

第五份专利解决的是硬件 Virtio 与加速器异步事件之间的接口缺口：

- Virtio PCI transport 已经管理配置变化和 virtqueue 通知；
- FPGA AFU 还可能产生计算异常、设备故障、性能监控等队列之外的事件；
- 用额外 eventq 传递事件符合 Virtio 风格，但需要描述符读写、used ring 更新和队列通知，路径较长；
- 直接使用额外 MSI-X 向量可以降低通知延迟，但必须与 Virtio PCI transport 的向量分配和回收保持一致。

它适合放在论文的“Implementation: Interrupt and Event Subsystem”中，用来解释：

1. 硬件 VF 如何预留足够的 MSI-X 表项；
2. 配置变化、virtqueue 和额外 AFU 事件如何划分向量命名空间；
3. Linux 驱动如何在不破坏 Virtio transport 生命周期的情况下注册额外 handler；
4. AFU 事件如何根据 Binding Context 路由到正确 VF；
5. 向量不足时如何退化为共享向量或 eventq。

第五份专利的价值主要是补齐实现闭环并降低异步事件延迟。除非实验显示显著的中断延迟、CPU 或 FPGA 资源收益，不建议将其列成与硬件 Virtio、多对多 VF–AFU 共享同等级的独立论文贡献。

## 三十五、专利描述中必须核实的 Linux IRQ 语义

### 1. Linux IRQ 编号不保证连续

`pci_alloc_irq_vectors()` 返回的是已分配向量数量。驱动必须对每个设备相对向量索引 `i` 调用：

```c
irq = pci_irq_vector(pdev, i);
```

才能获得传给 `request_irq()` 或 `free_irq()` 的 Linux IRQ 编号。不能记录首个 IRQ 后用 `irq_first + i` 推导其他 IRQ。

MSI 模式对硬件消息数量有连续性约束，不代表 Linux 分配的 IRQ number 在数值上连续；MSI-X 更不应作此假设。[Linux MSI Driver Guide](https://cdn.kernel.org/doc/html/latest/PCI/msi-howto.html)

### 2. MSI-X 表项不保存 Linux IRQ Number

论文需要区分三个概念：

| 概念 | 含义 |
|---|---|
| MSI-X table index | 设备内部表项序号，例如 0、1、2 |
| MSI-X message address/data | PCI 子系统配置到表项中的中断消息内容 |
| Linux IRQ number | 内核用于 `request_irq()` 的软件编号 |

硬件产生中断时选择的是 MSI-X table index，PCI/Linux 子系统负责表项编程和 IRQ 映射。专利中“把系统 IRQ Number 写入 MSI-X 表”的说法在论文中应修正为“PCI 子系统为对应 MSI-X 表项配置消息地址和数据，驱动通过 `pci_irq_vector()` 获取其 Linux IRQ 映射”。

### 3. 不能在 handler 仍注册时直接释放 vectors

如果 `virtio_find_vqs()` 已经为配置变化和 virtqueue 注册了 handler，直接调用 `pci_free_irq_vectors()` 会使这些 handler 和 Virtio PCI transport 保存的向量状态失效。正确清理顺序必须包括：

```text
mask/stop device interrupt sources
→ synchronize_irq
→ free_irq for every registered handler
→ release MSI-X vectors
```

标准 Virtio PCI transport 的 `vp_del_vqs()` 已负责解除 virtqueue/config handler 并释放 vectors；设备驱动不应在 transport 不知情的情况下抢先释放其资源。

### 4. 重新分配后不能只比较首个 IRQ

即使重新分配得到的第一个 Linux IRQ 与旧值相同，也不能推断所有向量映射均未变化。必须逐个重新调用 `pci_irq_vector(pdev, i)`，重新注册 handler，并重新建立设备向量索引映射。

Linux 内核自己的 PCI 驱动在释放并缩减后重新分配 vectors 时，也明确在重新分配完成后重新调用 `pci_irq_vector()`，因为具体映射可能变化。[Linux PCIe port driver example](https://github.com/torvalds/linux/blob/master/drivers/pci/pcie/portdrv.c)

### 5. Virtio transport 可能采用共享向量回退

不能无条件假设向量数量恒为：

```text
1 config + num_queues + num_extra_interrupts
```

当前 Linux Virtio PCI transport 会依次尝试：

1. 配置变化一个向量，每个 virtqueue 一个向量；
2. 慢路径队列共享向量，其他队列独立向量；
3. 配置变化一个向量，所有队列共享一个向量；
4. 必要时回退到 INTx。

Linux 源码中的 `vp_find_vqs()` 和 `vp_request_msix_vectors()`体现了这些策略。[Linux virtio_pci_common.c](https://github.com/torvalds/linux/blob/master/drivers/virtio/virtio_pci_common.c)

如果硬件和驱动要求额外事件使用独立 MSI-X，就必须把“MSI-X per queue + extra vectors”声明为设备运行的必要条件，并在分配不足时失败或明确降级，不能继续假设固定表项布局。

## 三十六、推荐的中断实现方案

第五份专利提出的“先让 Virtio 分配、再释放并扩大分配”流程工程风险较高。论文原型优先考虑下面两种实现。

### 方案 A：在 Virtio PCI transport 初次分配时一次性预留

在建立 virtqueues 前已知：

```text
standard_vectors = config_vector + queue_vectors
extra_vectors = accelerator_event_vectors
total_vectors = standard_vectors + extra_vectors
```

让 transport 在第一次调用 `pci_alloc_irq_vectors_affinity()` 时一次性申请 `total_vectors`，随后：

- 前部向量由 config 和 virtqueues 使用；
- 后部向量由 accelerator driver 注册额外 handler；
- transport 统一保存并释放全部向量；
- driver 仅管理额外 handler，不单独释放 transport 所拥有的 vector allocation。

优点是生命周期清晰、适用于固定内核版本的原型；缺点是通常需要对 Virtio PCI transport 增加一个很小的扩展接口，应如实报告内核修改行数。

### 方案 B：使用动态 MSI-X 分配 API

较新的 Linux PCI 子系统提供：

```c
pci_msix_can_alloc_dyn()
pci_msix_alloc_irq_at()
pci_msix_free_irq()
```

可以在 Virtio 标准 vectors 已建立后，从未使用的 MSI-X 表项中动态分配额外向量，而不释放和重建现有 virtqueue 中断。成功后应使用返回的 `msi_map.virq` 注册 handler，清理时先 `free_irq()`，再释放动态 MSI-X 映射。[Linux MSI Driver Guide](https://cdn.kernel.org/doc/html/latest/PCI/msi-howto.html)

优点是不扰动已有 Virtio 队列；缺点是依赖内核、体系结构和虚拟化路径对动态 MSI-X 的支持，需要在 VFIO passthrough 场景实际验证。

### 兼容性回退

如果不能获得额外 MSI-X vectors，可按优先级降级：

1. 多个低频额外事件共享一个 MSI-X，并读取只读 cause bitmap；
2. 通过一个共享 eventq 传递带类型和上下文的事件；
3. 将计算完成继续作为 commandq used-buffer completion，不额外产生事件中断。

论文应明确当前原型采用方案 A、方案 B 还是原专利流程，并给出内核版本及改动范围。

## 三十七、计算完成与异步事件应分流

第五份专利把“计算完成”列为额外硬件中断示例，但在 Virtio commandq 模型中，正常请求完成已经可以表示为：

```text
AFU completes
→ hardware writes completion status
→ updates used ring
→ triggers commandq MSI-X notification
```

如果同时产生 used-ring notification 和额外 completion IRQ，会造成：

- 一次完成对应两次通知；
- 驱动需要合并两个竞态到达的状态；
- 增加中断率；
- 破坏通用 Virtio 请求—完成语义。

因此建议：

- **正常命令完成：** 统一走 commandq used ring；
- **与请求无直接对应关系的事件：** 走额外 MSI-X 或 eventq；
- **严重故障：** 额外 MSI-X，同时在错误状态寄存器或事件记录中保存原因；
- **配置变化：** 使用 Virtio 标准 config change notification。

适合额外 MSI-X 的事件包括：

- AFU fatal error；
- DDR ECC/访问保护错误；
- 温度或功耗阈值；
- watchdog timeout；
- 性能计数器溢出；
- PF 发起的上下文撤销或紧急通知。

## 三十八、额外中断的硬件路由与安全

第五份专利加入多对多 VF–AFU 架构后，AFU 不能直接提供目标 VF 或 MSI-X table index。安全路由应为：

```text
AFU physical slot emits <trusted_slot_id, context_id, event_type>
→ event router resolves Binding Context
→ obtains <vf_id, event policy, vector index, generation>
→ validates context state and event type
→ updates pending/cause state
→ triggers that VF's MSI-X table entry
```

需要处理：

- VF 已 reset 或解绑时丢弃旧 generation 事件；
- MSI-X vector 被 mask 时设置 pending bit，不能丢失事件；
- 同一事件连续发生时选择计数、合并还是逐个通知；
- 恶意 AFU 的中断风暴限制；
- PF 更新路由表时采用影子表和原子提交；
- 多个 AFU 共享同一额外 vector 时通过 cause bitmap 或事件记录区分来源；
- VF passthrough 后由 guest 控制 MSI-X mask，硬件必须遵守对应表项状态。

可以在 Binding Context 中增加：

```text
event_vector_base
event_vector_count
event_rate_limit
event_pending_bitmap
event_generation
```

MSI-X 表大小在 PCIe 枚举后不能随 AFU 动态组合任意改变，因此每个 VF 应在静态区预留最大向量能力，运行时只分配或启用其中的子集。

## 三十九、第五份专利对应的实验

### RQ17：直接额外 MSI-X 是否值得

比较三条事件路径：

1. 额外 eventq；
2. 共享 MSI-X + cause register/bitmap；
3. 独立额外 MSI-X vector。

测量：

- FPGA 事件产生到 guest handler 执行的 P50/P95/P99 延迟；
- guest 中断处理 CPU 时间；
- 每秒最大事件数；
- 高事件率下的丢失、合并和抖动；
- FPGA LUT、FF、BRAM 和功耗；
- 对正常 commandq 吞吐和尾延迟的影响。

这组实验可以决定第五份专利是值得在论文正文突出，还是仅作为实现细节说明。

### RQ18：中断扩展是否在生命周期操作下保持正确

反复执行：

- VF reset；
- AFU context reset；
- VF–AFU 绑定和解绑；
- AFU 部分重配置；
- guest 驱动卸载和重载；
- MSI-X mask/unmask；
- CPU affinity 变更。

验证：

- 没有 use-after-free handler 或错误 IRQ 映射；
- 没有事件路由到其他 VF；
- 解绑后的旧 AFU 事件被 generation 检查拒绝；
- mask 期间的必要事件在 unmask 后仍可观察；
- 中断释放顺序不产生 kernel warning、卡死或丢失完成；
- 重复加载驱动后 vector 数和 handler 数保持稳定。

### RQ19：额外向量数量如何扩展

改变 VF 数量、每 VF virtqueue 数和 extra vector 数，测量：

- 可成功分配的最大配置；
- 中断初始化和卸载时间；
- MSI-X 表及 FPGA 中断路由资源开销；
- 向量共享回退后的性能变化；
- 不同 guest 内核和 VFIO/IOMMU 配置下的兼容性。

## 四十、第五份专利加入后的最终论文取舍

第五份专利不改变当前最合适的论文题目和主线：

> **ReVirt：面向 FPGA 云的硬件 Virtio SR-IOV 与 AFU 时空共享架构**

它应作为主线中的一个关键工程机制：

```text
硬件 Virtio VF
→ 标准 config/virtqueue 通知
→ 可选的直接 AFU 异步 MSI-X
→ Binding Context 安全路由
→ VF reset、解绑和重配置的一致生命周期
```

正文篇幅建议控制为一个实现小节和一组针对性实验。论文贡献列表仍以硬件 Virtio VF、多对多 VF–AFU 共享、隔离调度和事务化重配置为主；只有直接 MSI-X 相比 eventq 获得显著且稳定的端到端收益时，才在贡献中增加“低延迟可扩展事件机制”。

## 四十一、PPT 对论文定位的最大价值

PPT 将 FPGA 云服务分为 SaaS、PaaS 和 IaaS，并把目标系统定位为类似 GPU 加速库的 FPGA PaaS：用户感知可调用的加速功能，但不直接修改底层 FPGA 逻辑。

这与五份专利组合后的 ReVirt 架构高度吻合：

| PaaS 需求 | ReVirt 对应机制 |
|---|---|
| 用户不关心具体 FPGA 型号 | Virtio-compatible 稳定 ABI |
| 一个虚拟机申请一个或多个功能 | 一个 VF 聚合多个 AFU |
| 多个用户共享热门功能 | 一个 AFU 通过上下文服务多个 VF |
| 功能库存按需变化 | FPGA 部分重配置和动态绑定 |
| 多租户高利用率 | AFU 硬件调度和时间复用 |
| 用户间数据隔离 | VF/Binding Context/DDR 边界检查 |
| 统一驱动和运行时库 | 硬件 Virtio PCI 接口 |
| 功能变化不重启 VM | 稳定 VF、配置 generation 和事务化更新 |

因此，论文中的云场景不应仅写成“多个虚拟机共享 FPGA”，而可以更准确地定义为：

> ReVirt 是 FPGA Accelerator-as-a-Service/PaaS 的节点内执行底座，将异构 AFU 封装为可发现、可绑定、可共享且具有稳定 Virtio ABI 的虚拟加速器功能。

这一定位能够解释为什么系统同时需要统一接口、动态功能装配、多租户共享和生命周期管理。

## 四十二、论文必须控制云基础设施范围

PPT 描述了较完整的演进路线：

```text
单功能 FPGA
→ SR-IOV/Virtio 动态 VF
→ SIOV/ADI 细粒度虚拟设备
→ 节点内 FPGA 池化
→ 跨节点 FPGA 池化
```

这些内容不能全部作为当前 ReVirt 论文的已实现贡献。建议分层处理：

### 当前论文正文范围

- 单服务器、PCIe 本地 FPGA；
- 硬件 Virtio SR-IOV VF；
- VF 与 AFU 多对多绑定；
- 节点内 AFU 空间聚合和时间复用；
- 本地设备 DDR 共享；
- 动态部分重配置；
- PF 控制面与 guest 直通数据面。

### 可选系统集成

如果已经实现，可加入：

- OpenStack Cyborg/Nova 的资源发现、调度和 VF 透传；
- Kubernetes DRA 的设备声明和分配；
- AFU bitstream 仓库及兼容性校验；
- 节点级 ReVirt agent。

### 讨论与未来工作

- SIOV/ADI/PASID；
- VDCM 和跨代设备兼容；
- VM 热迁移；
- CXL/RDMA 远程访问；
- 单节点多卡及跨节点 FPGA 池化；
- 跨卡 AFU 功能链。

如果没有对应实现和实验，不应在摘要或贡献中声称“实现 FPGA 池化、热迁移或完整 PaaS”。这些内容放在 Discussion/Roadmap 中更合适。

## 四十三、从用户申请到 VF 交付的端到端服务流程

PPT 中的服务流程可以改写成论文中的控制平面工作流：

```text
1. 用户提交 Accelerator Profile
   <AFU function/version, count, DDR, QoS, isolation, composition>

2. 云控制面匹配服务器
   CPU/内存约束 + FPGA shell/slot/AFU/context/VF 约束

3. 节点 ReVirt agent 预留资源
   VF + AFU slots/contexts + DDR + queues + MSI-X

4. PF 管理面准备后端
   验证 bitstream → 必要时部分重配置 → 创建影子 Binding Context

5. 原子提交配置
   VF–AFU binding + DDR/queue/event routing + generation

6. 云平台创建 VM 并直通 VF

7. Guest Virtio driver 初始化
   feature negotiation → virtqueue setup → AFU capability discovery

8. 用户通过统一 runtime/API 调用 AFU

9. VM 结束或服务释放
   quiesce → drain → unbind → reclaim context/DDR/vector/VF
```

论文需要明确资源预留和 VM 启动之间的失败处理。例如：AFU 配置成功但 VM 创建失败时，必须回收 VF、上下文和 DDR；如果部分重配置失败，云调度器应尝试其他 FPGA 或返回确定的失败状态。

## 四十四、OpenStack 和 Kubernetes 的准确落点

### OpenStack

PPT 提到的 Cyborg 仍然适合作为 OpenStack 集成点。Cyborg 当前是通用加速器管理框架，通过 vendor driver 发现 FPGA/GPU/NIC 等设备，并与 Nova Placement 协作；用户请求通过 device profile 和 Accelerator Request（ARQ）表达，ARQ 随后绑定到主机、设备资源提供者和实例。[OpenStack Cyborg](https://docs.openstack.org/cyborg/latest/)、[Cyborg accelerator lifecycle](https://docs.openstack.org/cyborg/latest/admin/)

ReVirt Cyborg driver 可以上报：

```text
card_id / numa_node
shell_id / shell_abi
PR slot geometry and state
available AFU IDs and versions
free AFU contexts
free VF count
per-VF queue/MSI-X/BAR capability
available DDR capacity
sharing and isolation capability
```

需要注意，当前 Cyborg 支持矩阵中的多种 FPGA driver 仍标记为 experimental，因此论文中应说“基于 Cyborg 的原型集成”或“可集成设计”，不要把生态成熟度描述得过高。[Cyborg support matrix](https://docs.openstack.org/cyborg/latest/admin/support-matrix.html)

### Kubernetes

PPT 只提到 Device Plugin，但当前更适合表达复杂 FPGA 请求的是 Dynamic Resource Allocation（DRA）。DRA 支持设备类别、声明式 ResourceClaim、设备共享和基于属性的选择，更适合表达 AFU 功能、版本、上下文数量、DDR 和拓扑约束；传统 Device Plugin 更适合固定整数型设备资源。[Kubernetes Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/)

如果论文没有实现容器路径，只需在 Discussion 中说明 ReVirt 资源模型可映射到 DRA，不必同时实现 OpenStack 和 Kubernetes 两套集成。

## 四十五、FPGA PaaS 的角色与可信边界

PPT 定义了三个角色：

- FPGA 加速器设备提供者；
- 业务算法/AFU IP 提供者；
- FPGA PaaS 云服务提供者。

论文应将其转化成清晰的可信边界：

| 角色 | 提供内容 | 是否默认可信 | 需要的检查 |
|---|---|---|---|
| 设备提供者 | FPGA card、shell、驱动和开发包 | 平台信任根 | 固件版本、shell 认证 |
| AFU IP 提供者 | partial bitstream、manifest、AFU ABI | 不应默认完全可信 | 签名、兼容性、接口和资源检查 |
| 云服务提供者 | 调度、装载、VF 分配和隔离 | 可信控制面 | 审计、配额、回滚 |
| 租户 | 命令、数据和资源请求 | 不可信 | 地址、队列、命令和速率检查 |

PPT 中 bitstream 只包含 `Shell ID + AFU ID` 不足以支撑可靠 PaaS。建议 manifest 至少包括：

```text
artifact_digest and signature
afu_id / afu_version / afu_abi_version
shell_id / shell_abi_version
FPGA family and board compatibility
PR slot geometry / interface hash
clock and reset requirements
register/queue/event capability
DDR and bandwidth requirements
max task duration / preemption capability
resource usage and bitstream size
provider identity and security policy
```

PF 在重配置前验证 manifest、签名、shell compatibility 和资源需求；配置后再从可信静态区读取 AFU 实际能力进行二次校验，防止 manifest 与加载结果不一致。

## 四十六、动态区组织方式应作为设计取舍

PPT 提出了两个选择：

1. 一个动态区包含所有功能；
2. 一个功能对应一个独立动态区。

论文中可以将其规范成：

| 方案 | 优点 | 缺点 |
|---|---|---|
| Monolithic PR region | 装载组合自由、资源利用可能较高 | 任一更新影响全部 AFU，难以共享在线更新 |
| Fixed independent slots | 每个 AFU 可独立更新，隔离和生命周期简单 | slot 尺寸造成内部碎片，跨 slot 布线受限 |
| Heterogeneous slot classes | 在灵活性与碎片之间折中 | shell、调度和 bitstream 仓库更复杂 |

对于 ReVirt 的多租户共享场景，推荐固定独立 slot 或少量异构 slot class。原因是一个 AFU 被多个 VF 共享时，需要能够仅隔离和更新对应 AFU，而不干扰其他 slot。

实验可使用不同大小的 AFU 请求分布，比较：

- bitstream 覆盖率；
- FPGA 资源内部碎片；
- 请求接受率；
- 平均装载时间；
- 更新影响的 VF 数；
- 并行可运行 AFU 数量。

## 四十七、SIOV 是演进方向而不是当前实现的别名

PPT 准确捕捉了 SR-IOV 的限制：VF 数量和 PCIe 配置能力相对固定，细粒度资源组合和过量分配不够灵活。SIOV 使用 PASID 粒度隔离和 ADI/VDEV/VDCM 模型，可支持更细粒度、可组合的设备资源以及直接路径和拦截路径分离。[Intel Scalable I/O Virtualization Specification](https://cdrdv2-public.intel.com/671403/intel-scalable-io-virtualization-technical-specification.pdf)

但当前 ReVirt 实现仍是硬件 Virtio SR-IOV VF，论文必须避免把以下概念混用：

- VF 不是 ADI；
- VF requester ID 隔离不等于 PASID 粒度隔离；
- PF 动态绑定 AFU 不等于已经实现 VDCM；
- AFU 上下文超额分配不等于完整 SIOV over-provisioning；
- 稳定 Virtio ABI 不自动提供跨代热迁移。

ReVirt 向 SIOV 演进时，可以把当前 Binding Context 映射成 ADI 类似的细粒度资源，并将硬件 command queue 作为 direct path、配置和迁移操作作为 intercepted path。这个对应关系适合放在 Future Work，而不是当前贡献。

## 四十八、FPGA 池化必须区分本地逻辑池与远程物理池

PPT 中的“池化”至少包含三种不同含义：

1. **单卡逻辑池：** 一张 FPGA 上的 AFU slot/context 形成资源池；
2. **节点内多卡池：** 同一服务器的多个 PCIe FPGA 由统一 agent 管理；
3. **跨节点远程池：** VM 通过网络、RDMA 或 CXL fabric 使用远端 FPGA。

当前五份专利主要支持第一种，并可通过云控制面扩展到第二种的资源调度。第三种会改变数据路径、故障模型、一致性和性能边界，不能作为当前零拷贝或低延迟结论的自然延伸。

如果未来研究远程池化，需要单独评估：

- 网络/RDMA 往返延迟；
- 数据本地性与跨节点流量；
- 远程失效和重连；
- 多租户网络隔离；
- AFU 与数据源的联合放置；
- 本地 PCIe、节点内 P2P 和远程访问的分层调度。

## 四十九、适合作为论文评估的云端应用

PPT 对 FPGA 优势的判断较合理：FPGA 更适合流式、在线、低延迟和可定制数据路径。论文不需要寻找覆盖所有领域的“杀手级应用”，而应选择能够验证架构机制的代表性工作负载。

推荐至少包含：

- **短任务共享：** 压缩或加密，用于验证多个 VF 共享同一 AFU 的利用率、公平性和尾延迟；
- **多 AFU 聚合：** 压缩→加密或图像变换→编码，用于验证一个 VF 聚合多个 AFU 和共享 DDR；
- **动态功能变化：** 在压缩、加密或图像 AFU 之间切换，用于验证部分重配置和稳定 VF；
- **异常事件：** ECC、watchdog 或 AFU fault 注入，用于验证额外 MSI-X 和错误隔离。

应用性能不是唯一目标。每个工作负载应分别服务于一个架构研究问题，避免论文变成几个无关联算法的性能展示。

## 五十、PPT 加入后的新增研究问题

### RQ20：云控制面的配置代价有多大

如果实现了节点 agent 或 Cyborg 集成，应拆分测量：

- 资源发现时间；
- placement 查询和选择时间；
- bitstream 获取、验证与下载时间；
- Binding Context 配置时间；
- VF 绑定和 VM 启动时间；
- 失败回滚和重新调度时间。

需要区分冷路径与热路径：AFU 已存在且有空闲 context 时只需绑定；AFU 不存在时需要部分重配置。两者的服务交付时间可能相差很大。

### RQ21：动态组合能否提高请求接受率

使用真实或合成的 Accelerator Profile 请求序列，比较：

- 单功能整卡分配；
- 静态一 VF–一 AFU；
- 动态一 VF–多 AFU；
- 多对多 VF–AFU 时空共享。

指标包括：

- 请求接受率；
- 平均资源利用率；
- VF、AFU context 和 DDR 碎片；
- 等待时间；
- 重配置次数及累计重配置时间；
- 每租户性能 SLO 达成率。

如果没有完整云集群，可以使用单机原型测得的性能和重配置参数进行 trace-driven simulation，但必须把仿真结论与实机结果分开报告。

### RQ22：统一接口是否改善服务可运维性

可以量化：

- 支持一个新 AFU 所需的 driver/runtime 修改行数；
- 相同 guest 镜像跨不同 FPGA shell/AFU 后端的复用；
- AFU 升级前后应用是否需要重新编译；
- 统一 manifest 能发现多少类不兼容配置；
- 错误 bitstream 被阻止的比例和验证开销。

## 五十一、PPT 加入后的论文叙事建议

PPT 适合重写论文的引言和动机，但不应把论文扩展成覆盖所有云层级的大而全系统。推荐叙事顺序：

```text
FPGA PaaS 需要稳定、可共享且可动态装配的加速功能
→ 整卡分配利用率低，厂商接口导致应用绑定
→ 传统 SR-IOV VF 固定，难以适应可重构和多功能 FPGA
→ ReVirt 在硬件 VF 中实现 Virtio，并建立多对多 VF–AFU Binding Context
→ 一个 VF 聚合多个 AFU，一个 AFU 分时服务多个 VF
→ PF 事务化管理部分重配置，VF 数据面保持直通
→ 实验验证性能、利用率、公平性、隔离和更新连续性
```

对应的论文定位可以写成：

> ReVirt 不是完整的云调度器，而是 FPGA PaaS 在计算节点上的虚拟加速器执行层；它向上提供稳定的 Virtio VF，向下管理可重构且可共享的 AFU。

若已经完成 Cyborg 或 Kubernetes DRA 集成，可以把控制面纳入系统实现；否则仅用一个轻量节点管理器展示端到端流程，把完整云编排留作未来工作。

## 五十二、是否拆分基础设施论文

PPT 实际上包含了另一篇潜在论文的范围。建议按实现完成度决定：

### 当前 ReVirt 论文

集中于：

- 硬件 Virtio SR-IOV；
- VF–AFU 多对多虚拟化；
- 时空共享和隔离调度；
- 动态重配置与中断；
- 节点内端到端原型。

### 后续 FPGA PaaS 基础设施论文

集中于：

- AFU artifact registry 和供应链；
- Accelerator Profile；
- OpenStack/Kubernetes 集成；
- 多卡/跨节点放置；
- 重配置感知调度；
- 请求接受率、负载均衡和容错；
- 跨代兼容与迁移。

如果当前没有云管理系统原型，把 PPT 全部并入 ReVirt 会削弱硬件论文的聚焦度。更稳妥的处理是：PPT 为 ReVirt 提供问题背景和系统上下文，同时作为后续基础设施论文的研究路线图。
