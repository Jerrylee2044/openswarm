# OpenSwarm - 详细开发文档

## 文档信息

| 项目 | 内容 |
|------|------|
| 版本 | v0.1.0-alpha |
| 状态 | 开发中 |
| 最后更新 | 2026-03-11 |
| 开发语言 | Rust (Edition 2021) |
| 最低 Rust 版本 | 1.75.0 |

---

## 1. 项目概述

### 1.1 目标

构建一个高性能、可扩展、资源高效的 Agent Swarm 编排框架，满足以下核心需求：

1. **动态生命周期**：Agent 按需创建，任务完成后自动释放
2. **混合通信架构**：分层控制 + Mesh 对等通信
3. **资源零浪费**：精细的资源配额和回收机制
4. **云原生就绪**：支持单机和 Kubernetes 部署

### 1.2 核心特性

- **声明式 Agent 定义**：YAML/JSON 配置驱动
- **QUIC 通信**：0-RTT、内置 mTLS、多路复用
- **WASM 运行时**：轻量级、安全、快速启动
- **智能调度**：负载均衡、任务亲和性、多副本执行
- **完整可观测性**：OpenTelemetry、Prometheus、分布式追踪

### 1.3 技术栈

| 组件 | 选型 | 版本 |
|------|------|------|
| 异步运行时 | tokio | ^1.35 |
| HTTP 框架 | axum | ^0.7 |
| QUIC 实现 | quinn | ^0.10 |
| 序列化 | prost + serde | - |
| 数据库 | sqlx (PostgreSQL) | ^0.7 |
| 缓存 | redis | ^0.24 |
| WASM 运行时 | wasmtime | ^17.0 |
| 配置管理 | config + serde_yaml | - |
| 日志 | tracing + tracing-subscriber | - |
| 指标 | metrics + metrics-prometheus | - |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Client Layer                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   CLI Tool   │  │   Web UI     │  │  SDK (Py/JS) │  │   REST API   │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────────┘
          │                 │                 │                 │
          └─────────────────┴────────┬────────┴─────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────────────────┐
│                           API Gateway                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                     Axum HTTP/2 Server                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │  │
│  │  │  Auth MW    │  │  Rate Limit │  │   Router    │  │   Metrics   │ │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────────────────┐
│                         Control Plane                                        │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────────┐ │
│  │   Scheduler    │  │   Discovery    │  │      Resource Manager          │ │
│  │   (调度器)      │  │   (服务发现)    │  │       (资源管理器)              │ │
│  └────────┬───────┘  └────────┬───────┘  └───────────────┬────────────────┘ │
│           │                   │                          │                  │
│  ┌────────▼───────────────────▼──────────────────────────▼────────────────┐ │
│  │                      Event Bus (Tokio Broadcast)                       │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
┌────────────────────────────────────▼────────────────────────────────────────┐
│                      Agent Runtime Layer                                     │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Agent Pool Manager                                │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────────────┐ │   │
│  │  │  WASM Pod  │ │ WASM Pod   │ │ WASM Pod   │ │   Container Pod    │ │   │
│  │  │   #001     │ │   #002     │ │   #003     │ │      #001          │ │   │
│  │  │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │ │   ┌────────────┐   │ │   │
│  │  │ │Agent   │ │ │ │Agent   │ │ │ │Agent   │ │ │   │   Agent    │   │ │   │
│  │  │ │Instance│ │ │ │Instance│ │ │ │Instance│ │ │   │  Instance  │   │ │   │
│  │  │ └────────┘ │ │ └────────┘ │ │ └────────┘ │ │   └────────────┘   │ │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                      Mesh Network (QUIC)                             │   │
│  │         ┌──────────────┐  Gossip Protocol  ┌──────────────┐          │   │
│  │         │   Agent A    │◄─────────────────►│   Agent B    │          │   │
│  │         └──────────────┘                   └──────────────┘          │   │
│  │                   ▲                               ▲                  │   │
│  │                   └───────────────┬───────────────┘                  │   │
│  │                                   │                                  │   │
│  │                            ┌──────────────┐                          │   │
│  │                            │   Agent C    │                          │   │
│  │                            └──────────────┘                          │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Storage Layer                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │    Redis     │  │  PostgreSQL  │  │    MinIO     │  │   Etcd/ZK    │     │
│  │ (消息队列/缓存) │  │  (状态持久化) │  │ (对象存储)    │  │ (分布式协调)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 模块划分

```
OpenSwarm/
├── crates/
│   ├── openswarm-core/          # 核心类型和 trait
│   ├── openswarm-runtime/       # Agent 运行时 (WASM/Container)
│   ├── openswarm-scheduler/     # 任务调度器
│   ├── openswarm-network/       # QUIC 网络层
│   ├── openswarm-discovery/     # 服务发现
│   ├── openswarm-storage/       # 存储抽象
│   ├── openswarm-api/           # HTTP/gRPC API
│   ├── openswarm-cli/           # 命令行工具
│   └── openswarm-operator/      # K8s Operator
├── proto/                    # Protocol Buffer 定义
├── docs/                     # 文档
├── examples/                 # 示例代码
└── deployments/              # 部署配置
```

---

## 3. 核心模块详细设计

### 3.1 openswarm-core - 核心类型定义

**文件位置**: `crates/openswarm-core/src/lib.rs`

#### 3.1.1 Agent 基础定义

