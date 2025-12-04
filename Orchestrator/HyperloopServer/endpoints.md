## 开放接口

###  /me

**GET**\

#### 作用

获取沙箱信息

#？ 返回sandboxId

- findSandbox()

```go
// packages/orchestrator/internal/hyperloopserver/handlers/store.go
for _, sbx := range h.sandboxes.Items() {  
    if sbx.Slot.HostIPString() == reqIP {  // 根据请求的远程 IP 地址查找对应的沙箱
       return sbx, nil  
    }  
}
```

#### 调用方

由沙箱内部的用户应用程序 调用，如Node.js / Python / 任意代码想知道自己的 sandboxID    

### /logs

**POST**\

#### 作用

接收存储沙箱日志

1. findSandbox() 获取沙箱ID
2. log

```go
if err := c.ShouldBindJSON(&payload);  ...  // 检查请求体格式并存入payload

payload["instanceID"] = sbxID  
payload["teamID"] = sbx.Runtime.TeamID

logs, err := json.Marshal(payload) // 序列化
```

TeamID 即租户id

1. 发送至日志收集器\
   目标地址`APIStore.collectorAddr`

```go
request, err := http.NewRequestWithContext(c, http.MethodPost, h.collectorAddr, bytes.NewBuffer(logs))

request.Header.Set("Content-Type", "application/json")  
response, err := h.collectorClient.Do(request)
```



#### 调用方

由 envd 系统自动调用，用户无感知

- envd 启动时创建 HTTPExporter，它实现了 io.Writer 接口

- 所有通过 zerolog 记录的日志都会被拦截

- HTTPExporter 自动将日志 POST 到 LogsCollectorAddress（hyperloop 的 /logs 端点）

#### collectorAddr**来源：**

```54:56:packages/shared/pkg/env/env.go
func LogsCollectorAddress() string {
	return os.Getenv("LOGS_COLLECTOR_ADDRESS")
}
```

从环境变量 `LOGS_COLLECTOR_ADDRESS` 读取。

它指向运行在每个节点上的 **Vector 日志收集器**：

```44:55:iac/provider-gcp/nomad/jobs/logs-collector.hcl
    task "start-collector" {
      driver = "docker"

      config {
        network_mode = "host"
        image        = "timberio/vector:0.34.X-alpine"

        ports = [
          "health",
          "logs",
        ]
      }
```

**架构流程：**

```
沙箱内应用 
   ↓ (HTTP POST)
Hyperloop Server (/logs 端点) 
   ↓ (验证 + 添加 TeamID/SandboxID)
Vector 日志收集器 (LOGS_COLLECTOR_ADDRESS)
   ↓
日志存储系统 (ClickHouse/S3 等)
```

典型的**多租户 SaaS 架构**，通过 Hyperloop Server 作为信任边界，确保日志的可靠性和安全性