---
name: openkruise-lightweight-scheduler-design
description: 基于 OpenKruise 实现轻量级调度器的设计方案，包含非侵入式集成路径、内核/硬件协同优化机制、预热池协同策略
created: 2026-03-27
audience: 内部技术决策
status: approved
---

# 基于 OpenKruise 的轻量级调度器设计与内核/硬件协同优化方案

## 1. 概述

### 1.1 背景

基于《沙箱高并发调度：无状态轻量级调度与 K8s 传统调度的核心技术差异分析》（2026-03-27），本设计文档解决两个关键问题：

1. **如何基于 OpenKruise 实现轻量级调度器**，以及是否涉及侵入式修改
2. **如何实现内核/硬件协同优化**，以及预热池管理与调度器的协同机制

### 1.2 设计目标

| 目标 | 描述 |
|------|------|
| **非侵入式集成** | 不修改 OpenKruise 核心代码，仅通过扩展点集成 |
| **硬件感知调度** | 支持 NUMA 亲和性、NVLink 拓扑、鲲鹏 SVE 等硬件优化 |
| **预热池协同** | 硬件感知预热，池管理与调度深度协同 |
| **性能目标** | Fast Path 端到端延迟 <50ms，命中率 >95% |

---

## 2. OpenKruise 集成架构

### 2.1 OpenKruise 现有架构分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenKruise 架构                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Kruise Manager (Manager)                                │    │
│  │  ├─ Webhook (Admission)                                  │    │
│  │  └─ Controllers                                          │    │
│  │      ├─ CloneSet Controller                              │    │
│  │      ├─ Advanced StatefulSet Controller                  │    │
│  │      ├─ SidecarSet Controller                            │    │
│  │      ├─ PodUnavailableBudget Controller                  │    │
│  │      └─ sandbox-manager (E2B protocol)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│                              ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Kruise Daemon (DaemonSet, 每个 Node 一个)               │    │
│  │  ├─ Pod Lifecycle Adapter                                │    │
│  │  ├─ Resource Distribution                                │    │
│  │  └─ Pre-download Image                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  依赖: K8s kube-scheduler 做调度决策                              │
└─────────────────────────────────────────────────────────────────┘
```

**核心约束：OpenKruise 本身不做调度，它依赖 K8s 原生 kube-scheduler。**

### 2.2 实现路径对比

| 路径 | 描述 | 侵入性 | 复杂度 | 推荐度 |
|------|------|--------|--------|--------|
| **A: Scheduler Plugin** | 在 kube-scheduler 中注入自定义插件 | 中（需部署配置） | 中 | ⭐⭐⭐⭐ |
| **B: Side Scheduler** | 独立调度器 + OpenKruise 只做生命周期管理 | 低（完全解耦） | 低 | ⭐⭐⭐⭐⭐ |
| **C: 改造 OpenKruise** | 在 sandbox-manager 中加入调度逻辑 | 高（侵入式） | 高 | ⭐⭐ |

### 2.3 推荐架构：Side Scheduler（非侵入式）

```
┌─────────────────────────────────────────────────────────────────┐
│                      整体架构（Side Scheduler）                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  用户请求入口 (SDK / API Gateway)                          │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                   │
│              ┌──────────────┴──────────────┐                    │
│              ▼                             ▼                    │
│  ┌───────────────────────┐   ┌───────────────────────────────┐  │
│  │  Fast Scheduler       │   │  K8s Scheduler                │  │
│  │  (轻量级调度器)         │   │  + Scheduler Plugin           │  │
│  │                       │   │  (池补充 + 特殊约束)            │  │
│  │  - 内存槽位索引        │   │                               │  │
│  │  - Best-Fit 分配       │   │  - 使用 K8s Scheduler         │  │
│  │  - 节点心跳收集        │   │    Framework 的扩展点          │  │
│  │  - 预热池状态感知       │   │  - 作为 K8s 调度的一部分运行    │  │
│  └───────────┬───────────┘   └───────────────┬───────────────┘  │
│              │                               │                  │
│              │      池补充请求                 │                  │
│              └───────────────────────────────┘                  │
│                             │                                   │
│                             ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  OpenKruise (不做调度决策，只做生命周期管理)                │   │
│  │                                                          │   │
│  │  sandbox-manager:                                        │   │
│  │  - Sandbox/SandboxSet CRD 控制器                         │   │
│  │  - 预热池声明式管理 (WarmPool CRD)                        │   │
│  │  - 沙箱生命周期操作 (Fork/Checkpoint/Resume/Pause)        │   │
│  │  - E2B 协议兼容                                          │   │
│  │                                                          │   │
│  │  Kruise Daemon (per-node):                               │   │
│  │  - 预热沙箱实例的本地管理                                  │   │
│  │  - 心跳上报 (向 Fast Scheduler)     ◀── 新增扩展          │   │
│  │  - 沙箱激活/注入用户代码             ◀── 新增扩展          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                             │                                   │
│                             ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  K8s 集群                                                 │   │
│  │  - Node A (Kruise Daemon + 预热池)                        │   │
│  │  - Node B (Kruise Daemon + 预热池)                        │   │
│  │  - Node C (Kruise Daemon + 预热池)                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 组件职责分离