```rust
//! 核心类型定义
//! 
//! 本模块定义了整个系统的基础类型和 trait

use std::collections::HashMap;
use std::time::{Duration, SystemTime};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// 全局唯一标识符类型别名
pub type AgentId = Uuid;
pub type TaskId = Uuid;
pub type SessionId = Uuid;
pub type NodeId = Uuid;

/// Agent 执行状态
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum AgentState {
    /// 正在创建
    Creating,
    /// 就绪，等待任务
    Idle,
    /// 执行中
    Running,
    /// 暂停
    Paused,
    /// 正在释放
    Terminating,
    /// 已释放
    Terminated,
    /// 故障
    Failed,
}

/// Agent 运行时类型
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum RuntimeType {
    /// WASM 运行时（推荐）
    Wasm,
    /// 容器运行时
    Container,
    /// 进程运行时（仅单机模式）
    Process,
}

/// Agent 资源配额
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceQuota {
    /// CPU 限制 (millicores)
    pub cpu_limit: u32,
    /// 内存限制 (bytes)
    pub memory_limit: u64,
    /// 临时存储限制 (bytes)
    pub ephemeral_storage: u64,
    /// GPU 限制（可选）
    pub gpu_limit: Option<u32>,
    /// 最大并发任务数
    pub max_concurrent_tasks: usize,
    /// 空闲超时时间
    pub idle_timeout: Duration,
    /// 最大生命周期（硬限制）
    pub max_lifetime: Duration,
}

impl Default for ResourceQuota {
    fn default() -> Self {
        Self {
            cpu_limit: 1000,  // 1 core
            memory_limit: 512 * 1024 * 1024,  // 512MB
            ephemeral_storage: 1 * 1024 * 1024 * 1024,  // 1GB
            gpu_limit: None,
            max_concurrent_tasks: 1,
            idle_timeout: Duration::from_secs(300),  // 5分钟
            max_lifetime: Duration::from_secs(3600),  // 1小时
        }
    }
}

/// Agent 能力声明
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Capability {
    /// 能力名称
    pub name: String,
    /// 能力描述
    pub description: String,
    /// 输入类型（JSON Schema）
    pub input_schema: serde_json::Value,
    /// 输出类型（JSON Schema）
    pub output_schema: serde_json::Value,
    /// 执行超时
    pub timeout: Duration,
    /// 是否需要人工确认
    pub require_human_approval: bool,
}

/// Agent 定义（声明式配置）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentSpec {
    /// API 版本
    pub api_version: String,
    /// 元数据
    pub metadata: AgentMetadata,
    /// 运行时配置
    pub runtime: RuntimeConfig,
    /// 资源配额
    pub resources: ResourceQuota,
    /// 能力列表
    pub capabilities: Vec<Capability>,
    /// 交接规则
    pub handoff_rules: Vec<HandoffRule>,
    /// 模型配置
    pub model: Option<ModelConfig>,
    /// 环境变量
    pub env: HashMap<String, String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentMetadata {
    /// 名称（唯一）
    pub name: String,
    /// 命名空间
    pub namespace: String,
    /// 版本
    pub version: String,
    /// 标签
    pub labels: HashMap<String, String>,
    /// 注解
    pub annotations: HashMap<String, String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RuntimeConfig {
    /// 运行时类型
    pub runtime_type: RuntimeType,
    /// WASM 模块或容器镜像
    pub image: String,
    /// 入口点（可选，覆盖默认）
    pub entrypoint: Option<Vec<String>>,
    /// 启动命令参数
    pub args: Option<Vec<String>>,
    /// 工作目录
    pub working_dir: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ModelConfig {
    /// 提供商（openai, anthropic, local 等）
    pub provider: String,
    /// 模型名称
    pub name: String,
    /// 温度参数
    pub temperature: Option<f32>,
    /// 最大 token
    pub max_tokens: Option<u32>,
    /// API 密钥引用（从 secret 读取）
    pub api_key_secret: Option<String>,
}

/// 交接规则
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HandoffRule {
    /// 条件表达式（CEL 格式）
    pub condition: String,
    /// 目标 Agent 模板
    pub target_agent: String,
    /// 上下文传递方式
    pub context_transfer: ContextTransferMode,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ContextTransferMode {
    /// 完整上下文传递
    Full,
    /// 仅传递摘要
    Summary,
    /// 自定义过滤
    Filtered(Vec<String>),
}

/// Agent 实例信息
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AgentInstance {
    /// 实例 ID
    pub id: AgentId,
    /// 所属节点
    pub node_id: NodeId,
    /// Agent 模板名称
    pub spec_name: String,
    /// 当前状态
    pub state: AgentState,
    /// 资源使用
    pub resource_usage: ResourceUsage,
    /// 启动时间
    pub started_at: SystemTime,
    /// 最后活跃时间
    pub last_active_at: SystemTime,
    /// 网络地址（用于 Mesh 通信）
    pub mesh_addr: Option<String>,
    /// 会话列表
    pub sessions: Vec<SessionId>,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ResourceUsage {
    /// CPU 使用量 (millicores)
    pub cpu_millicores: u32,
    /// 内存使用量 (bytes)
    pub memory_bytes: u64,
    /// 存储使用量 (bytes)
    pub storage_bytes: u64,
}

/// 任务定义
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Task {
    /// 任务 ID
    pub id: TaskId,
    /// 父任务 ID（用于子任务）
    pub parent_id: Option<TaskId>,
    /// 任务类型
    pub task_type: String,
    /// 输入数据
    pub input: serde_json::Value,
    /// 优先级
    pub priority: TaskPriority,
    /// 超时时间
    pub timeout: Duration,
    /// 关联的 Agent 类型
    pub required_capabilities: Vec<String>,
    /// 任务标签（用于调度亲和性）
    pub labels: HashMap<String, String>,
    /// 创建时间
    pub created_at: SystemTime,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TaskPriority {
    Critical = 0,
    High = 1,
    Normal = 2,
    Low = 3,
}

/// 任务状态
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TaskStatus {
    /// 任务 ID
    pub task_id: TaskId,
    /// 执行状态
    pub state: TaskState,
    /// 分配的 Agent
    pub assigned_agent: Option<AgentId>,
    /// 开始时间
    pub started_at: Option<SystemTime>,
    /// 完成时间
    pub completed_at: Option<SystemTime>,
    /// 输出结果
    pub result: Option<serde_json::Value>,
    /// 错误信息
    pub error: Option<String>,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum TaskState {
    Pending,
    Scheduled,
    Running,
    Paused,
    Completed,
    Failed,
    Cancelled,
    Timeout,
}

/// 会话（多轮交互上下文）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Session {
    /// 会话 ID
    pub id: SessionId,
    /// 根任务 ID
    pub root_task_id: TaskId,
    /// 参与的 Agent 列表
    pub agents: Vec<AgentId>,
    /// 当前活跃的 Agent
    pub current_agent: Option<AgentId>,
    /// 消息历史
    pub messages: Vec<Message>,
    /// 上下文变量
    pub context_variables: HashMap<String, serde_json::Value>,
    /// 创建时间
    pub created_at: SystemTime,
    /// 最后更新时间
    pub updated_at: SystemTime,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Message {
    /// 消息 ID
    pub id: Uuid,
    /// 发送者（Agent ID 或 "user"）
    pub sender: String,
    /// 接收者
    pub recipient: Option<String>,
    /// 消息类型
    pub message_type: MessageType,
    /// 内容
    pub content: serde_json::Value,
    /// 时间戳
    pub timestamp: SystemTime,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum MessageType {
    UserQuery,
    AgentResponse,
    ToolCall,
    ToolResult,
    HandoffRequest,
    HandoffAccept,
    SystemEvent,
}

/// 核心 trait：Agent 生命周期管理
#[async_trait::async_trait]
pub trait AgentLifecycle: Send + Sync {
    /// 创建 Agent 实例
    async fn create(&self, spec: AgentSpec) -> Result<AgentInstance, AgentError>;
    
    /// 启动 Agent
    async fn start(&self, agent_id: AgentId) -> Result<(), AgentError>;
    
    /// 暂停 Agent
    async fn pause(&self, agent_id: AgentId) -> Result<(), AgentError>;
    
    /// 恢复 Agent
    async fn resume(&self, agent_id: AgentId) -> Result<(), AgentError>;
    
    /// 终止 Agent
    async fn terminate(&self, agent_id: AgentId) -> Result<(), AgentError>;
    
    /// 获取 Agent 状态
    async fn get_status(&self, agent_id: AgentId) -> Result<AgentInstance, AgentError>;
    
    /// 列出所有 Agent
    async fn list_agents(&self, filter: AgentFilter) -> Result<Vec<AgentInstance>, AgentError>;
}

/// Agent 过滤条件
#[derive(Debug, Clone, Default)]
pub struct AgentFilter {
    pub namespace: Option<String>,
    pub spec_name: Option<String>,
    pub state: Option<AgentState>,
    pub node_id: Option<NodeId>,
    pub labels: HashMap<String, String>,
}

/// Agent 错误类型
#[derive(Debug, thiserror::Error)]
pub enum AgentError {
    #[error("Agent not found: {0}")]
    NotFound(AgentId),
    
    #[error("Resource exhausted: {0}")]
    ResourceExhausted(String),
    
    #[error("Invalid spec: {0}")]
    InvalidSpec(String),
    
    #[error("Runtime error: {0}")]
    RuntimeError(String),
    
    #[error("Communication error: {0}")]
    CommunicationError(String),
    
    #[error("Timeout")]
    Timeout,
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

/// 核心 trait：任务调度
#[async_trait::async_trait]
pub trait TaskScheduler: Send + Sync {
    /// 提交任务
    async fn submit(&self, task: Task) -> Result<TaskId, SchedulerError>;
    
    /// 取消任务
    async fn cancel(&self, task_id: TaskId) -> Result<(), SchedulerError>;
    
    /// 获取任务状态
    async fn get_status(&self, task_id: TaskId) -> Result<TaskStatus, SchedulerError>;
    
    /// 等待任务完成
    async fn wait_for_completion(&self, task_id: TaskId) -> Result<TaskStatus, SchedulerError>;
}

#[derive(Debug, thiserror::Error)]
pub enum SchedulerError {
    #[error("Task not found: {0}")]
    NotFound(TaskId),
    
    #[error("No available agent for task")]
    NoAvailableAgent,
    
    #[error("Queue full")]
    QueueFull,
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

/// 核心 trait：网络通信
#[async_trait::async_trait]
pub trait NetworkTransport: Send + Sync {
    /// 发送消息到指定 Agent
    async fn send_to(&self, agent_id: AgentId, message: Message) -> Result<(), NetworkError>;
    
    /// 广播消息到所有 Agent
    async fn broadcast(&self, message: Message) -> Result<(), NetworkError>;
    
    /// 接收消息
    async fn receive(&self) -> Result<(AgentId, Message), NetworkError>;
}

#[derive(Debug, thiserror::Error)]
pub enum NetworkError {
    #[error("Connection refused")]
    ConnectionRefused,
    
    #[error("Agent unreachable: {0}")]
    Unreachable(AgentId),
    
    #[error("Timeout")]
    Timeout,
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

---

### 3.2 openswarm-runtime - Agent 运行时

**文件位置**: `crates/openswarm-runtime/src/lib.rs`

#### 3.2.1 运行时管理器

```rust
//! Agent 运行时管理
//!
//! 负责管理 WASM/Container/Process 三种运行时的生命周期

