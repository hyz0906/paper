| **文档编号** | **REQ-INFRA-AI-2026**                             | **版本号** | **1.0**  |
| -------- | ------------------------------------------------- | ------- | -------- |
| **项目名称** | 基础设施智能运维大脑 (AIOps Engineering Brain)              | **优先级** | P0 (战略级) |
| **参考来源** | 工程系统转型报告, LangGraph架构, MCP协议, Rancher/Ceph/DC运维方案 | **状态**  | 待评审      |

## 1. 引言 (Introduction)

### 1.1 背景与目标

在超大规模操作系统（Android/鸿蒙）研发场景下，计算平台组负责维护海量的 Kubernetes 集群及异构算力资源。当前基础设施面临告警风暴、资源预留浪费及硬件故障不可预测三大痛点。

本项目旨在构建“智能化工程大脑”，从传统的自动化工厂升级为具备自感知、自修复、自优化能力的下一代基础设施平台。

### 1.2 范围界定 (Scope)

本需求文档仅涵盖**基础设施层（Infrastructure Layer）**，具体涉及：

- **对象：** Kubernetes 集群、物理服务器（CPU/内存/磁盘）、网络链路、存储系统（Ceph）。
    
- **场景：** 智能监控与告警降噪、硬件故障预测与自愈、资源弹性伸缩、自动化运维编排。
    

---

## 2. 现状与痛点分析 (As-Is Analysis)

基于《工程系统 AI 化转型分析与设计报告》，当前计算平台组面临以下核心挑战：

|**痛点分类**|**具体表现**|**业务影响**|
|---|---|---|
|**被动响应**|Prometheus 产生海量告警，信噪比极低，SRE 疲于应对，缺乏根因分析能力。|MTTR（平均修复时间）过长，甚至导致构建中断。|
|**资源浪费**|为应对突发构建高峰，往往过量预留资源，非高峰期算力闲置。|计算成本高昂，资源利用率低下。|
|**故障不可测**|磁盘/内存等硬件故障通常导致任务崩溃后才被发现，缺乏预测性维护。|影响研发效能，导致构建任务反复重试。|

---

## 3. 系统架构需求 (System Architecture)

系统需采用 **数据面 (Data Plane)** 与 **智能面 (Intelligence Plane)** 分离的架构，并基于 **LangGraph** 实现状态机管理。

### 3.1 核心组件设计

- **感知层 (Perception):** 集成 Prometheus、Loki、K8s Events 及物理层传感器（SMART、光模块电压）数据。
    
- **大脑层 (Brain):** 基于 LLM 的推理引擎，采用 **ReAct (Reasoning + Acting)** 模式。
    
    - **编排引擎:** 使用 **LangGraph** 构建有向有环图，支持循环推理、状态持久化和“人在环路” (HITL)。
        
    - **记忆模块:** 结合 RAG 技术，索引历史故障工单、厂商文档及运维知识库。
        
- **执行层 (Action):**
    
    - **接口标准:** 采用 **MCP (Model Context Protocol)** 标准化工具调用（如 kubectl, helm, ssh）。
        
    - **安全网关:** 所有写操作需经过权限校验，高危操作需 GitOps 审计或人工审批。
        

---

## 4. 功能需求详情 (Functional Requirements)

### 4.1 模块一：智能监控与根因分析 (Intelligent Monitoring & RCA)

**目标：** 解决告警风暴，实现从“现象”到“根因”的自动推导。

- **FR-1.1 异常检测 (Anomaly Detection):**
    
    - 系统应采集 CPU、内存、IO、网络流量等时序数据，使用 **LSTM 或 Prophet 模型** 训练基线。
        
    - **需求细化：** 支持基于 **Reinforcement Learning (RL)** 的自适应采样，在网络平静期低频采样，异常时自动切换高频 INT (In-band Network Telemetry) 采集。
        
- **FR-1.2 智能告警降噪 (Alert Noise Reduction):**
    
    - 利用聚类算法（如 **DBSCAN**）将同一时间窗内的相关告警（如网络抖动引发的节点超时）聚合为单一“事件”。
        
    - **拓扑关联:** 结合 K8s 拓扑与物理网络拓扑（知识图谱），识别告警传播链（如存储备份导致的“嘈杂邻居”效应）。
        
- **FR-1.3 根因分析 Agent (RCA Agent):**
    
    - 基于 **LangGraph** 的循环推理诊断：观察 -> 假设 -> 验证。
        
    - **能力要求:** 能够自主查询 Logs (Loki)、Metrics (Prometheus) 和 K8s Events，自动生成故障拓扑图并指出 Root Cause。
        

### 4.2 模块二：预测性维护与硬件自愈 (Predictive Maintenance)

**目标：** 在硬件故障发生前进行隔离，实现“无感替换”。