| 组件 | 职责 | 是否修改 OpenKruise |
|------|------|-------------------|
| **Fast Scheduler** | 调度决策（内存操作，<1ms） | 否，独立部署 |
| **K8s Scheduler Plugin** | 池补充调度（K8s 原生路径） | 否，标准扩展点 |
| **OpenKruise sandbox-manager** | 预热池 CRD 控制器 | 否，已有能力 |
| **Kruise Daemon** | 心跳上报 + 沙箱激活 | **小改动**：增加心跳上报逻辑 |

### 2.5 对 OpenKruise 的最小改动

#### 改动 1: Kruise Daemon 心跳上报

**位置**: `kruise/daemon/heartbeat_reporter.go`（新增文件）

```go
package daemon

import (
    "encoding/json"
    "net"
    "time"
)

// Heartbeat 心跳数据结构
type Heartbeat struct {
    NodeID      string  `json:"node_id"`
    PoolType    string  `json:"pool_type"`
    FreeSlots   int     `json:"free_slots"`
    Prewarmed   int     `json:"prewarmed"`
    GpuUtil     float64 `json:"gpu_util"`
    CpuUtil     float64 `json:"cpu_util"`
    NumaInfo    []NUMAInfo `json:"numa_info,omitempty"`
    Timestamp   int64   `json:"timestamp"`
}

// NUMAInfo NUMA 节点信息
type NUMAInfo struct {
    ID        int `json:"id"`
    FreeGpus  int `json:"free_gpus"`
    FreeCpus  int `json:"free_cpus"`
    FreeSlots int `json:"free_slots"`
}

// HeartbeatReporter 心跳上报器
type HeartbeatReporter struct {
    schedulerEndpoint string
    nodeID            string
    poolManager       *PoolManager
    interval          time.Duration
    conn              *net.UDPConn
}

// NewHeartbeatReporter 创建心跳上报器
func NewHeartbeatReporter(schedulerEndpoint, nodeID string, poolManager *PoolManager) *HeartbeatReporter {
    return &HeartbeatReporter{
        schedulerEndpoint: schedulerEndpoint,
        nodeID:            nodeID,
        poolManager:       poolManager,
        interval:          3 * time.Second,
    }
}

// Run 启动心跳上报（独立 goroutine）
func (r *HeartbeatReporter) Run(stopCh <-chan struct{}) {
    // 解析调度器地址
    addr, err := net.ResolveUDPAddr("udp", r.schedulerEndpoint)
    if err != nil {
        log.Errorf("Failed to resolve scheduler address: %v", err)
        return
    }

    // 创建 UDP 连接
    conn, err := net.DialUDP("udp", nil, addr)
    if err != nil {
        log.Errorf("Failed to create UDP connection: %v", err)
        return
    }
    defer conn.Close()

    ticker := time.NewTicker(r.interval)
    defer ticker.Stop()

    for {
        select {
        case <-stopCh:
            return
        case <-ticker.C:
            r.sendHeartbeat(conn)
        }
    }
}

// sendHeartbeat 发送心跳（UDP，不等待响应）
func (r *HeartbeatReporter) sendHeartbeat(conn *net.UDPConn) {
    heartbeat := Heartbeat{
        NodeID:    r.nodeID,
        PoolType:  r.poolManager.GetPoolType(),
        FreeSlots: r.poolManager.GetFreeSlots(),
        Prewarmed: r.poolManager.GetPrewarmedCount(),
        GpuUtil:   r.poolManager.GetGpuUtilization(),
        CpuUtil:   r.poolManager.GetCpuUtilization(),
        NumaInfo:  r.poolManager.GetNUMAInfo(),
        Timestamp: time.Now().UnixMilli(),
    }

    data, err := json.Marshal(heartbeat)
    if err != nil {
        log.Errorf("Failed to marshal heartbeat: %v", err)
        return
    }

    // UDP 发送，不等待响应
    _, err = conn.Write(data)
    if err != nil {
        log.Errorf("Failed to send heartbeat: %v", err)
    }
}
```

**代码量**: ~100 行

#### 改动 2: Kruise Daemon 快速激活接口

**位置**: `kruise/daemon/quick_activate.go`（新增文件）