use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{Mutex, RwLock};
use wasmtime::{Engine, Module, Store};

use pswarm_core::*;

/// 运行时管理器
pub struct RuntimeManager {
    /// WASM 引擎
    wasm_engine: Engine,
    /// 模块缓存
    module_cache: Arc<RwLock<HashMap<String, Module>>>,
    /// 活跃实例
    instances: Arc<Mutex<HashMap<AgentId, Box<dyn AgentRuntime>>>>,
    /// 资源池
    resource_pool: Arc<ResourcePool>,
    /// 配置
    config: RuntimeConfig,
}

#[derive(Debug, Clone)]
pub struct RuntimeConfig {
    /// WASM 编译缓存目录
    pub wasm_cache_dir: std::path::PathBuf,
    /// 最大并发实例数
    pub max_instances: usize,
    /// 预热实例数
    pub warmup_instances: usize,
    /// 容器运行时（containerd/cri-o）
    pub container_runtime: String,
    /// 网络模式
    pub network_mode: NetworkMode,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NetworkMode {
    /// 隔离网络
    Isolated,
    /// 共享网络
    Shared,
    /// 主机网络（仅单机）
    Host,
}

/// 运行时 trait
#[async_trait::async_trait]
pub trait AgentRuntime: Send + Sync {
    /// 获取 Agent ID
    fn agent_id(&self) -> AgentId;
    
    /// 获取运行时类型
    fn runtime_type(&self) -> RuntimeType;
    
    /// 执行单个任务
    async fn execute(&self, task: Task) -> Result<TaskResult, RuntimeError>;
    
    /// 开始会话模式
    async fn start_session(&self, session: Session) -> Result<(), RuntimeError>;
    
    /// 发送会话消息
    async fn send_message(&self, message: Message) -> Result<(), RuntimeError>;
    
    /// 接收会话消息
    async fn receive_message(&self) -> Result<Message, RuntimeError>;
    
    /// 获取资源使用
    fn resource_usage(&self) -> ResourceUsage;
    
    /// 创建检查点
    async fn checkpoint(&self) -> Result<Checkpoint, RuntimeError>;
    
    /// 从检查点恢复
    async fn restore(&self, checkpoint: Checkpoint) -> Result<(), RuntimeError>;
    
    /// 优雅关闭
    async fn graceful_shutdown(&self) -> Result<(), RuntimeError>;
    
    /// 强制终止
    async fn force_terminate(&self) -> Result<(), RuntimeError>;
}

/// 任务执行结果
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TaskResult {
    /// 任务 ID
    pub task_id: TaskId,
    /// 执行状态
    pub status: TaskStatus,
    /// 输出数据
    pub output: Option<serde_json::Value>,
    /// 执行指标
    pub metrics: ExecutionMetrics,
}

#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct ExecutionMetrics {
    /// 启动延迟 (ms)
    pub startup_latency_ms: u64,
    /// 执行耗时 (ms)
    pub execution_time_ms: u64,
    /// 内存峰值 (bytes)
    pub memory_peak_bytes: u64,
    /// Token 使用量（LLM 任务）
    pub token_usage: Option<TokenUsage>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TokenUsage {
    pub prompt_tokens: u32,
    pub completion_tokens: u32,
    pub total_tokens: u32,
}

/// 运行时检查点
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Checkpoint {
    /// 检查点 ID
    pub id: Uuid,
    /// Agent ID
    pub agent_id: AgentId,
    /// 创建时间
    pub created_at: SystemTime,
    /// 状态数据
    pub state_data: Vec<u8>,
    /// 内存快照（WASM 线性内存）
    pub memory_snapshot: Option<Vec<u8>>,
}

/// WASM 运行时实现
pub struct WasmRuntime {
    agent_id: AgentId,
    store: Arc<Mutex<Store<WasmContext>>>,
    instance: wasmtime::Instance,
    resource_usage: Arc<RwLock<ResourceUsage>>,
    metrics_collector: MetricsCollector,
}

struct WasmContext {
    /// 标准输出
    stdout: Vec<u8>,
    /// 标准错误
    stderr: Vec<u8>,
    /// 主机函数接口
    host_functions: HostFunctions,
}

/// 主机函数（WASM 可以调用的宿主功能）
pub struct HostFunctions {
    /// HTTP 请求
    pub http_client: Arc<dyn HttpClient>,
    /// 日志
    pub logger: Arc<dyn Logger>,
    /// KV 存储
    pub kv_store: Arc<dyn KVStore>,
    /// LLM 调用
    pub llm_client: Arc<dyn LLMClient>,
}

/// 容器运行时实现
pub struct ContainerRuntime {
    agent_id: AgentId,
    container_id: String,
    /// containerd/cri-o 客户端
    client: Arc<dyn ContainerClient>,
    /// 网络端点
    network_endpoint: String,
    resource_usage: Arc<RwLock<ResourceUsage>>,
}

/// 资源池（预热实例）
pub struct ResourcePool {
    /// 按 spec 分组的预热实例
    warm_instances: Arc<RwLock<HashMap<String, Vec<Box<dyn AgentRuntime>>>>>,
    /// 最大预热数
    max_warm_per_spec: usize,
    /// 资源管理器
    resource_manager: Arc<ResourceManager>,
}

impl ResourcePool {
    /// 获取或创建实例
    pub async fn acquire(&self, spec: &AgentSpec) -> Result<Box<dyn AgentRuntime>, PoolError> {
        // 1. 尝试获取预热实例
        if let Some(instance) = self.try_get_warm(spec).await {
            // 重新配置
            instance.reconfigure(spec).await?;
            return Ok(instance);
        }
        
        // 2. 创建新实例
        self.create_new(spec).await
    }
    
