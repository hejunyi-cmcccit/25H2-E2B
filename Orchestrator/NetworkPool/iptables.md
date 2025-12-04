## iptables

### iptables 基础概念

iptables 有几个"表"，每个表有不同的"链"：

| 表 (Table) | 链 (Chain)   | 作用               |
| ---------- | ------------ | ------------------ |
| nat        | PREROUTING   | 数据包进入前修改   |
| nat        | POSTROUTING  | 数据包离开后修改   |
| filter     | FORWARD      | 转发的数据包是否允许 |
| filter     | INPUT        | 进入本机的数据包   |
| filter     | OUTPUT       | 离开本机的数据包   |

### 代码中的规则详解

#### 规则 1：SNAT（源地址转换）

```go
tables.Append("nat", "POSTROUTING", 
    "-o", s.VpeerName(),           // 从 vpeer 接口出去的包
    "-s", s.NamespaceIP(),         // 源地址是沙箱 IP
    "-j", "SNAT",                  // 动作：修改源地址
    "--to", s.HostIPString())      // 改成：宿主 IP
```

**场景**：沙箱内的应用访问外部  
**为什么**：外部网络不认识沙箱的内部 IP，必须伪装成宿主 IP

#### 规则 2：DNAT（目标地址转换）

```go
tables.Append("nat", "PREROUTING",
    "-i", s.VpeerName(),           // 从 vpeer 接口进来的包
    "-d", s.HostIPString(),        // 目标地址是宿主 IP
    "-j", "DNAT",                  // 动作：修改目标地址
    "--to", s.NamespaceIP())       // 改成：沙箱 IP
```

**场景**：外部访问沙箱内的服务  
**为什么**：把发给宿主 IP 的包转发到沙箱内部

#### 规则 3：允许从沙箱到互联网的转发

```go
tables.Append("filter", "FORWARD",
    "-i", s.VethName(),            // 从 veth 接口进来
    "-o", defaultGateway,          // 从默认网关出去 (通常是 eth0)
    "-j", "ACCEPT")                // 动作：允许
```

#### 规则 4：允许从互联网到沙箱的转发

```go
tables.Append("filter", "FORWARD",
    "-i", defaultGateway,          // 从默认网关进来
    "-o", s.VethName(),            // 从 veth 出去
    "-j", "ACCEPT")                // 动作：允许
```

**为什么**：Linux 默认不转发数据包，必须明确允许

#### 规则 5：MASQUERADE（IP 伪装）

```go
tables.Append("nat", "POSTROUTING",
    "-s", s.HostCIDR(),            // 源地址是沙箱网段 (如 10.0.0.0/24)
    "-o", defaultGateway,          // 从默认网关出去
    "-j", "MASQUERADE")            // 动作：动态 SNAT
```

**MASQUERADE vs SNAT**：
- `SNAT`：改成固定 IP
- `MASQUERADE`：改成出口网卡的当前 IP（适合 DHCP 等动态 IP 环境）

#### 规则 6：Hyperloop 代理重定向

```go
tables.Append("nat", "PREROUTING",
    "-i", s.VethName(),                // 从 veth 进来
    "-p", "tcp",                       // 协议：TCP
    "-d", s.HyperloopIPString(),       // 目标：Hyperloop IP
    "--dport", "80",                   // 目标端口：80 (HTTP)
    "-j", "REDIRECT",                  // 动作：重定向
    "--to-port", s.hyperloopPort)      // 重定向到：本地代理端口
```

**场景**：拦截 HTTP 请求  
**为什么**：可能是为了监控、缓存或修改 HTTP 流量

```
沙箱应用: curl http://特定IP:80
     ↓ REDIRECT 规则
实际访问: localhost:本地代理端口
```