```go
package daemon

import (
    "context"
    "errors"
    "fmt"

    "google.golang.org/grpc"
    pb "kruise.io/api/sandbox/v1alpha1"
)

// QuickActivateServer 快速激活 gRPC 服务
type QuickActivateServer struct {
    poolManager *PoolManager
    pb.UnimplementedQuickActivateServer
}

// NewQuickActivateServer 创建快速激活服务
func NewQuickActivateServer(poolManager *PoolManager) *QuickActivateServer {
    return &QuickActivateServer{
        poolManager: poolManager,
    }
}

// Activate 快速激活沙箱
func (s *QuickActivateServer) Activate(ctx context.Context, req *pb.ActivateRequest) (*pb.ActivateResponse, error) {
    // 1. 从预热池取出实例
    sandbox, err := s.poolManager.TakeOne(ctx, req.GetNumaNode(), req.GetGpuGroup())
    if err != nil {
        return nil, fmt.Errorf("no prewarmed sandbox available: %w", err)
    }

    // 2. 注入用户代码/配置
    if req.GetUserCode() != "" {
        if err := sandbox.InjectUserCode(ctx, req.GetUserCode()); err != nil {
            s.poolManager.ReturnOne(sandbox) // 归还到池中
            return nil, fmt.Errorf("failed to inject user code: %w", err)
        }
    }

    // 3. 注入环境变量
    if len(req.GetEnv()) > 0 {
        if err := sandbox.InjectEnv(ctx, req.GetEnv()); err != nil {
            s.poolManager.ReturnOne(sandbox)
            return nil, fmt.Errorf("failed to inject env: %w", err)
        }
    }

    // 4. 激活（从 Paused 状态恢复）
    if err := sandbox.Activate(ctx); err != nil {
        s.poolManager.ReturnOne(sandbox)
        return nil, fmt.Errorf("failed to activate sandbox: %w", err)
    }

    // 5. 返回沙箱连接信息
    return &pb.ActivateResponse{
        SandboxId: sandbox.ID,
        Endpoint:  sandbox.Endpoint,
        Runtime:   sandbox.Runtime,
    }, nil
}

// Register 注册 gRPC 服务
func (s *QuickActivateServer) Register(server *grpc.Server) {
    pb.RegisterQuickActivateServer(server, s)
}
```

**代码量**: ~80 行

#### 改动 3: Daemon 入口集成

**位置**: `kruise/daemon/daemon.go`（修改现有文件）

```go
// 在 NewDaemon 函数中添加
func NewDaemon(cfg *config.Config) (*Daemon, error) {
    // ... 现有代码 ...

    // 新增: 初始化心跳上报器
    heartbeatReporter := NewHeartbeatReporter(
        cfg.SchedulerEndpoint,  // 从配置读取
        cfg.NodeName,
        poolManager,
    )

    // 新增: 初始化快速激活服务
    activateServer := NewQuickActivateServer(poolManager)

    return &Daemon{
        // ... 现有字段 ...
        heartbeatReporter: heartbeatReporter,
        activateServer:    activateServer,
    }, nil
}

// 在 Run 函数中添加
func (d *Daemon) Run(ctx context.Context) error {
    // ... 现有代码 ...

    // 新增: 启动心跳上报
    go d.heartbeatReporter.Run(ctx.Done())

    // 新增: 注册快速激活 gRPC 服务
    d.grpcServer.RegisterService(activateServer)

    // ... 现有代码 ...
}
```

**代码量**: ~20 行修改

#### 总改动量

| 改动 | 文件 | 行数 | 类型 |
|------|------|------|------|
| 心跳上报器 | `heartbeat_reporter.go` | ~100 | 新增 |
| 快速激活接口 | `quick_activate.go` | ~80 | 新增 |
| Daemon 集成 | `daemon.go` | ~20 | 修改 |
| 配置扩展 | `config.go` | ~10 | 修改 |
| **总计** | | **~210** | |

**结论：非侵入式集成，仅需约 210 行代码扩展，不修改 OpenKruise 核心控制器。**

---

## 3. 内核/硬件协同优化设计

### 3.1 两层调度中的优化层次

```
┌─────────────────────────────────────────────────────────────────┐
│           两层调度 + 内核/硬件协同优化架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Layer 1: Fast Scheduler (轻量级)                          │  │
│  │                                                           │  │
│  │  扩展点: Scorer Plugin Interface (类似 Scheduler Plugin)    │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  type ScorerPlugin interface {                      │  │  │
│  │  │      Score(node *NodeInfo, req *SandboxRequest) int │  │  │
│  │  │      Name() string                                  │  │  │
│  │  │      Weight() float64                               │  │  │
│  │  │  }                                                  │  │  │
│  │  │                                                     │  │  │
│  │  │  内置插件:                                           │  │  │
│  │  │  - FreeSlotsScorer     (空闲槽位，默认权重 0.3)       │  │  │
│  │  │  - GpuTopologyScorer   (NVLink 拓扑，权重 0.25)      │  │  │
│  │  │  - NumaAffinityScorer  (NUMA 亲和性，权重 0.2)       │  │  │
│  │  │  - CacheLocalityScorer (LLC 缓存，权重 0.15)         │  │  │
│  │  │  - KunpengSveScorer    (鲲鹏 SVE，权重 0.1)          │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  打分公式:                                                 │  │
│  │  total_score = Σ(plugin.Score(node, req) * plugin.Weight()) │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Layer 2: K8s Scheduler + Plugins                         │  │
│  │                                                           │  │
│  │  标准 K8s Scheduler Plugin (用于池补充调度):                │  │
│  │  - GpuTopologyPlugin (自定义或 K8s 原生)                   │  │
│  │  - NumaNodePlugin                                         │  │
│  │  - DevicePlugin (GPU/NPU 设备感知)                         │  │
│  │  - CustomHardwarePlugin (自定义硬件感知)                    │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  硬件感知层 (Hardware Awareness Layer)                     │  │
│  │                                                           │  │
│  │  数据来源:                                                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │  │
│  │  │ Node Exporter│  │ DCGM Exporter│  │ Custom Agent  │     │  │
│  │  │ (CPU/内存)   │  │ (GPU 指标)   │  │ (NUMA/缓存)   │     │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │  │
│  │         │                 │                 │              │  │
│  │         └─────────────────┼─────────────────┘              │  │
│  │                           ▼                                │  │
│  │              ┌──────────────────────┐                      │  │
│  │              │ Hardware State Cache │                      │  │
│  │              │ (内存级，心跳同步)     │                      │  │
│  │              └──────────────────────┘                      │  │
│  │                           │                                │  │
│  │         ┌─────────────────┼─────────────────┐              │  │
│  │         ▼                 ▼                 ▼              │  │
│  │    Fast Scheduler   K8s Scheduler   Pool Manager           │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Scorer Plugin 接口定义

```go
package scheduler