    /// 释放实例回池子
    pub async fn release(&self, runtime: Box<dyn AgentRuntime>) {
        let spec_name = runtime.spec_name();
        let mut instances = self.warm_instances.write().await;
        
        let pool = instances.entry(spec_name).or_default();
        if pool.len() < self.max_warm_per_spec {
            pool.push(runtime);
        } else {
            // 池子满了，销毁
            drop(runtime);
        }
    }
}

/// 资源管理器
pub struct ResourceManager {
    /// 总资源配额
    total_quota: ResourceQuota,
    /// 已分配资源
    allocated: Arc<RwLock<ResourceUsage>>,
    /// 节点信息（K8s 模式）
    node_info: Option<NodeInfo>,
}

impl ResourceManager {
    /// 检查是否有足够资源
    pub async fn check_availability(&self, required: &ResourceQuota) -> Result<(), ResourceError> {
        let allocated = self.allocated.read().await;
        let total = &self.total_quota;
        
        if allocated.cpu_millicores + required.cpu_limit > total.cpu_limit {
            return Err(ResourceError::CpuExceeded);
        }
        
        if allocated.memory_bytes + required.memory_limit > total.memory_limit {
            return Err(ResourceError::MemoryExceeded);
        }
        
        Ok(())
    }
    
    /// 分配资源
    pub async fn allocate(&self, quota: &ResourceQuota) -> Result<ResourceHandle, ResourceError> {
        self.check_availability(quota).await?;
        
        let mut allocated = self.allocated.write().await;
        allocated.cpu_millicores += quota.cpu_limit;
        allocated.memory_bytes += quota.memory_limit;
        
        Ok(ResourceHandle {
            quota: quota.clone(),
            allocator: Arc::downgrade(&self.allocated),
        })
    }
}

/// 资源句柄（自动释放）
pub struct ResourceHandle {
    quota: ResourceQuota,
    allocator: std::sync::Weak<RwLock<ResourceUsage>>,
}

impl Drop for ResourceHandle {
    fn drop(&mut self) {
        if let Some(allocator) = self.allocator.upgrade() {
            // 异步释放资源
            let quota = self.quota.clone();
            tokio::spawn(async move {
                let mut allocated = allocator.write().await;
                allocated.cpu_millicores -= quota.cpu_limit;
                allocated.memory_bytes -= quota.memory_limit;
            });
        }
    }
}
```

---

### 3.3 openswarm-scheduler - 任务调度器

**文件位置**: `crates/openswarm-scheduler/src/lib.rs`

```rust
//! 任务调度器
//!
//! 负责任务队列管理、调度策略、负载均衡

use std::collections::{BinaryHeap, HashMap, VecDeque};
use std::sync::Arc;
use tokio::sync::{Mutex, Notify, RwLock};
use tokio::time::{interval, Duration, Instant};

use pswarm_core::*;

/// 调度器配置
#[derive(Debug, Clone)]
pub struct SchedulerConfig {
    /// 调度策略
    pub strategy: SchedulingStrategy,
    /// 任务队列容量
    pub queue_capacity: usize,
    /// 重试次数
    pub max_retries: u32,
    /// 重试间隔
    pub retry_backoff: Duration,
    /// 任务超时检查间隔
    pub timeout_check_interval: Duration,
    /// 是否启用任务亲和性
    pub enable_affinity: bool,
    /// 是否启用多副本执行（关键任务）
    pub enable_replication: bool,
}

impl Default for SchedulerConfig {
    fn default() -> Self {
        Self {
            strategy: SchedulingStrategy::Adaptive,
            queue_capacity: 10000,
            max_retries: 3,
            retry_backoff: Duration::from_secs(5),
            timeout_check_interval: Duration::from_secs(10),
            enable_affinity: true,
            enable_replication: false,
        }
    }
}

/// 调度策略
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SchedulingStrategy {
    /// 轮询
    RoundRobin,
    /// 最少连接
    LeastConnections,
    /// 资源加权
    ResourceWeighted,
    /// 自适应（推荐）
    Adaptive,
}

/// 调度器实现
pub struct Scheduler {
    /// 配置
    config: SchedulerConfig,
    /// 任务队列（按优先级）
    task_queue: Arc<Mutex<BinaryHeap<QueuedTask>>>,
    /// 运行中任务
    running_tasks: Arc<RwLock<HashMap<TaskId, TaskHandle>>>,
    /// Agent 管理器
    agent_manager: Arc<dyn AgentManager>,
    /// 调度通知
    notify: Arc<Notify>,
    /// 指标
    metrics: Arc<SchedulerMetrics>,
}

/// 队列中的任务（带优先级）
#[derive(Debug, Clone)]
struct QueuedTask {
    /// 优先级（越小越高）
    priority: u8,
    /// 入队时间
    enqueued_at: Instant,
    /// 任务
    task: Task,
    /// 重试次数
    retries: u32,
}

impl Ord for QueuedTask {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        // 优先级优先，然后 FIFO
        self.priority
            .cmp(&other.priority)
            .then_with(|| other.enqueued_at.cmp(&self.enqueued_at))
    }
}

impl PartialOrd for QueuedTask {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl PartialEq for QueuedTask {
    fn eq(&self, other: &Self) -> bool {
        self.priority == other.priority && self.enqueued_at == other.enqueued_at
    }
}

impl Eq for QueuedTask {}

impl Scheduler {
    pub fn new(config: SchedulerConfig, agent_manager: Arc<dyn AgentManager>) -> Self {
        Self {
            config,
            task_queue: Arc::new(Mutex::new(BinaryHeap::new())),
            running_tasks: Arc::new(RwLock::new(HashMap::new())),
            agent_manager,
            notify: Arc::new(Notify::new()),
            metrics: Arc::new(SchedulerMetrics::default()),
        }
    }
    
