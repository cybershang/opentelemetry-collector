# OpenTelemetry Collector 架构概览

[English](../README.md) | 中文

## 概述

OpenTelemetry Collector 是一个供应商中立的遥测数据收集器，用于接收、处理和导出遥测数据。它消除了运行、操作和维护多个代理/收集器的需要，以支持开源遥测数据格式（如 Jaeger、Prometheus 等）到多个开源或商业后端。

## 设计目标

- **易用性**: 合理的默认配置，支持流行协议，开箱即用
- **高性能**: 在不同负载和配置下高度稳定和高性能
- **可观测性**: 可观测服务的典范
- **可扩展性**: 无需修改核心代码即可自定义
- **统一性**: 单一代码库，可部署为代理或收集器，支持链路追踪、指标和日志

## 架构组件

### 核心组件类型

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Receivers  │    │ Processors  │    │  Exporters  │
│   接收器     │────│   处理器     │────│   导出器     │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       ▼                   ▼                   ▼
   接收数据              处理数据            导出数据
```

#### 接收器（Receivers）
- 从各种来源接收遥测数据
- 支持多种协议（OTLP、Jaeger、Zipkin、Prometheus 等）
- 将外部格式转换为内部数据模型

#### 处理器（Processors）
- 对遥测数据进行转换、过滤、聚合等处理
- 批处理、采样、属性修改、资源检测等
- 可链式组合多个处理器

#### 导出器（Exporters）
- 将处理后的数据发送到目标系统
- 支持多种后端（Jaeger、Prometheus、Elasticsearch、云服务等）
- 处理重试、批处理和错误处理

#### 扩展（Extensions）
- 提供额外功能，不直接处理遥测数据
- 健康检查、性能分析、认证等
- 独立于管道运行

#### 连接器（Connectors）
- 连接多个管道的特殊组件
- 同时充当导出器和接收器的角色
- 支持复杂的数据路由场景

### 管道架构

```
管道示例：
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   OTLP       │   │   Batch      │   │ Attributes   │   │   Jaeger     │
│  Receiver    │──▶│  Processor   │──▶│  Processor   │──▶│   Exporter   │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘

支持的信号类型：
• Traces（链路追踪）
• Metrics（指标）  
• Logs（日志）
• Profiles（性能分析，实验性）
```

## 配置系统

Collector 使用声明式 YAML 配置：

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
  attributes:
    actions:
      - key: environment
        value: "production"
        action: insert

exporters:
  jaeger:
    endpoint: jaeger:14250

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [attributes, batch]
      exporters: [jaeger]
```

### 配置特性

- **多源配置**: 支持文件、环境变量、远程配置
- **配置合并**: 自动合并多个配置源
- **热重载**: 支持配置变更的热重载
- **验证**: 严格的配置验证和错误报告

## 内部架构

### 启动流程

1. **命令行解析**: 解析启动参数和配置文件路径
2. **配置加载**: 从多个源加载和合并配置
3. **组件创建**: 使用工厂模式创建组件实例
4. **图构建**: 构建组件依赖图
5. **组件启动**: 按拓扑顺序启动组件
6. **服务运行**: 开始处理数据流

### 数据流

```
外部数据源 → 接收器 → 处理器链 → 导出器 → 目标系统
                ↑         ↑         ↑
              协议解析   数据处理   格式转换
```

### 内存管理

- **零拷贝**: 尽可能避免数据拷贝
- **对象池**: 重用数据结构减少 GC 压力
- **批处理**: 优化网络和存储 I/O

## 可观测性

### 内部监控

Collector 本身是高度可观测的：

- **内部指标**: 组件性能、错误率、数据流量
- **健康检查**: 组件状态和整体健康状况
- **日志记录**: 结构化日志，支持多级别
- **分布式追踪**: 内部操作的追踪

### 监控指标

- 接收、处理、导出的数据量
- 组件延迟和错误率
- 队列深度和背压
- 内存和 CPU 使用情况

## 部署模式

### Agent 模式
```
应用 → Collector (Agent) → Collector (Gateway) → 后端
```
- 与应用同主机部署
- 轻量级，低延迟
- 本地缓冲和基本处理

### Gateway 模式
```
多个 Agent → Collector (Gateway) → 多个后端
```
- 集中式部署
- 高级处理和路由
- 跨租户数据隔离

### Standalone 模式
```
数据源 → Collector (Standalone) → 后端
```
- 独立部署
- 完整功能集
- 适合简单场景

## 扩展开发

### 组件开发

开发自定义组件需要：

1. 实现对应的接口（Receiver、Processor、Exporter）
2. 提供工厂函数
3. 定义配置结构
4. 实现生命周期方法

### 工厂模式

```go
type Factory interface {
    Type() component.Type
    CreateDefaultConfig() component.Config
    CreateTraces(context.Context, Settings, component.Config, consumer.Traces) (component.Component, error)
    // ... 其他 Create 方法
}
```

## 社区和支持

- **GitHub**: [opentelemetry-collector](https://github.com/open-telemetry/opentelemetry-collector)
- **文档**: [OpenTelemetry 官方文档](https://opentelemetry.io/docs/collector/)
- **Slack**: [#otel-collector](https://cloud-native.slack.com/archives/C01N6P7KR6W)
- **会议**: 每周社区会议

## 相关文档

- [内部架构详解](internal-architecture-zh.md)
- [组件开发指南](https://opentelemetry.io/docs/collector/custom-components/)
- [配置参考](https://opentelemetry.io/docs/collector/configuration/)
- [部署指南](https://opentelemetry.io/docs/collector/deployment/)

---

*本文档提供了 OpenTelemetry Collector 的中文架构概览。如需了解更多详细信息，请参阅相关链接文档。*