import "context"

// ScorerPlugin 打分插件接口
type ScorerPlugin interface {
    // Name 返回插件名称
    Name() string

    // Weight 返回插件权重 (0.0-1.0)
    Weight() float64

    // Score 计算节点得分 (0-100)
    // node: 节点信息（包含硬件拓扑）
    // req: 沙箱请求
    Score(ctx context.Context, node *NodeInfo, req *SandboxRequest) (int, error)
}

// NodeInfo 节点信息（包含硬件拓扑）
type NodeInfo struct {
    NodeID     string
    PoolType   string
    FreeSlots  int
    Prewarmed  int

    // 硬件拓扑信息
    HardwareInfo HardwareInfo
}

// HardwareInfo 硬件拓扑信息
type HardwareInfo struct {
    // NUMA 拓扑
    NumaNodes []NumaNodeInfo `json:"numa_nodes"`

    // GPU 拓扑
    GpuTopology GpuTopologyInfo `json:"gpu_topology"`

    // 缓存信息
    CacheInfo CacheInfo `json:"cache_info"`

    // CPU 特性
    CpuFeatures CpuFeatures `json:"cpu_features"`
}

// NumaNodeInfo NUMA 节点信息
type NumaNodeInfo struct {
    ID        int `json:"id"`
    FreeGpus  int `json:"free_gpus"`
    FreeCpus  int `json:"free_cpus"`
    FreeSlots int `json:"free_slots"`
    MemoryMB  int `json:"memory_mb"`
}

// GpuTopologyInfo GPU 拓扑信息
type GpuTopologyInfo struct {
    GpuCount    int           `json:"gpu_count"`
    GpuModel    string        `json:"gpu_model"`
    NvlinkGroups []NvlinkGroup `json:"nvlink_groups"`
}

// NvlinkGroup NVLink 连接组
type NvlinkGroup struct {
    GpuIDs     []int  `json:"gpu_ids"`
    Bandwidth  string `json:"bandwidth"`
    FullyConnected bool `json:"fully_connected"`
}

// CacheInfo 缓存信息
type CacheInfo struct {
    LLCSizeMB    int  `json:"llc_size_mb"`
    NumaShared   bool `json:"numa_shared"`
    CacheLineSize int  `json:"cache_line_size"`
}

// CpuFeatures CPU 特性
type CpuFeatures struct {
    Architecture  string `json:"architecture"`  // x86_64, aarch64
    Vendor        string `json:"vendor"`        // Intel, AMD, ARM, Huawei
    VectorLength  int    `json:"vector_length"` // SVE 向量长度
    SupportsSVE   bool   `json:"supports_sve"`
    SupportsSVE2  bool   `json:"supports_sve2"`
    SupportsSME   bool   `json:"supports_sme"`  // 矩阵扩展
    SupportsAMX   bool   `json:"supports_amx"`  // Intel AMX
}

// SandboxRequest 沙箱请求
type SandboxRequest struct {
    PoolType    string            `json:"pool_type"`
    SlotsNeeded int               `json:"slots_needed"`
    UserID      string            `json:"user_id"`

    // 硬件约束
    HardwareConstraints HardwareConstraints `json:"hardware_constraints"`
}

// HardwareConstraints 硬件约束
type HardwareConstraints struct {
    GpuCount             int    `json:"gpu_count,omitempty"`
    GpuModel             string `json:"gpu_model,omitempty"`
    RequireNVLink        bool   `json:"require_nvlink,omitempty"`
    RequireNUMAAffinity  bool   `json:"require_numa_affinity,omitempty"`
    MinVectorLength      int    `json:"min_vector_length,omitempty"`
    RequireSVE           bool   `json:"require_sve,omitempty"`
    PreferCacheLocality  bool   `json:"prefer_cache_locality,omitempty"`
}
```

### 3.3 内置 Scorer Plugin 实现

#### 3.3.1 NVLink 拓扑感知打分

```go
package plugins