    /// 启动调度循环
    pub async fn start(&self) {
        let config = self.config.clone();
        let queue = self.task_queue.clone();
        let running = self.running_tasks.clone();
        let agent_mgr = self.agent_manager.clone();
        let notify = self.notify.clone();
        let metrics = self.metrics.clone();
        
        // 调度循环
        tokio::spawn(async move {
            loop {
                notify.notified().await;
                
                while let Some(queued) = queue.lock().await.pop() {
                    match Self::try_schedule(&queued, &agent_mgr).await {
                        Ok(handle) => {
                            running.write().await.insert(queued.task.id, handle);
                            metrics.tasks_scheduled.inc();
                        }
                        Err(e) => {
                            // 重新入队或标记失败
                            if queued.retries < config.max_retries {
                                let mut retry = queued;
                                retry.retries += 1;
                                queue.lock().await.push(retry);
                            } else {
                                metrics.tasks_failed.inc();
                            }
                        }
                    }
                }
            }
        });
        
        // 超时检查循环
        let running_check = self.running_tasks.clone();
        let timeout_interval = self.config.timeout_check_interval;
        tokio::spawn(async move {
            let mut ticker = interval(timeout_interval);
            loop {
                ticker.tick().await;
                Self::check_timeouts(&running_check).await;
            }
        });
    }
    
    /// 提交任务
    pub async fn submit(&self, task: Task) -> Result<TaskId, SchedulerError> {
        // 检查队列容量
        let queue_len = self.task_queue.lock().await.len();
        if queue_len >= self.config.queue_capacity {
            return Err(SchedulerError::QueueFull);
        }
        
        let priority = match task.priority {
            TaskPriority::Critical => 0,
            TaskPriority::High => 1,
            TaskPriority::Normal => 2,
            TaskPriority::Low => 3,
        };
        
        let queued = QueuedTask {
            priority,
            enqueued_at: Instant::now(),
            task: task.clone(),
            retries: 0,
        };
        
        self.task_queue.lock().await.push(queued);
        self.metrics.tasks_submitted.inc();
        
        // 通知调度器
        self.notify.notify_one();
        
        Ok(task.id)
    }
    
    /// 尝试调度任务
    async fn try_schedule(
        queued: &QueuedTask,
        agent_mgr: &Arc<dyn AgentManager>,
    ) -> Result<TaskHandle, SchedulerError> {
        let task = &queued.task;
        
        // 1. 查找匹配能力的 Agent
        let candidates = agent_mgr
            .find_candidates(&task.required_capabilities)
            .await?;
        
        if candidates.is_empty() {
            // 没有匹配 Agent，尝试动态创建
            let new_agent = agent_mgr.create_for_task(task).await?;
            return Self::assign_task(task, new_agent).await;
        }
        
        // 2. 应用调度策略选择最优 Agent
        let selected = Self::select_agent(&candidates, task).await?;
        
        // 3. 分配任务
        Self::assign_task(task, selected).await
    }
    
    /// 选择最优 Agent
    async fn select_agent(
        candidates: &[AgentInstance],
        task: &Task,
    ) -> Result<AgentInstance, SchedulerError> {
        // 策略：最少负载 + 亲和性
        let mut scored: Vec<_> = candidates
            .iter()
            .map(|agent| {
                let load_score = agent.resource_usage.cpu_millicores as f64
                    / agent.resource_quota.cpu_limit as f64;
                
                // 亲和性加分（历史协作过）
                let affinity_score = if task.labels.get("session_id").map(|s| {
                    agent.sessions.iter().any(|sid| sid.to_string() == *s)
                }).unwrap_or(false) {
                    0.2
                } else {
                    0.0
                };
                
                let score = load_score - affinity_score;
                (agent.clone(), score)
            })
            .collect();
        
        scored.sort_by(|a, b| a.1.partial_cmp(&b.1).unwrap());
        
        scored
            .into_iter()
            .next()
            .map(|(agent, _)| agent)
            .ok_or(SchedulerError::NoAvailableAgent)
    }
    
    /// 分配任务到 Agent
    async fn assign_task(
        task: &Task,
        agent: AgentInstance,
    ) -> Result<TaskHandle, SchedulerError> {
        // 发送任务到 Agent
        // 实际实现通过消息队列或网络调用
        let handle = TaskHandle {
            task_id: task.id,
            agent_id: agent.id,
            started_at: Instant::now(),
            timeout: task.timeout,
        };
        
        Ok(handle)
    }
    
    /// 检查超时任务
    async fn check_timeouts(running: &Arc<RwLock<HashMap<TaskId, TaskHandle>>>) {
        let now = Instant::now();
        let mut to_cancel = Vec::new();
        
        {
            let running_lock = running.read().await;
            for (task_id, handle) in running_lock.iter() {
                if now.duration_since(handle.started_at) > handle.timeout {
                    to_cancel.push(*task_id);
                }
            }
        }
        
        for task_id in to_cancel {
            // 取消超时任务
            // ...
        }
    }
}

/// 任务句柄
#[derive(Debug)]
struct TaskHandle {
    task_id: TaskId,
    agent_id: AgentId,
    started_at: Instant,
    timeout: Duration,
}

/// Agent 管理 trait
#[async_trait::async_trait]
pub trait AgentManager: Send + Sync {
    /// 查找匹配能力的候选 Agent
    async fn find_candidates(
        &self,
        capabilities: &[String],
    ) -> Result<Vec<AgentInstance>, SchedulerError>;
    
    /// 为任务动态创建 Agent
    async fn create_for_task(&self, task: &Task) -> Result<AgentInstance, SchedulerError>;
}

/// 调度器指标
#[derive(Debug, Default)]
struct SchedulerMetrics {
    tasks_submitted: Counter,
    tasks_scheduled: Counter,
    tasks_completed: Counter,
    tasks_failed: Counter,
    tasks_timeout: Counter,
    queue_depth: Gauge,
    scheduling_latency: Histogram,
}
```

---

### 3.4 openswarm-network - QUIC 网络层

**文件位置**: `crates/openswarm-network/src/lib.rs`

```rust
//! QUIC 网络传输层
//!
//! 基于 quinn 实现高性能、安全的 Agent 间通信

use std::collections::HashMap;
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::sync::{mpsc, RwLock};
use quinn::{ClientConfig, Connection, Endpoint, ServerConfig};
use rustls::pki_types::{CertificateDer, PrivateKeyDer};

use pswarm_core::*;

/// 网络配置
#[derive(Debug, Clone)]
pub struct NetworkConfig {
    /// 监听地址
    pub bind_addr: SocketAddr,
    /// 证书路径
    pub cert_path: std::path::PathBuf,
    /// 私钥路径
    pub key_path: std::path::PathBuf,
    /// 最大并发连接
    pub max_connections: usize,
    /// 连接超时
    pub connection_timeout: Duration,
    /// 流超时
    pub stream_timeout: Duration,
    /// 心跳间隔
    pub heartbeat_interval: Duration,
}

/// QUIC 网络传输实现
pub struct QuicTransport {
    /// 本地端点
    endpoint: Endpoint,
    /// 活跃连接
    connections: Arc<RwLock<HashMap<AgentId, Connection>>>,
    /// 消息接收器
    message_rx: mpsc::Receiver<(AgentId, Message)>,
    /// 配置
    config: NetworkConfig,
}

impl QuicTransport {
    pub async fn new(config: NetworkConfig) -> Result<Self, NetworkError> {
        // 加载证书
        let (cert, key) = Self::load_certs(&config.cert_path, &config.key_path).await?;
        
        // 配置 TLS
        let tls_config = Self::configure_tls(cert, key)?;
        
        // 创建端点
        let endpoint = Endpoint::server(config.bind_addr, tls_config)?;
        
        let (tx, rx) = mpsc::channel(1000);
        let transport = Self {
            endpoint,
            connections: Arc::new(RwLock::new(HashMap::new())),
            message_rx: rx,
            config,
        };
        
        // 启动连接接受循环
        transport.start_accept_loop(tx).await;
        
        Ok(transport)
    }
    
