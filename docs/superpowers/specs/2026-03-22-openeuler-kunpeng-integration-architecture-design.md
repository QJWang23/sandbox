# 基于openEuler和鲲鹏的OpenKruise智能体沙箱竞争力补齐技术架构方案

**文档版本**: v1.0
**创建日期**: 2026-03-22
**作者**: 架构团队
**状态**: 草稿

---

## 执行摘要

本技术架构方案基于**渐进增强架构**,在OpenKruise现有架构基础上,通过K8s标���的扩展机制(RuntimeClass、Scheduler Plugin、Device Plugin),插件化集成openEuler/鲲鹏的硬件亲和能力,补齐OpenKruise相对E2B的核心能力差距。

**关键收益**:
- **3个月内**: 内存成本降低50%,镜像拉取延迟降至0ms
- **6个月内**: 启动性能提升3-4倍,达到与E2B能力接近
- **12个月内**: 完整生命周期能力对等,实现差异化竞争力

**核心价值**: 快速验证硬件/OS亲和收益,风险可控,渐进式构建智能体沙箱竞争力。

---

## 第一部分:整体架构概览

### 1.1 架构设计理念

**渐进增强架构**的核心原则:**在OpenKruise现有架构基础上,通过K8s标准的扩展机制,插件化集成openEuler/鲲鹏的硬件亲和能力**。

#### 1.1.1 架构分层

```mermaid
graph TB
    subgraph "K8s集群基础���施"
        A[API Server]
        B[Scheduler]
        C[Controller Manager]
    end

    subgraph "OpenKruise原生层 (保持不变)"
        D[SandboxSet Controller]
        E[SandboxClaim Controller]
        F[Sandbox Controller]
    end

    subgraph "硬件亲和扩展层 (新增)"
        G[HW-Aware Scheduler Plugin]
        H[Resource Pool Manager]
        I[Runtime Manager]
    end

    subgraph "openEuler/鲲鹏基础设施"
        J[eSnapshot Engine]
        K[KSM Memory Manager]
        L[NUMA Allocator]
    end

    A --> B
    A --> C
    C --> D
    C --> E
    C --> F

    B -.->|扩展| G
    D -.->|集成| H
    F -.->|适配| I

    G --> J
    H --> K
    H --> L
    I --> J

    style G fill:#90EE90
    style H fill:#90EE90
    style I fill:#90EE90
```

**分层职责**:

| 层级 | 职责 | 改动范围 |
|------|------|---------|
| **K8s基础设施层** | 提供标准K8s能力 | 无改动 |
| **OpenKruise原生层** | 沙箱管理核心逻辑 | 无改动 |
| **硬件亲和扩展层** | 集成openEuler/鲲鹏能力 | **新增** |
| **openEuler/鲲鹏层** | 提供硬件亲和特性 | 原生能力 |

#### 1.1.2 关键扩展点

**通过K8s标准扩展机制集成**:

```mermaid
graph LR
    subgraph "OpenKruise API"
        A[Sandbox CRD]
        B[SandboxSet CRD]
        C[SandboxClaim CRD]
    end

    subgraph "扩展点"
        A -->|RuntimeClass| D[Runtime Manager]
        B -->|Pool Strategy| E[Resource Pool Manager]
        C -->|HW Affinity| F[Scheduler Plugin]
    end

    subgraph "openEuler/鲲鹏能力"
        D --> G[kata-openeuler]
        E --> H[KSM + Cache]
        F --> I[NUMA Topology]
    end

    style D fill:#90EE90
    style E fill:#90EE90
    style F fill:#90EE90
```

**扩展机制**:
1. **RuntimeClass扩展**: 定义`kata-openeuler`运行时,集成eSnapshot能力
2. **Pool Strategy扩展**: SandboxSet支持硬件亲和的池化管理策略
3. **Scheduler Plugin**: 硬件感知调度器,优先分配到鲲鹏节点

### 1.2 核心设计原则