import (
    "context"
    "fmt"

    "scheduler"
)

// GpuTopologyScorer NVLink 拓扑感知打分插件
type GpuTopologyScorer struct {
    topologyCache *TopologyCache
    weight        float64
}

func NewGpuTopologyScorer(cache *TopologyCache) *GpuTopologyScorer {
    return &GpuTopologyScorer{
        topologyCache: cache,
        weight:        0.25,
    }
}

func (s *GpuTopologyScorer) Name() string {
    return "GpuTopologyScorer"
}

func (s *GpuTopologyScorer) Weight() float64 {
    return s.weight
}

func (s *GpuTopologyScorer) Score(ctx context.Context, node *scheduler.NodeInfo, req *scheduler.SandboxRequest) (int, error) {
    // 如果请求不需要多 GPU，不参与打分
    gpuCount := req.HardwareConstraints.GpuCount
    if gpuCount <= 1 {
        return 0, nil
    }

    // 如果不要求 NVLink，降低此插件影响
    if !req.HardwareConstraints.RequireNVLink {
        return 0, nil
    }

    topology := node.HardwareInfo.GpuTopology

    // 检查 NVLink 组是否有足够连接的 GPU
    for _, group := range topology.NvlinkGroups {
        if len(group.GpuIDs) >= gpuCount && group.FullyConnected {
            // NVLink 全连接，最高分
            return 100, nil
        }
    }

    // 检查部分连接
    for _, group := range topology.NvlinkGroups {
        if len(group.GpuIDs) >= gpuCount/2 {
            // 部分连接，中等分
            return 50, nil
        }
    }

    // 无 NVLink 连接，低分
    return 10, nil
}
```

#### 3.3.2 NUMA 亲和性打分

```go
package plugins

import (
    "context"

    "scheduler"
)

// NumaAffinityScorer NUMA 亲和性打分插件
type NumaAffinityScorer struct {
    weight float64
}

func NewNumaAffinityScorer() *NumaAffinityScorer {
    return &NumaAffinityScorer{weight: 0.2}
}

func (s *NumaAffinityScorer) Name() string {
    return "NumaAffinityScorer"
}

func (s *NumaAffinityScorer) Weight() float64 {
    return s.weight
}

func (s *NumaAffinityScorer) Score(ctx context.Context, node *scheduler.NodeInfo, req *scheduler.SandboxRequest) (int, error) {
    // 如果不要求 NUMA 亲和性，不参与打分
    if !req.HardwareConstraints.RequireNUMAAffinity {
        return 0, nil
    }

    gpuCount := req.HardwareConstraints.GpuCount
    if gpuCount == 0 {
        gpuCount = 1
    }

    // 找到能满足请求的 NUMA Node（避免跨 NUMA 访问）
    for _, numa := range node.HardwareInfo.NumaNodes {
        if numa.FreeGpus >= gpuCount && numa.FreeSlots > 0 {
            // 单 NUMA Node 可满足，最高分
            return 100, nil
        }
    }

    // 需要跨 NUMA，降低分数（跨 NUMA 访问延迟增加 2-3 倍）
    return 30, nil
}
```

#### 3.3.3 鲲鹏 SVE 向量优化打分

```go
package plugins

import (
    "context"

    "scheduler"
)

// KunpengSveScorer 鲲鹏 SVE 向量优化打分插件
type KunpengSveScorer struct {
    weight float64
}

func NewKunpengSveScorer() *KunpengSveScorer {
    return &KunpengSveScorer{weight: 0.1}
}

func (s *KunpengSveScorer) Name() string {
    return "KunpengSveScorer"
}

func (s *KunpengSveScorer) Weight() float64 {
    return s.weight
}

func (s *KunpengSveScorer) Score(ctx context.Context, node *scheduler.NodeInfo, req *scheduler.SandboxRequest) (int, error) {
    constraints := req.HardwareConstraints

    // 如果请求不需要向量优化，不参与打分
    if constraints.MinVectorLength == 0 && !constraints.RequireSVE {
        return 0, nil
    }

    cpuFeatures := node.HardwareInfo.CpuFeatures

    // 检查是否为鲲鹏/ARM 架构
    if cpuFeatures.Architecture != "aarch64" {
        return 0, nil
    }

    score := 0

    // SVE 向量长度加分（鲲鹏 920: 256-bit, 鲲鹏 930: 512-bit）
    if constraints.MinVectorLength > 0 {
        if cpuFeatures.VectorLength >= constraints.MinVectorLength {
            score += 50
        } else {
            return 0, nil // 不满足最小要求
        }
    } else if cpuFeatures.VectorLength >= 512 {
        score += 50
    } else if cpuFeatures.VectorLength >= 256 {
        score += 30
    }

    // SVE 指令集支持加分
    if constraints.RequireSVE && !cpuFeatures.SupportsSVE {
        return 0, nil // 不满足要求
    }
    if cpuFeatures.SupportsSVE2 {
        score += 30
    }

    // 矩阵扩展支持加分（用于 Transformer 推理优化）
    if cpuFeatures.SupportsSME {
        score += 20
    }

    return score, nil
}
```

#### 3.3.4 LLC 缓存局部性打分

```go
package plugins