    /// 连接到远程 Agent
    pub async fn connect(&self, agent_id: AgentId, addr: SocketAddr) -> Result<(), NetworkError> {
        let connection = self.endpoint.connect(addr, "swarm.local")?.await?;
        
        self.connections.write().await.insert(agent_id, connection);
        
        // 启动消息接收循环
        let conn = self
            .connections
            .read()
            .await
            .get(&agent_id)
            .cloned()
            .ok_or(NetworkError::Unreachable(agent_id))?;
        
        self.start_receive_loop(agent_id, conn).await;
        
        Ok(())
    }
    
    /// 发送消息到指定 Agent
    pub async fn send_to(&self, agent_id: AgentId, message: Message) -> Result<(), NetworkError> {
        let conn = self
            .connections
            .read()
            .await
            .get(&agent_id)
            .cloned()
            .ok_or(NetworkError::Unreachable(agent_id))?;
        
        // 打开双向流
        let (mut send, _recv) = conn.open_bi().await?;
        
        // 序列化消息
        let data = serde_json::to_vec(&message)?;
        
        // 发送
        send.write_all(&data).await?;
        send.finish()?;
        
        Ok(())
    }
    
    /// 广播消息
    pub async fn broadcast(&self, message: Message) -> Result<(), NetworkError> {
        let connections = self.connections.read().await;
        
        for (agent_id, _) in connections.iter() {
            // 克隆消息（因为需要多次发送）
            let msg = message.clone();
            let agent_id = *agent_id;
            
            // 并行发送
            let connections = self.connections.clone();
            tokio::spawn(async move {
                if let Some(conn) = connections.read().await.get(&agent_id) {
                    // ... 发送逻辑
                }
            });
        }
        
        Ok(())
    }
    
    /// 接收消息（阻塞）
    pub async fn receive(&mut self) -> Result<(AgentId, Message), NetworkError> {
        self.message_rx
            .recv()
            .await
            .ok_or(NetworkError::Other(anyhow::anyhow!("Channel closed")))
    }
    
    /// 启动连接接受循环
    async fn start_accept_loop(&self, tx: mpsc::Sender<(AgentId, Message)>) {
        while let Some(conn) = self.endpoint.accept().await {
            let tx = tx.clone();
            tokio::spawn(async move {
                match conn.await {
                    Ok(connection) => {
                        // 处理连接
                        Self::handle_connection(connection, tx).await;
                    }
                    Err(e) => {
                        tracing::error!("Connection failed: {}", e);
                    }
                }
            });
        }
    }
    
    /// 处理单个连接
    async fn handle_connection(
        connection: Connection,
        tx: mpsc::Sender<(AgentId, Message)>,
    ) {
        loop {
            match connection.accept_bi().await {
                Ok((send, recv)) => {
                    // 处理双向流
                    tokio::spawn(async move {
                        // 读取数据并转发到消息通道
                        // ...
                    });
                }
                Err(_) => break,
            }
        }
    }
    
    /// 启动消息接收循环
    async fn start_receive_loop(&self, agent_id: AgentId, connection: Connection) {
        // 持续接收消息
        // ...
    }
    
    /// 加载证书
    async fn load_certs(
        cert_path: &std::path::Path,
        key_path: &std::path::Path,
    ) -> Result<(Vec<CertificateDer<'static>>, PrivateKeyDer<'static>), NetworkError> {
        // 实现证书加载
        todo!()
    }
    
    /// 配置 TLS
    fn configure_tls(
        cert: Vec<CertificateDer<'static>>,
        key: PrivateKeyDer<'static>,
    ) -> Result<ServerConfig, NetworkError> {
        // 实现 TLS 配置
        todo!()
    }
}

/// Gossip 协议实现（服务发现 + 状态同步）
pub struct GossipProtocol {
    /// 已知节点
    nodes: Arc<RwLock<HashMap<NodeId, NodeInfo>>>,
    /// 消息序列号（防止重复传播）
    seen_messages: Arc<RwLock<lru::LruCache<Uuid, ()>>>,
    /// 传输层
    transport: Arc<dyn NetworkTransport>,
}

impl GossipProtocol {
    /// 传播消息
    pub async fn gossip(&self, message: GossipMessage) -> Result<(), NetworkError> {
        // 检查是否已处理
        if self.seen_messages.write().await.put(message.id, ()).is_some() {
            return Ok(());  // 已处理，忽略
        }
        
        // 获取随机子集的节点（通常 3-5 个）
        let targets = self.select_gossip_targets(3).await;
        
        // 发送给目标节点
        for node_id in targets {
            // ...
        }
        
        Ok(())
    }
    