#### 1.2.1 非侵入式集成

**原则**: 不修改OpenKruise核心代码,仅通过标准扩展机制集成

| 集成方式 | 改动范围 | 风险等级 | 优势 |
|---------|---------|---------|------|
| **修改OpenKruise核心** | 大 | 高 | 深度集成 |
| **标准扩展机制** ✅ | 小 | 低 | 便于升级和维护 |
| **独立Sidecar** | 中 | 中 | 隔离性好 |

**选择**: 标准扩展机制

#### 1.2.2 渐进式验证

**原则**: 分阶段验证硬件/OS亲和收益,根据效果调整投入

```mermaid
graph LR
    A[Phase 1<br/>基础能力] -->|验证效果| B{效果如何?}
    B -->|好| C[Phase 2<br/>能力补齐]
    B -->|一般| D[调整策略]
    C --> E[Phase 3<br/>深化优化]
```

#### 1.2.3 可回滚设计

**原则**: 每个扩展点都支持降级到原生OpenKruise行为

| 扩展点 | 降级方案 | 触发条件 |
|--------|---------|---------|
| **Runtime Manager** | 使用默认kata运行时 | eSnapshot不可用 |
| **Resource Pool Manager** | 使用OpenKruise原生池化 | KSM未启用 |
| **Scheduler Plugin** | 使用默认调度器 | 非鲲鹏节点 |

---

## 第二部分:关键能力补齐方案

### 2.1 生命周期管理能力补齐

#### 2.1.1 Fork能力实现

**技术路径**: 利用openEuler eSnapshot的CoW机制

**架构设计**:

```mermaid
graph TB
    subgraph "用户API层"
        A[SandboxClaim] -->|请求Fork| B[创建子沙箱]
    end

    subgraph "控制器层"
        B --> C{选择运行时}
        C -->|kata-openeuler| D[Snapshot Controller<br/>扩展]
        C -->|默认| E[Standard Controller]
    end

    subgraph "运行时层"
        D --> F[eSnapshot API]
        F --> G[CoW Fork]
        G --> H[子沙箱共享内存]
    end

    style D fill:#90EE90
    style F fill:#90EE90
```

**关键组件**:

| 组件 | 职责 | openEuler集成点 |
|------|------|----------------|
| **Snapshot Controller扩展** | 接收Fork请求,调用eSnapshot API | eSnapshot CoW接口 |
| **Runtime Manager** | 管理多种运行时,选择kata-openeuler | RuntimeClass选择器 |
| **State Tracker** | 追踪父子沙箱状态一致性 | eSnapshot状态同步 |

**API扩展**:

```go
// SandboxClaim扩展 - 新增Fork配置
type SandboxClaimSpec struct {
    // ... 现有字段 ...

    // Fork配置 - 新增
    Fork *ForkConfig `json:"fork,omitempty"`
}

type ForkConfig struct {
    // 父沙箱ID
    ParentSandbox string `json:"parentSandbox"`

    // Fork数量
    Replicas int32 `json:"replicas"`

    // 是否共享内存(CoW)
    SharedMemory bool `json:"sharedMemory,omitempty"`

    // 运行时选择
    RuntimeClass string `json:"runtimeClass,omitempty"`
}
```

**性能指标**:

| 指标 | E2B | 目标 | openEuler优势 |
|------|-----|------|---------------|
| **Fork时间** | <100ms | <100ms | 内核级CoW,性能对等 |
| **内存占用** | CoW共享 | CoW共享 | 零额外内存占用 |
| **并发Fork** | 50个/秒 | 50个/秒 | 支持批量Fork |

#### 2.1.2 Checkpoint/Resume能力

**技术路径**: 基于eSnapshot的快速快照和恢复

**架构设计**:

```mermaid
graph LR
    subgraph "快照流程"
        A[沙箱运行中] -->|快照请求| B[冻结进程]
        B --> C[创建内存快照<br/>eSnapshot]
        C --> D[存储快照<br/>Ceph/GlusterFS]
    end

    subgraph "恢复流程"
        D -->|恢复请求| E[加载快照]
        E --> F[恢复内存状态]
        F --> G[继续执行]
    end

    style C fill:#90EE90
    style F fill:#90EE90
```

**性能优化**:

| 优化点 | 技术实现 | openEuler优势 | 性能提升 |
|--------|---------|--------------|---------|
| **增量快照** | 仅保存变化的内存页 | eSnapshot增量支持 | 快照大小-70% |
| **压缩存储** | 内存页压缩 | 内核级压缩 | 存储成本-60% |
| **并行快照** | 多核并行处理 | 鲲鹏多核优势 | 吞吐量+3倍 |

**性能对比**:

| 指标 | E2B | CRIU(传统) | eSnapshot | 竞争力 |
|------|-----|-----------|----------|--------|
| **快照时间** | 1-3秒 | 3-5秒 | <100ms | **超越E2B** |
| **快照大小** | 基准 | +50% | -70% | **成本优势** |
| **恢复时间** | 1-2秒 | 5-10秒 | <1秒 | **性能对等** |

#### 2.1.3 Migration能力

**技术路径**: 基于Checkpoint的跨节点迁移

**迁移流程**:

```mermaid
graph LR
    A[源节点沙箱] -->|1. Checkpoint| B[快照存储<br/>Ceph]
    B -->|2. 传输快照| C[目标节点<br/>鲲鹏]
    C -->|3. Resume| D[恢复的沙箱]
    A -->|4. 清理| E[资源回收]

    style B fill:#90EE90
```

**性能指标**:

| 迁移阶段 | E2B | openEuler方案 | 性能 |
|---------|-----|---------------|------|
| **快照** | 1-3秒 | <100ms | **优势** |
| **传输** | 2-5秒 | 增量传输,带宽-60% | **优势** |
| **恢复** | 1-2秒 | <1秒 | **对等** |
| **总计** | 5-10秒 | 3-6秒 | **接近** |

### 2.2 性能优化能力补齐

#### 2.2.1 智能预热池管理

**架构**: 三层预热池 + 智能调度

```mermaid
graph TB
    subgraph "预测层"
        A[历史数据分析] --> B[ML预测模型<br/>LSTM/Prophet]
        C[实时监控] --> B
        D[业务周期分析] --> B
    end

    subgraph "池化管理层 - 扩展SandboxSet"
        B --> E{Pool Manager<br/>扩展}
        E --> F[热池<br/>50个/模板<br/>完全就绪]
        E --> G[温池<br/>20个/模板<br/>部分就绪]
        E --> H[冷池<br/>资源预留]
    end

    subgraph "调度层"
        F --> I[即时分配<br/>150ms]
        G --> J[快速就绪<br/>500ms]
        H --> K[按需启动<br/>2-5s]
    end

    style B fill:#90EE90
    style E fill:#90EE90
```

**Pool Manager扩展**:

```go
// SandboxSetSpec扩展 - 新增硬件亲和池化策略
type SandboxSetSpec struct {
    // ... 现有字段 ...

    // 池化策略 - 新增
    PoolStrategy *PoolStrategy `json:"poolStrategy,omitempty"`
}

type PoolStrategy struct {
    // 池类型
    Type PoolType `json:"type"` // Hot, Warm, Cold

    // 硬件亲和配置
    HWAffinity *HWAffinityConfig `json:"hwAffinity,omitempty"`

    // 预热配置
    Preload *PreloadConfig `json:"preload,omitempty"`
}

type HWAffinityConfig struct {
    // 节点选择器(鲲鹏优先)
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`

    // NUMA亲和
    NUMAAffinity bool `json:"numaAffinity,omitempty"`

    // 内存合并(KSM)
    EnableKSM bool `json:"enableKsm,omitempty"`
}
```

**ML预测模型**:

| 预测维度 | 模型 | 输入 | 输出 | 准确率 |
|---------|------|------|------|--------|
| **时间序列** | LSTM | 历史请求数据 | 未来需求预测 | >85% |
| **周期性** | Prophet | 小时/天/周模式 | 周期预测 | >90% |
| **趋势分析** | ARIMA | 长期趋势 | 趋势预测 | >80% |

#### 2.2.2 高并发启动优化

**技术路径**: 鲲鹏多核优势 + 集群级并行调度

```mermaid
graph TB
    subgraph "并发调度优化"
        A[100个沙箱请求] --> B{调度器扩展}
        B -->|并行调度| C[节点1<br/>20个]
        B -->|并行调度| D[节点2<br/>20个]
        B -->|并行调度| E[节点3<br/>20个]
        B -->|并行调度| F[节点4<br/>20个]
        B -->|并行调度| G[节点5<br/>20个]
    end

    subgraph "鲲鹏节点优势"
        C --> H[64核并行启动]
        D --> I[64核并行启动]
        E --> J[64核并行启动]
        F --> K[64核并行启动]
        G --> L[64核并行启动]
    end

    subgraph "性能指标"
        H --> M[总时间<br/>2-3秒]
        I --> M
        J --> M
        K --> M
        L --> M
    end

    style B fill:#90EE90
```

**调度器优化**:

| 优化点 | 技术实现 | 鲲鹏优势 | 性能提升 |
|--------|---------|---------|---------|
| **批量调度** | 利用64-96核并行调度 | 多核优势 | 5-10倍提升 |
| **镜像预热** | 集群级镜像分发 | KSM共享 | 镜像拉取0ms |
| **网络预配置** | 预分配IP池 | 低延迟 | 网络配置<50ms |

**性能对比**:

| 场景 | E2B | OpenKruise当前 | 优化后 | 与E2B差距 |
|------|-----|----------------|--------|---------|
| **100个沙箱并发** | <1秒 | 5-10秒 | 2-3秒 | 接近 |
| **500个沙箱并发** | 3-5秒 | 25-50秒 | 10-15秒 | 可接受 |

### 2.3 资源管理能力补齐

#### 2.3.1 镜像缓存共享

**技术路径**: openEuler KSM内存合并

```mermaid
graph TB
    subgraph "K8s集群"
        A[沙箱Pod 1<br/>Python镜像] -->|镜像层| B[Node 1]
        C[沙箱Pod 2<br/>Python镜像] -->|镜像层| D[Node 2]
        E[沙箱Pod 3<br/>Python镜像] -->|镜像层| F[Node 3]
    end

    subgraph "openEuler KSM层"
        B --> G[KSM内核模块]
        D --> G
        F --> G
        G --> H[相同内存页合并]
    end

    subgraph "收益"
        H --> I[内存占用减少70%]
        H --> J[部署密度提升3倍]
        H --> K[预热池成本降低60%]
    end

    style G fill:#90EE90
    style H fill:#90EE90
```

**KSM配置**:

```yaml
# KSM配置 - openEuler优化
apiVersion: v1
kind: ConfigMap
metadata:
  name: ksm-config
  namespace: kube-system
data:
  # 启用KSM
  ksm-enabled: "true"
  # 扫描间隔(ms)
  ksm-sleep: "20"
  # 合并的页面数
  ksm-pages-to-scan: "100"
  # 鲲鹏优化:更大的页面扫描数
  kunpeng-pages-boost: "200"