- **FR-2.1 硬件寿命预测:**
    
    - **磁盘/内存:** 结合 SMART 数据与系统日志，预测磁盘与内存条故障概率。
        
    - **光链路:** 通过读取 DDM 数据（接收光功率、电压），利用双指数平滑算法预测光模块失效。
        
- **FR-2.2 自动隔离与迁移 (Auto-Remediation):**
    
    - 当预测到硬件故障风险 > 阈值（如 90%）时，触发自动维护流程。
        
    - **动作序列:** 自动 Cordon 节点 -> 驱逐 Pod (Evict) -> 触发工单通知数据中心现场更换。
        
    - **OSD 自愈:** 针对 Ceph OSD 进程崩溃，自动提取堆栈信息，匹配已知 Bug 库，若为元数据损坏则自动触发 `ceph-bluestore-tool repair`。
        

### 4.3 模块三：智能负载预测与弹性伸缩 (Intelligent Auto-Scaling)

**目标：** 解决冷启动延迟，提升资源利用率。

- **FR-3.1 负载预测:**
    
    - 基于历史构建任务波峰波谷数据，预测未来 1-4 小时的算力需求。
        
- **FR-3.2 抢占式扩缩容:**
    
    - 提前 10 分钟自动扩容 K8s Node 资源池，消除冷启动延迟。
        
    - 在非工作时间自动缩容闲置资源，但需结合业务 SLA 确保最低可用度。
        

### 4.4 模块四：自动化运维编排 (Agentic Orchestration)

**目标：** 处理复杂的跨集群运维任务。

- **FR-4.1 动态上下文管理:**
    
    - Agent 需具备通过 Rancher API 动态生成 **Just-in-Time Kubeconfig** 的能力，实现跨集群管理。
        
    - 支持处理 Rancher 下游集群 Agent 断连（Disconnected）场景，自动切换至 SSH 通道进行带外修复。
        
- **FR-4.2 证书与配置管理:**
    
    - 自动巡检 RKE2/K8s 证书有效期，发现过期风险自动执行轮换（Certificate Rotation）。
        
    - **GitOps 闭环:** 监控配置漂移（Drift），当检测到手动修改导致与 Git 仓库不一致时，自动发起修复 PR 或强制同步。
        

---

## 5. 非功能需求 (Non-Functional Requirements)

### 5.1 安全性 (Security) & 治理

- **NFR-1 最小权限原则:** Agent 身份应使用独立的 Service Account，且 Token 需动态轮换。
    
- **NFR-2 人在环路 (HITL):** 对于高危操作（如删除节点、升级内核），必须通过 LangGraph 的 `interrupt_before` 机制暂停，等待人工审批（Slack/钉钉卡片）。
    
- **NFR-3 GitOps 安全气闸:** Agent 对核心架构的变更不应直接操作 API，而应通过提交 Pull Request 的方式进行，由人工 Code Review 后合并生效。
    
- **NFR-4 防注入:** 系统需具备 Prompt Injection 防御机制，对输入日志进行清洗，防止恶意指令执行。
    

### 5.2 可靠性与可观测性

- **NFR-5 状态持久化:** 运维任务状态需通过 **PostgresCheckpointer** 持久化，确保 Agent 重启后能从断点恢复执行。
    
- **NFR-6 Agent 自我监控:** 需监控 Agent 的 Token 消耗、循环次数及工具调用成功率，防止 Agent 陷入死循环。
    

---

## 6. 实施路线图 (Implementation Roadmap)

建议分三个阶段实施，每阶段 6-8 个月。

|**阶段**|**模式**|**关键交付物**|**技术重点**|
|---|---|---|---|
|**Phase 1**|**Copilot (辅助模式)**|智能问答机器人、Prometheus 异常预警 Dashboard、构建耗时分析看板。|RAG 知识库构建、只读类 MCP 工具集成。|
|**Phase 2**|**Agent (协作模式)**|动态门禁系统、智能弹性资源池、低风险自动修复（如清理日志、重启 Pod）。|LangGraph 状态机、RCA 根因分析、预测性扩缩容。|
|**Phase 3**|**Autonomous (自治模式)**|硬件故障自愈系统、无人值守集群维护、全自动依赖治理。|跨系统多 Agent 协同、GitOps 闭环、数字孪生验证。|

---

## 7. 风险与缓解 (Risk Mitigation)

- **风险：** 幻觉（Hallucination）导致错误操作（如误删核心 Pod）。
    
    - **缓解：** 实施 **数字孪生/沙箱验证**，在下发配置前先在仿真环境中验证；强制实施“读写分离”和 HITL 审批流。
        
- **风险：** 告警风暴导致 Agent 上下文溢出。
    
    - **缓解：** 前置部署聚类算法（DBSCAN）进行事件压缩；采用 **Supervisor 模式** 分发任务给子 Agent 处理。
        

---

**下一步建议：** 立即启动 Phase 1 的 POC，选取“Prometheus 智能降噪与根因分析”作为切入点，验证 LangGraph 与 MCP 架构的可行性。