    /// 选择 Gossip 目标（随机，但优先未更新的节点）
    async fn select_gossip_targets(&self, count: usize) -> Vec<NodeId> {
        let nodes = self.nodes.read().await;
        let mut candidates: Vec<_> = nodes.keys().copied().collect();
        
        // 随机打乱
        use rand::seq::SliceRandom;
        candidates.shuffle(&mut rand::thread_rng());
        
        candidates.into_iter().take(count).collect()
    }
}

/// Gossip 消息类型
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum GossipMessage {
    /// 节点加入
    NodeJoin(NodeInfo),
    /// 节点心跳
    NodeHeartbeat { node_id: NodeId, timestamp: u64 },
    /// Agent 状态更新
    AgentUpdate(AgentInstance),
    /// 任务状态更新
    TaskUpdate(TaskStatus),
    /// 自定义应用消息
    AppMessage { channel: String, payload: Vec<u8> },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeInfo {
    pub id: NodeId,
    pub address: SocketAddr,
    pub region: String,
    pub capacity: ResourceQuota,
    pub available: ResourceQuota,
}
```

---

### 3.5 openswarm-api - HTTP/gRPC API

**文件位置**: `crates/openswarm-api/src/lib.rs`

```rust
//! HTTP/gRPC API 层
//!
//! 对外暴露 REST API 和 gRPC 服务

use axum::{
    extract::{Path, State, Json},
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post, put, delete},
    Router,
};
use std::sync::Arc;
use tower_http::trace::TraceLayer;

use pswarm_core::*;

/// API 状态
#[derive(Clone)]
pub struct ApiState {
    /// 调度器
    scheduler: Arc<dyn TaskScheduler>,
    /// Agent 生命周期管理
    agent_lifecycle: Arc<dyn AgentLifecycle>,
    /// 配置存储
    storage: Arc<dyn Storage>,
}

/// 创建 API 路由
pub fn create_router(state: ApiState) -> Router {
    Router::new()
        // Agent 管理
        .route("/api/v1/agents", post(create_agent))
        .route("/api/v1/agents", get(list_agents))
        .route("/api/v1/agents/:id", get(get_agent))
        .route("/api/v1/agents/:id", delete(delete_agent))
        .route("/api/v1/agents/:id/start", post(start_agent))
        .route("/api/v1/agents/:id/stop", post(stop_agent))
        .route("/api/v1/agents/:id/pause", post(pause_agent))
        .route("/api/v1/agents/:id/resume", post(resume_agent))
        // 任务管理
        .route("/api/v1/tasks", post(submit_task))
        .route("/api/v1/tasks/:id", get(get_task))
        .route("/api/v1/tasks/:id/cancel", post(cancel_task))
        // 会话管理
        .route("/api/v1/sessions", post(create_session))
        .route("/api/v1/sessions/:id", get(get_session))
        .route("/api/v1/sessions/:id/messages", post(send_message))
        // 系统
        .route("/api/v1/health", get(health_check))
        .route("/api/v1/metrics", get(get_metrics))
        // 中间件
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}

// ==================== Agent API ====================

async fn create_agent(
    State(state): State<ApiState>,
    Json(spec): Json<AgentSpec>,
) -> Result<impl IntoResponse, ApiError> {
    // 验证配置
    spec.validate().map_err(|e| ApiError::BadRequest(e.to_string()))?;
    
    // 创建 Agent
    let instance = state.agent_lifecycle.create(spec).await?;
    
    Ok((StatusCode::CREATED, Json(instance)))
}

async fn list_agents(
    State(state): State<ApiState>,
    // query: Query<ListAgentsQuery>,
) -> Result<impl IntoResponse, ApiError> {
    let filter = AgentFilter::default();  // 从 query 解析
    let agents = state.agent_lifecycle.list_agents(filter).await?;
    
    Ok(Json(agents))
}

async fn get_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    let agent = state.agent_lifecycle.get_status(id).await?;
    
    Ok(Json(agent))
}

async fn delete_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    state.agent_lifecycle.terminate(id).await?;
    
    Ok(StatusCode::NO_CONTENT)
}

async fn start_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    state.agent_lifecycle.start(id).await?;
    
    Ok(StatusCode::OK)
}

async fn stop_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    state.agent_lifecycle.terminate(id).await?;
    
    Ok(StatusCode::OK)
}

async fn pause_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    state.agent_lifecycle.pause(id).await?;
    
    Ok(StatusCode::OK)
}

async fn resume_agent(
    State(state): State<ApiState>,
    Path(id): Path<AgentId>,
) -> Result<impl IntoResponse, ApiError> {
    state.agent_lifecycle.resume(id).await?;
    
    Ok(StatusCode::OK)
}

// ==================== Task API ====================

async fn submit_task(
    State(state): State<ApiState>,
    Json(task): Json<SubmitTaskRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let task = Task {
        id: TaskId::new_v4(),
        parent_id: task.parent_id,
        task_type: task.task_type,
        input: task.input,
        priority: task.priority.unwrap_or(TaskPriority::Normal),
        timeout: task.timeout.unwrap_or(Duration::from_secs(300)),
        required_capabilities: task.required_capabilities,
        labels: task.labels.unwrap_or_default(),
        created_at: SystemTime::now(),
    };
    
    let task_id = state.scheduler.submit(task).await?;
    
    Ok((StatusCode::CREATED, Json(SubmitTaskResponse { task_id })))
}

async fn get_task(
    State(state): State<ApiState>,
    Path(id): Path<TaskId>,
) -> Result<impl IntoResponse, ApiError> {
    let status = state.scheduler.get_status(id).await?;
    
    Ok(Json(status))
}

async fn cancel_task(
    State(state): State<ApiState>,
    Path(id): Path<TaskId>,
) -> Result<impl IntoResponse, ApiError> {
    state.scheduler.cancel(id).await?;
    
    Ok(StatusCode::OK)
}

// ==================== Session API ====================

async fn create_session(
    State(state): State<ApiState>,
    Json(req): Json<CreateSessionRequest>,
) -> Result<impl IntoResponse, ApiError> {
    let session = Session {
        id: SessionId::new_v4(),
        root_task_id: req.initial_task_id,
        agents: vec![req.initial_agent_id],
        current_agent: Some(req.initial_agent_id),
        messages: vec![],
        context_variables: req.context_variables.unwrap_or_default(),
        created_at: SystemTime::now(),
        updated_at: SystemTime::now(),
    };
    
    state.storage.save_session(&session).await?;
    
    Ok((StatusCode::CREATED, Json(session)))
}

async fn get_session(
    State(state): State<ApiState>,
    Path(id): Path<SessionId>,
) -> Result<impl IntoResponse, ApiError> {
    let session = state.storage.get_session(id).await?;
    
    Ok(Json(session))
}

async fn send_message(
    State(state): State<ApiState>,
    Path(id): Path<SessionId>,
    Json(req): Json<SendMessageRequest>,
) -> Result<impl IntoResponse, ApiError> {
    // 获取会话
    let mut session = state.storage.get_session(id).await?;
    
    // 创建消息
    let message = Message {
        id: Uuid::new_v4(),
        sender: req.sender,
        recipient: req.recipient,
        message_type: req.message_type,
        content: req.content,
        timestamp: SystemTime::now(),
    };
    
    // 添加到会话
    session.messages.push(message.clone());
    session.updated_at = SystemTime::now();
    
    // 保存
    state.storage.save_session(&session).await?;
    
    // 转发到目标 Agent
    if let Some(recipient) = req.recipient {
        // 解析 recipient 为 AgentId
        let agent_id = AgentId::parse_str(&recipient)?;
        // 通过传输层发送
        // ...
    }
    
    Ok(Json(message))
}

// ==================== System API ====================

async fn health_check() -> impl IntoResponse {
    Json(serde_json::json!({
        "status": "healthy",
        "version": env!("CARGO_PKG_VERSION"),
        "timestamp": chrono::Utc::now().to_rfc3339(),
    }))
}

async fn get_metrics() -> impl IntoResponse {
    // 返回 Prometheus 格式的指标
    "# metrics here"
}

// ==================== 请求/响应类型 ====================

#[derive(Debug, Deserialize)]
struct SubmitTaskRequest {
    parent_id: Option<TaskId>,
    task_type: String,
    input: serde_json::Value,
    priority: Option<TaskPriority>,
    timeout: Option<Duration>,
    required_capabilities: Vec<String>,
    labels: Option<HashMap<String, String>>,
}

#[derive(Debug, Serialize)]
struct SubmitTaskResponse {
    task_id: TaskId,
}

#[derive(Debug, Deserialize)]
struct CreateSessionRequest {
    initial_task_id: TaskId,
    initial_agent_id: AgentId,
    context_variables: Option<HashMap<String, serde_json::Value>>,
}

#[derive(Debug, Deserialize)]
struct SendMessageRequest {
    sender: String,
    recipient: Option<String>,
    message_type: MessageType,
    content: serde_json::Value,
}

// ==================== 错误处理 ====================