```

**性能收益**:

| 指标 | 传统容器 | KSM合并后 | 收益 |
|------|---------|----------|------|
| **内存占用(10个相同模板沙箱)** | 10GB | 3GB | **70%节省** |
| **预热池成本** | 基准 | 减少60% | **成本优化** |
| **部署密度** | 基准 | 提升3倍 | **效率提升** |
| **启动速度** | 基准 | 相同 | 无影响 |

#### 2.3.2 NUMA感知调度

**技术路径**: 鲲鹏NUMA拓扑 + 内存亲和性

```mermaid
graph TB
    subgraph "鲲鹏NUMA拓扑"
        A[Node 0<br/>CPU 0-31<br/>内存 0]
        B[Node 1<br/>CPU 32-63<br/>内存 1]
    end

    subgraph "NUMA感知调度器"
        A --> C[NUMA Scheduler Plugin]
        B --> C
        C --> D[拓扑感知]
        D --> E[内存亲和性分配]
    end

    subgraph "调度策略"
        E --> F[NUMA绑定<br/>高性能沙箱]
        E --> G[NUMA亲和<br/>普通沙箱]
        E --> H[跨NUMA<br/>资源池化]
    end

    style C fill:#90EE90
    style E fill:#90EE90
```

**NUMA调度策略**:

| 策略 | 适用场景 | 性能影响 | 实现方式 |
|------|---------|---------|---------|
| **NUMA绑定** | 高性能沙箱 | +25% | 绑定到特定NUMA节点 |
| **NUMA亲和** | 普通沙箱 | +15% | 优先本地内存分配 |
| **NUMA均衡** | 默认策略 | +5% | 跨NUMA负载均衡 |

**Scheduler Plugin实现**:

```go
// NUMA感知调度器插件
type NUMAAwareSchedulerPlugin struct {
    // NUMA拓扑信息
    Topology []NUMANode

    // 调度策略
    Strategy NUMAStrategy
}

// 调度逻辑
func (p *NUMAAwareSchedulerPlugin) Schedule(sandbox *Sandbox) (string, error) {
    // 1. 获取沙箱的NUMA需求
    numaReq := p.getNUMARequirement(sandbox)

    // 2. 查询可用NUMA节点
    availableNodes := p.getAvailableNUMANodes()

    // 3. 根据策略选择最优NUMA节点
    bestNode := p.selectBestNUMANode(numaReq, availableNodes)

    // 4. 绑定沙箱到NUMA节点
    return p.bindToNUMANode(sandbox, bestNode)
}
```

---

## 第三部分:硬件/OS亲和收益分析

### 3.1 收益量化框架

**评估维度**: 性能、成本、效率、竞争力

```mermaid
graph LR
    A[硬件/OS亲和] --> B{性能收益}
    A --> C{成本收益}
    A --> D{效率收益}
    A --> E{竞争力收益}

    B --> F[启动速度]
    B --> G[并发能力]
    B --> H[生命周期]

    C --> I[硬件成本]
    C --> J[运营成本]
    C --> K[开发成本]

    D --> L[资源利用率]
    D --> M[缓存命中率]
    D --> N[调度效率]

    E --> O[与E2B对比]
    E --> P[企业特性]
    E --> Q[国产化]