import (
    "context"

    "scheduler"
)

// CacheLocalityScorer LLC 缓存局部性打分插件
type CacheLocalityScorer struct {
    weight float64
}

func NewCacheLocalityScorer() *CacheLocalityScorer {
    return &CacheLocalityScorer{weight: 0.15}
}

func (s *CacheLocalityScorer) Name() string {
    return "CacheLocalityScorer"
}

func (s *CacheLocalityScorer) Weight() float64 {
    return s.weight
}

func (s *CacheLocalityScorer) Score(ctx context.Context, node *scheduler.NodeInfo, req *scheduler.SandboxRequest) (int, error) {
    // 如果不要求缓存局部性，不参与打分
    if !req.HardwareConstraints.PreferCacheLocality {
        return 0, nil
    }

    cacheInfo := node.HardwareInfo.CacheInfo

    // LLC 大小加分（越大越好）
    score := 0
    switch {
    case cacheInfo.LLCSizeMB >= 96:
        score += 60
    case cacheInfo.LLCSizeMB >= 48:
        score += 40
    case cacheInfo.LLCSizeMB >= 24:
        score += 20
    }

    // NUMA 共享 LLC 是加分项（避免跨 NUMA 缓存一致性开销）
    if cacheInfo.NumaShared {
        score += 40
    }

    return score, nil
}
```

---

## 4. 预热池与调度器协同机制

### 4.1 硬件感知预热

**核心思想：预热池不只是"数量"，还包含"硬件亲和性"。**

```
传统预热池:
  Node A: 10 个预热沙箱（随机分布在 NUMA 0/1，随机 GPU）

硬件感知预热池:
  Node A:
    NUMA 0: 4 个预热沙箱（绑定 GPU 0-3, NVLink 组 1）
    NUMA 1: 3 个预热沙箱（绑定 GPU 4-6, NVLink 组 2）
    通用池: 3 个预热沙箱（无硬件绑定，灵活分配）
```

### 4.2 WarmPool CRD 扩展

```yaml
apiVersion: agent-sandbox.kruise.io/v1alpha1
kind: WarmPool
metadata:
  name: gpu-a100-numa0-nvlink1
  labels:
    pool-type: gpu-a100
    node: node-a
spec:
  # 池类型标识
  poolType: gpu-a100

  # 目标节点
  targetNode: node-a

  # 池大小
  replicas: 4

  # 硬件亲和性配置（扩展字段）
  hardwareAffinity:
    # NUMA 节点绑定
    numaNode: 0

    # GPU 组（引用预定义的 NVLink 组）
    gpuGroup: "nvlink-group-1"

    # 可选：GPU 型号
    gpuModel: "NVIDIA-A100-SXM4-80GB"

  # 预热沙箱模板
  template:
    spec:
      runtimeClassName: kata-clh
      containers:
      - name: sandbox
        image: sandbox-base:latest
        resources:
          limits:
            nvidia.com/gpu: 1
            cpu: "8"
            memory: "16Gi"
        # NUMA 绑定通过 annotation 指定
        # 或依赖 kubelet CPU Manager Policy
      nodeSelector:
        kubernetes.io/hostname: node-a

  # 水位策略
  watermarks:
    minReady: 2        # 最小就绪数量
    targetReady: 4     # 目标就绪数量
    maxReady: 6        # 最大就绪数量
```

### 4.3 三层协同模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    三层协同模型                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Layer 1: Fast Scheduler                                  │  │
│  │  速度: 毫秒级                                              │  │
│  │  职责: 从已有的硬件感知预热池中快速分配                      │  │
│  │  协同方式: 读取 Pool Manager 维护的硬件拓扑索引              │  │
│  │  输入: 心跳数据（含硬件拓扑）                               │  │
│  │  输出: 分配决策 + 激活请求                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             ▲                                   │
│                             │ 索引同步                          │
│                             │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Pool Manager                                             │  │
│  │  速度: 秒级                                                │  │
│  │  职责: 预测需求 + 触发池补充 + 硬件亲和性规划                 │  │
│  │  协同方式: 向 Layer 2 发起池补充请求，携带硬件亲和性要求       │  │
│  │  输入: 历史请求数据 + 节点负载 + 硬件拓扑                     │  │
│  │  输出: WarmPool CRD（含 hardwareAffinity）                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             ▲                                   │
│                             │ WarmPool CRD                      │
│                             │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Layer 2: K8s Scheduler + Plugins                         │  │
│  │  速度: 秒级                                                │  │
│  │  职责: 池补充调度 + 硬件拓扑感知 + 全局优化                   │  │
│  │  协同方式: 使用 Scheduler Plugin 实现硬件优化调度             │  │
│  │  输入: WarmPool CRD + 硬件约束                              │  │
│  │  输出: 调度决策（Pod → Node）                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                             ▲                                   │
│                             │ 创建预热沙箱                       │
│                             │                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  OpenKruise                                               │  │
│  │  速度: 百毫秒级                                            │  │
│  │  职责: 预热沙箱生命周期管理 + 心跳上报                        │  │
│  │  协同方式: 根据 hardwareAffinity 创建沙箱，心跳上报硬件状态   │  │
│  │  输入: 调度决策 + hardwareAffinity                         │  │
│  │  输出: 预热沙箱实例 + 心跳数据                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 完整协同流程

```
时间线: 沙箱创建请求处理流程（带硬件优化）