#[derive(Debug, thiserror::Error)]
enum ApiError {
    #[error("Bad request: {0}")]
    BadRequest(String),
    
    #[error("Not found")]
    NotFound,
    
    #[error(transparent)]
    Agent(#[from] AgentError),
    
    #[error(transparent)]
    Scheduler(#[from] SchedulerError),
    
    #[error(transparent)]
    Storage(#[from] StorageError),
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> axum::response::Response {
        let (status, body) = match self {
            ApiError::BadRequest(msg) => (
                StatusCode::BAD_REQUEST,
                Json(serde_json::json!({ "error": msg })),
            ),
            ApiError::NotFound => (
                StatusCode::NOT_FOUND,
                Json(serde_json::json!({ "error": "Not found" })),
            ),
            ApiError::Agent(e) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({ "error": e.to_string() })),
            ),
            ApiError::Scheduler(e) => (
                StatusCode::SERVICE_UNAVAILABLE,
                Json(serde_json::json!({ "error": e.to_string() })),
            ),
            ApiError::Storage(e) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({ "error": e.to_string() })),
            ),
            ApiError::Other(e) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({ "error": e.to_string() })),
            ),
        };
        
        (status, body).into_response()
    }
}
```

---

## 4. 开发规范

### 4.1 代码风格

- 使用 `cargo fmt` 统一格式化
- 使用 `cargo clippy` 检查代码
- 文档注释使用 `///` 和 `//!`
- 错误处理使用 `thiserror` 或 `anyhow`

### 4.2 测试规范

```rust
// 单元测试
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_scheduler_submit() {
        let scheduler = create_test_scheduler().await;
        let task = create_test_task();
        
        let task_id = scheduler.submit(task).await.unwrap();
        
        assert!(!task_id.is_nil());
    }
    
    #[tokio::test]
    async fn test_agent_lifecycle() {
        let manager = create_test_manager().await;
        let spec = create_test_spec();
        
        let agent = manager.create(spec).await.unwrap();
        assert_eq!(agent.state, AgentState::Creating);
        
        manager.start(agent.id).await.unwrap();
        let status = manager.get_status(agent.id).await.unwrap();
        assert_eq!(status.state, AgentState::Idle);
        
        manager.terminate(agent.id).await.unwrap();
    }
}

// 集成测试
#[tokio::test]
async fn test_end_to_end_task_execution() {
    // 启动完整系统
    let (scheduler, agent_mgr) = start_test_cluster().await;
    
    // 创建 Agent
    let spec = AgentSpec {
        // ...
    };
    let agent = agent_mgr.create(spec).await.unwrap();
    
    // 提交任务
    let task = Task {
        // ...
    };
    let task_id = scheduler.submit(task).await.unwrap();
    
    // 等待完成
    let status = scheduler.wait_for_completion(task_id).await.unwrap();
    
    assert_eq!(status.state, TaskState::Completed);
}
```

### 4.3 日志规范

```rust
use tracing::{info, warn, error, debug, span, Level};

// 结构化日志
info!(
    agent_id = %agent_id,
    task_id = %task_id,
    duration_ms = execution_time,
    "Task completed successfully"
);

// Span 追踪
let span = span!(Level::INFO, "agent_execution", agent_id = %agent_id);
let _enter = span.enter();
```

---

## 5. 部署架构

### 5.1 单机部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  openswarm-api:
    image: paperclip/pswarm:latest
    command: ["api-server"]
    ports:
      - "8080:8080"
    environment:
      - RUST_LOG=info
      - DATABASE_URL=postgres://pswarm:secret@postgres:5432/pswarm
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  openswarm-worker:
    image: paperclip/pswarm:latest
    command: ["worker"]
    environment:
      - RUST_LOG=info
      - API_ENDPOINT=http://openswarm-api:8080
    deploy:
      replicas: 2

  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: pswarm
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: pswarm
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 5.2 Kubernetes 部署

```yaml
# deployments/k8s/operator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openswarm-operator
  namespace: openswarm-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openswarm-operator
  template:
    metadata:
      labels:
        app: openswarm-operator
    spec:
      serviceAccountName: openswarm-operator
      containers:
        - name: operator
          image: paperclip/openswarm-operator:latest
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
          env:
            - name: WATCH_NAMESPACE
              value: ""
---
# CRD 定义
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: agents.pswarm.io
spec:
  group: pswarm.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                runtime:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: [wasm, container]
                    image:
                      type: string
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                    memory:
                      type: string
  scope: Namespaced
  names:
    plural: agents
    singular: agent
    kind: Agent
    shortNames:
      - ag
```

---

## 6. 开发任务分解

### Phase 1: 基础框架（2周）

| 任务 | 负责人 | 工作量 | 依赖 |
|------|--------|--------|------|
| 项目脚手架搭建 | TBD | 1天 | - |
| openswarm-core 类型定义 | TBD | 2天 | - |
| openswarm-runtime WASM 基础 | TBD | 3天 | openswarm-core |
| openswarm-network QUIC 基础 | TBD | 3天 | openswarm-core |
| 集成测试框架 | TBD | 1天 | 上述全部 |

### Phase 2: 编排能力（2周）

| 任务 | 负责人 | 工作量 | 依赖 |
|------|--------|--------|------|
| openswarm-scheduler 实现 | TBD | 4天 | openswarm-core |
| openswarm-discovery 服务发现 | TBD | 3天 | openswarm-network |
| Agent 生命周期管理 | TBD | 3天 | openswarm-runtime |
| Handoff 协议实现 | TBD | 2天 | openswarm-scheduler |

### Phase 3: API 与存储（2周）

| 任务 | 负责人 | 工作量 | 依赖 |
|------|--------|--------|------|
| openswarm-storage 实现 | TBD | 3天 | openswarm-core |
| openswarm-api REST 接口 | TBD | 3天 | openswarm-scheduler |
| gRPC 接口 | TBD | 2天 | openswarm-api |
| CLI 工具 | TBD | 2天 | openswarm-api |

### Phase 4: 可观测性与部署（2周）

| 任务 | 负责人 | 工作量 | 依赖 |
|------|--------|--------|------|
| OpenTelemetry 集成 | TBD | 2天 | 全部 |
| Prometheus 指标 | TBD | 2天 | 全部 |
| Web UI Console | TBD | 4天 | openswarm-api |
| K8s Operator | TBD | 3天 | openswarm-api |
| Helm Chart | TBD | 1天 | K8s Operator |

---

## 7. 附录

### 7.1 术语表

| 术语 | 说明 |
|------|------|
| Agent | 执行任务的智能体实例 |
| Handoff | Agent 间的任务交接机制 |
| Session | 多轮交互的上下文会话 |
| WASM | WebAssembly，轻量级运行时 |
| QUIC | 基于 UDP 的安全传输协议 |
| Gossip | 去中心化的消息传播协议 |

### 7.2 参考资源

- [OpenAI Swarm](https://github.com/openai/swarm)
- [QUIC RFC 9000](https://datatracker.ietf.org/doc/html/rfc9000)
- [Wasmtime Docs](https://docs.wasmtime.dev/)
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial)

---

**文档结束**

如需任何部分更详细的说明，请联系架构团队。
