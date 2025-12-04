## SandboxService

所有 6 个 gRPC 方法的实现位置：

- 📁 `packages/orchestrator/internal/server/sandboxes.go` - Create, Update, List, Delete, Pause
  - 创建新的沙箱，和恢复暂停的沙箱，都是Create接口


- 📁 `packages/orchestrator/internal/server/template_cache.go` - ListCachedBuilds

它们通过 `Server` 结构体实现 `SandboxServiceServer` 接口，在 `main.go` 中被注册到 gRPC 服务器上。



## TemplateService

### 1. **TemplateCreate** 创建模板

**功能：**创建新模板，支持从 Docker 镜像或已有模板构建，配置资源（CPU、内存、磁盘）和执行步骤，后台异步构建。

### 2. **TemplateBuildStatus** 端点

**功能：** 查询模板构建状态，返回构建状态（Building/Failed/Completed）、日志条目、失败原因和元数据，支持分页和日志级别过滤。

### 3. **TemplateBuildDelete** 端点

**功能：** 删除模板构建，如果构建正在进行中会先取消，然后清理相关文件和存储。

### 4. **InitLayerFileUpload** 端点

**功能：** 初始化层文件上传，返回预签名上传 URL 和文件是否已存在的状态，用于优化模板构建时的文件缓存（30 分钟有效期）。



## InfoService

```proto
service InfoService {
  rpc ServiceInfo(google.protobuf.Empty) returns (ServiceInfoResponse);
  rpc ServiceStatusOverride(ServiceStatusChangeRequest) returns (google.protobuf.Empty);
```

### 1. **ServiceInfo** 端点

**实现代码：**

```go
func (s *Server) ServiceInfo(_ context.Context, _ *emptypb.Empty) (*orchestratorinfo.ServiceInfoResponse, error) {
	info := s.info

	// Get host metrics for the orchestrator
	cpuMetrics, err := metrics.GetCPUMetrics()
	if err != nil {
		zap.L().Warn("Failed to get host metrics", zap.Error(err))
		cpuMetrics = &metrics.CPUMetrics{}
	}

	memoryMetrics, err := metrics.GetMemoryMetrics()
	if err != nil {
		zap.L().Warn("Failed to get host metrics", zap.Error(err))
		memoryMetrics = &metrics.MemoryMetrics{}
	}

	diskMetrics, err := metrics.GetDiskMetrics()
	if err != nil {
		zap.L().Warn("Failed to get host metrics", zap.Error(err))
		diskMetrics = []metrics.DiskInfo{}
	}

	// Calculate sandbox resource allocation
	sandboxVCpuAllocated := uint32(0)
	sandboxMemoryAllocated := uint64(0)
	sandboxDiskAllocated := uint64(0)

	for _, item := range s.sandboxes.Items() {
		sandboxVCpuAllocated += uint32(item.Config.Vcpu)
		sandboxMemoryAllocated += uint64(item.Config.RamMB) * 1024 * 1024
		sandboxDiskAllocated += uint64(item.Config.TotalDiskSizeMB) * 1024 * 1024
	}

	return &orchestratorinfo.ServiceInfoResponse{
		NodeId:        info.ClientId,
		ServiceId:     info.ServiceId,
		ServiceStatus: info.GetStatus(),

		ServiceVersion: info.SourceVersion,
		ServiceCommit:  info.SourceCommit,

		ServiceStartup: timestamppb.New(info.Startup),
		ServiceRoles:   info.Roles,

		// Allocated resources to sandboxes
		MetricCpuAllocated:         sandboxVCpuAllocated,
		MetricMemoryAllocatedBytes: sandboxMemoryAllocated,
		MetricDiskAllocatedBytes:   sandboxDiskAllocated,
		MetricSandboxesRunning:     uint32(s.sandboxes.Count()),

		// Host system usage metrics
		MetricCpuPercent:      uint32(cpuMetrics.UsedPercent),
		MetricMemoryUsedBytes: memoryMetrics.UsedBytes,

		// Host system total resources
		MetricCpuCount:         cpuMetrics.Count,
		MetricMemoryTotalBytes: memoryMetrics.TotalBytes,

		// Detailed disk metrics
		MetricDisks: convertDiskMetrics(diskMetrics),

		// TODO: Remove when migrated
		MetricVcpuUsed:     int64(sandboxVCpuAllocated),
		MetricMemoryUsedMb: int64(sandboxMemoryAllocated / (1024 * 1024)),
		MetricDiskMb:       int64(sandboxDiskAllocated / (1024 * 1024)),
	}, nil
}
```

**功能：** 获取编排器服务的详细信息，包括：
- 节点和服务标识信息
- 服务状态、版本和启动时间
- 主机系统指标（CPU、内存、磁盘）
- 沙箱资源分配情况
- 当前运行的沙箱数量

### 2. **ServiceStatusOverride** 端点

```go
func (s *Server) ServiceStatusOverride(_ context.Context, req *orchestratorinfo.ServiceStatusChangeRequest) (*emptypb.Empty, error) {
	zap.L().Info("service status override request received", zap.String("status", req.GetServiceStatus().String()))
	s.info.SetStatus(req.GetServiceStatus())

	return &emptypb.Empty{}, nil
}
```

**功能：** 手动覆盖服务状态（Healthy、Draining 或 Unhealthy），用于维护和排空操作。

这两个端点共同提供了对编排器服务状态的监控和管理能力。



## HealthService

grpc标准的健康检查服务

// todo