T=0ms:  用户请求到达
        ├─ 请求参数: {
        │    pool_type: "gpu-a100",
        │    gpu_count: 2,
        │    numa_aware: true,
        │    require_nvlink: true
        │  }
        │
        ▼
T=0.1ms: Fast Scheduler 路由决策
        ├─ 检查: 有 NVLink 组且有足够槽位? YES
        │
        ▼
T=0.5ms: Fast Scheduler 执行 Scorer Plugins
        ├─ FreeSlotsScorer: Node A = 5 slots → 80 分
        ├─ GpuTopologyScorer: Node A NVLink 全连接 → 100 分
        ├─ NumaAffinityScorer: Node A NUMA 0 有 2 GPU → 100 分
        ├─ 加权总分:
        │    Node A = 0.3*80 + 0.25*100 + 0.2*100 + 0.15*50 + 0.1*0 = 81.5 分
        │    Node B = 0.3*60 + 0.25*30  + 0.2*30  + 0.15*40 + 0.1*0 = 42.5 分
        │
        ▼
T=1ms:   选择 Node A，从 NUMA 0 预热池分配 2 个槽位
        ├─ 这 2 个沙箱已经在 NVLink 组 1 内
        │
        ▼
T=1.5ms: 通知 Node A Kruise Daemon
        ├─ gRPC: ActivateRequest {
        │    slots: 2,
        │    numa_node: 0,
        │    gpu_group: "nvlink-group-1"
        │  }
        │
        ▼
T=5ms:   Node A 本地激活
        ├─ 从 NUMA 0 预热池取 2 个沙箱
        ├─ 注入用户代码
        ├─ 激活（从 Paused 恢复）
        │
        ▼
T=50ms:  沙箱就绪，返回连接信息
        │
        ▼
T=55ms:  用户开始使用沙箱

---

后台协同流程（并行）:

T=0ms~60s: Pool Manager 监控
        ├─ 发现 Node A NUMA 0 池水位从 4 降到 2
        ├─ 预测模型: 未来 5 分钟需要 6 个沙箱
        ├─ 触发池补充请求
        │
        ▼
T=60s:   创建 WarmPool CRD
        ├─ replicas: 4
        ├─ targetNode: node-a
        ├─ hardwareAffinity: {
        │    numaNode: 0,
        │    gpuGroup: "nvlink-group-1"
        │  }
        │
        ▼
T=65s:   K8s Scheduler 调度预热 Pod
        ├─ Filter: 检查节点是否满足 hardwareAffinity
        ├─ Score: 使用硬件优化插件打分
        ├─ Bind: 绑定到 Node A
        │
        ▼
T=70s:   OpenKruise 创建预热沙箱
        ├─ kubelet 启动容器
        ├─ 根据 hardwareAffinity 绑定资源:
        │    - cpuset.cpus: 绑定到 NUMA 0 的 CPU
        │    - GPU 设备: 绑定到 GPU 0-3（NVLink 组 1）
        ├─ 沙箱进入 Paused 状态
        │
        ▼
T=75s:   Kruise Daemon 心跳更新
        ├─ free_slots: 4
        ├─ numa_info: [
        │    { id: 0, free: 4, free_gpus: 2 },
        │    { id: 1, free: 3, free_gpus: 3 }
        │  ]
        │
        ▼
T=76s:   Fast Scheduler 槽位索引更新
        ├─ Node A 槽位恢复
        ├─ 硬件拓扑信息同步更新
```

---

## 5. 配置与部署

### 5.1 Fast Scheduler 配置

```yaml
# fast-scheduler-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fast-scheduler-config
  namespace: sandbox-system
data:
  scheduler.yaml: |
    # 调度器配置
    server:
      grpc:
        port: 9090
      http:
        port: 8080

    # 节点心跳配置
    heartbeat:
      listenAddr: ":9100"
      protocol: "udp"
      timeout: 9s          # 3 次心跳超时

    # 槽位池配置
    pools:
      - name: gpu-a100
        type: gpu
        scorerPlugins:
          - name: FreeSlotsScorer
            weight: 0.3
          - name: GpuTopologyScorer
            weight: 0.25
          - name: NumaAffinityScorer
            weight: 0.2
          - name: CacheLocalityScorer
            weight: 0.15
          - name: KunpengSveScorer
            weight: 0.1

      - name: cpu-only
        type: cpu
        scorerPlugins:
          - name: FreeSlotsScorer
            weight: 0.5
          - name: CacheLocalityScorer
            weight: 0.3
          - name: NumaAffinityScorer
            weight: 0.2

    # Layer 2 降级配置
    fallback:
      enabled: true
      kubernetesScheduler:
        kubeconfig: /etc/kubernetes/scheduler.conf
      threshold: 0.3    # 池水位低于 30% 时触发降级