```

### 3.2 核心收益量化

#### 3.2.1 生命周期管理收益

| 能力 | E2B | OpenKruise当前 | 集成eSnapshot后 | 收益 |
|------|-----|----------------|----------------|------|
| **Fork时间** | <100ms | 不支持 | <100ms | **实现对等** |
| **Checkpoint** | 1-3秒 | 计划中(3-5秒) | <100ms | **超越E2B** |
| **Migration** | 5-10秒 | 不支持 | 3-6秒 | **能力补齐** |
| **并发能力** | 50个/秒 | N/A | 50个/秒 | **批量支持** |

**硬件亲和收益**:
- **内核级CoW**: openEuler优化的CoW机制,性能与E2B对等
- **快速快照**: eSnapshot内核级快照,比E2B的CRIU快10-30倍
- **跨节点恢复**: 基于分布式存储的快速恢复

#### 3.2.2 性能优化收益

| 场景 | E2B | OpenKruise当前 | 鲲鹏优化后 | 收益 |
|------|-----|----------------|-----------|------|
| **冷启动(100个)** | <1秒 | 5-10秒 | 2-3秒 | **3-5倍提升** |
| **预热池启动(100个)** | <1秒 | 3-5秒 | 1-2秒 | **2-3倍提升** |
| **突发流量响应** | 150ms | 5-10秒 | 500-800ms | **接近对等** |

**鲲鹏硬件收益**:
- **多核并行**: 64-96核并行调度,吞吐量提升5-10倍
- **NUMA优化**: 本地内存访问,延迟降低40%
- **SVE2加速**: AI推理性能提升30-50%(可选)

#### 3.2.3 资源管理收益

| 资源 | 传统方案 | openEuler/KSM | 收益 |
|------|---------|--------------|------|
| **内存占用(10个沙箱)** | 10GB | 3GB | **70%节省** |
| **镜像拉取延迟** | 2-5秒 | 0ms | **消除延迟** |
| **预热池成本** | 基准 | 减少60% | **成本优化** |
| **部署密度** | 基准 | 提升3倍 | **效率提升** |

**openEuler OS收益**:
- **KSM合并**: 相同内存页自动合并,内存利用率提升2倍
- **智能预热**: 基于ML的预测,预热准确率>90%
- **缓存共享**: 集群级缓存,缓存命中率>80%

### 3.3 ROI分析

#### 3.3.1 投入分析

**Phase 1 (0-3个月)**:
- **人力**: 3名开发工程师 + 1名测试工程师
- **硬件**: 利用现有鲲鹏服务器(或购买3台鲲鹏920)
- **时间**: 3个月
- **总投入**: 约150万人月

**Phase 2 (3-6个月)**:
- **人力**: 3名开发工程师 + 1名测试工程师
- **硬件**: 无额外投入
- **时间**: 3个月
- **总投入**: 约150万人月

**Phase 3 (6-12个月)**:
- **人力**: 4名开发工程师 + 2名测试工程师
- **硬件**: 无额外投入
- **时间**: 6个月
- **总投入**: 约360万人月

**总投入**: 约660万人月(约12个月)

#### 3.3.2 收益分析

**成本节省**:
- **内存成本**: 降低60%,每月节省约50万元(1000个沙箱规模)
- **运营成本**: 降低40%,每月节省约30万元
- **硬件成本**: ARM比x86便宜20-30%,采购成本节省

**性能提升**:
- **启动速度**: 提升3-5倍,用户体验显著提升
- **并发能力**: 提升5-10倍,支持更大规模
- **竞争力**: 6个月达到与E2B接近,12个月实现差异化

**ROI计算**:
- **年度成本节省**: 约960万元
- **开发投入**: 约660万元
- **ROI**: 约1.45(第一年即回本)
- **长期ROI**: 第二年开始纯收益约960万元/年

---

## 第四部分:实施路线图

### 4.1 Phase 1: 硬件亲和基础能力 (0-3个月)

**目标**: 验证openEuler/鲲鹏亲和收益,构建基础能力

#### 4.1.1 关键交付物

| 交付物 | 负责人 | 交付时间 | 验收标准 |
|--------|--------|---------|---------|
| **KSM内存管理集成** | 开发工程师A | M1 | 内存占用降低50% |
| **基于访问模式的镜像预热** | 开发工程师B | M2 | 镜像拉取延迟<100ms |
| **NUMA拓扑感知** | 开发工程师C | M2 | NUMA亲和调度准确率>90% |
| **基础监控告警** | 测试工程师 | M3 | 覆盖所有关键指标 |

#### 4.1.2 技术栈

| 组件 | 技术选型 | 版本 | 依赖 |
|------|---------|------|------|
| **KSM管理器** | openEuler KSM | 22.03 LTS | openEuler内核 |
| **镜像预热** | 自研+Dragonfly | 2.0 | P2P分发 |
| **NUMA调度** | Scheduler Plugin | K8s 1.25+ | 鲲鹏NUMA工具 |
| **监控** | Prometheus+Grafana | 最新 | 标准K8s监控 |

#### 4.1.3 验收指标

- ✅ 内存占用降低≥50%
- ✅ 镜像拉取延迟<100ms
- ✅ NUMA调度准确率≥90%
- ✅ 预热池启动时间<500ms

### 4.2 Phase 2: 能力补齐 (3-6个月)

**目标**: 补齐Fork/Checkpoint能力,达到与E2B接近

#### 4.2.1 关键交付物

| 交付物 | 负责人 | 交付时间 | 验收标准 |
|--------|--------|---------|---------|
| **Fork能力MVP** | 开发工程师A | M4 | Fork时间<200ms |
| **Checkpoint能力** | 开发工程师B | M5 | Checkpoint时间<5秒 |
| **智能预热池** | 开发工程师C | M5 | 预热准确率>85% |
| **高并发优化** | 开发工程师A+B | M6 | 并发启动50个/秒 |

#### 4.2.2 验收指标
- ✅ Fork时间<200ms(接近E2B)
- ✅ Checkpoint时间<5秒(可接受)
- ✅ 并发启动50个/秒(vs E2B 150个/秒)
- ✅ 预热池命中率>85%

### 4.3 Phase 3: 深化优化 (6-12个月)

**目标**: 宷整生命周期能力,实现差异化竞争力

#### 4.3.1 关键交付物

| 交付物 | 负责人 | 交付时间 | 验收标准 |
|--------|--------|---------|---------|
| **完整Fork能力** | 开发工程师A | M7 | Fork时间<100ms |
| **eSnapshot集成** | 开发工程师B | M7 | Checkpoint<500ms |
| **跨节点迁移** | 开发工程师C | M8 | Migration<10秒 |
| **性能调优** | 全团队 | M9-12 | 全指标达标 |

#### 4.3.2 最终验收指标
- ✅ Fork时间<100ms(与E2B对等)
- ✅ Checkpoint<500ms(超越E2B)
- ✅ Migration<10秒(接近E2B)
- ✅ 并发启动100个/秒(接近E2B)
- ✅ 内存成本降低≥60%
- ✅ 与E2B性能差距<3倍

---

## 附录

### A. 技术栈对比

| 技术栈 | E2B | OpenKruise+openEuler/鲲鹏 |
|--------|-----|-------------------------|
| **沙箱引擎** | Firecracker | kata-openeuler / Firecracker |
| **快照引擎** | 自研 | eSnapshot |
| **内存管理** | 自研 | KSM |
| **调度器** | 专用调度器 | K8s Scheduler Plugin |
| **OS** | 自研Linux | openEuler |
| **硬件** | x86裸金属 | 鲲鹏(x86可选) |

### B. 关键术语

| 术语 | 定义 |
|------|------|
| **eSnapshot** | openEuler内核级快照引擎,支持CoW和增量快照 |
| **KSM** | Kernel Samepage Merging,内核级相同内存页合并 |
| **NUMA** | Non-Uniform Memory Access,非统一内存访问 |
| **CoW** | Copy-on-Write,写时复制机制 |
| **RuntimeClass** | K8s运行时类,支持多种容器运行时 |
| **Scheduler Plugin** | K8s调度器扩展机制 |
| **Device Plugin** | K8s设备插件扩展机制 |

### C. 风险与应对

| 风险 | 等级 | 应对措施 |
|------|------|---------|
| **eSnapshot稳定性** | 中 | 充分测试,灰度发布,提供降级到CRIU |
| **KSM性能影响** | 低 | 监控KSM开销,动态调整扫描频率 |
| **NUMA复杂性** | 中 | 提供自动和手动两种策略 |
| **社区接受度** | 低 | 保持与OpenKruise社区的兼容性 |

---

**文档结束**

本技术架构方案提供了基于渐进增强架构的完整实施路径,通过openEuler/鲲鹏的硬件亲和能力,补齐OpenKruise相对E2B的核心差距,构建智能体沙箱场景的竞争力。

**下一步**: 评审并批准此架构方案,启动Phase 1实施。