```

### 5.2 Kruise Daemon 配置扩展

```yaml
# kruise-daemon-config.yaml (扩展部分)
apiVersion: v1
kind: ConfigMap
metadata:
  name: kruise-daemon-config
  namespace: kruise-system
data:
  daemon.yaml: |
    # 现有配置 ...

    # 新增: Fast Scheduler 配置
    fastScheduler:
      enabled: true
      endpoint: "fast-scheduler.sandbox-system.svc:9100"
      heartbeatInterval: 3s

      # 快速激活服务
      quickActivate:
        enabled: true
        grpcPort: 9091

    # 新增: 硬件感知配置
    hardwareAware:
      enabled: true
      collectors:
        - numaTopology: true
        - gpuTopology: true
        - cacheInfo: true
        - cpuFeatures: true
      updateInterval: 10s
```

### 5.3 部署架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    部署架构                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Namespace: sandbox-system                                 │  │
│  │                                                           │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │  │
│  │  │ Fast Scheduler  │  │ Fast Scheduler  │  (3 replicas)   │  │
│  │  │ Pod (leader)    │  │ Pod (follower)  │                 │  │
│  │  └─────────────────┘  └─────────────────┘                 │  │
│  │           │                    │                          │  │
│  │           └────────────────────┘                          │  │
│  │                        │                                  │  │
│  │                        ▼                                  │  │
│  │           ┌───────────────────────┐                       │  │
│  │           │ Fast Scheduler Service│                       │  │
│  │           │ (LoadBalancer)        │                       │  │
│  │           └───────────────────────┘                       │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Namespace: kruise-system                                  │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────┐      │  │
│  │  │ Kruise Daemon DaemonSet (per-node)               │      │  │
│  │  │ ┌─────────┐ ┌─────────┐ ┌─────────┐              │      │  │
│  │  │ │ Node A  │ │ Node B  │ │ Node C  │              │      │  │
│  │  │ │ Daemon  │ │ Daemon  │ │ Daemon  │              │      │  │
│  │  │ └────┬────┘ └────┬────┘ └────┬────┘              │      │  │
│  │  │      │ UDP心跳    │          │                   │      │  │
│  │  │      └────────────┼──────────┘                   │      │  │
│  │  │                   ▼                              │      │  │
│  │  │         Fast Scheduler Service                   │      │  │
│  │  └─────────────────────────────────────────────────┘      │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 决策记录

| 决策点 | 选择 | 原因 |
|--------|------|------|
| OpenKruise 集成方式 | Side Scheduler | 非侵入式，零核心代码修改 |
| Kruise Daemon 改动范围 | 新增心跳上报 + 快速激活 | 最小改动，~210 行代码 |
| Scorer Plugin 设计 | 轻量级接口，插件链 | 类似 K8s Scheduler Plugin，但更简单 |
| 预热池模型 | 硬件感知分层 | 支持硬件亲和性调度 |
| Layer 1 ↔ Layer 2 通信 | WarmPool CRD | 声明式，K8s 原生 |
| 心跳协议 | UDP | 低开销，适合高频状态同步 |

---

## 7. 实施路线图

| 阶段 | 内容 | 依赖 | 交付物 |
|------|------|------|--------|
| **Phase 1** | Fast Scheduler 核心框架 | 无 | 调度器原型（内存索引 + Best-Fit） |
| **Phase 2** | Kruise Daemon 扩展 | Phase 1 | 心跳上报 + 快速激活接口 |
| **Phase 3** | 硬件感知插件 | Phase 2 | NUMA/NVLink/SVE Scorer Plugins |
| **Phase 4** | 预热池协同 | Phase 3 | Pool Manager + WarmPool CRD |
| **Phase 5** | 生产化 | Phase 4 | 监控 + 高可用 + 文档 |

---

## 8. 附录

### 8.1 性能预期

| 指标 | 目标值 | 测量方法 |
|------|--------|---------|
| Fast Path 调度延迟 | <1ms | 调度器内部计时 |
| Fast Path 端到端延迟 | <50ms | 请求入口到沙箱就绪 |
| Fast Path 命中率 | >95% | Fast Path 请求数 / 总请求数 |
| 心跳延迟 | <10ms | 节点到调度器 RTT |
| 预热池补充延迟 | <10s | 触发补充到沙箱就绪 |

### 8.2 兼容性矩阵

| 组件 | 版本要求 |
|------|---------|
| Kubernetes | >= 1.24 |
| OpenKruise | >= 1.5 |
| containerd | >= 1.6 |
| NVIDIA GPU Operator | >= 23.3 (如需 GPU 拓扑感知) |
| 鲲鹏硬件 | 920/930 (如需 SVE 优化) |
