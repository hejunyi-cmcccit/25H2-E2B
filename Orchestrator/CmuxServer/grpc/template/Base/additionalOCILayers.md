## 函数位置

packages\orchestrator\internal\template\build\core\rootfs\rootfs.go:180

## 函数功能概述

```180:186:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
func additionalOCILayers(
	_ context.Context,
	config config.TemplateConfig,
	provisionScript string,
	provisionLogPrefix string,
	provisionResultPath string,
) ([]containerregistry.Layer, error) {
```

**作用**：在 Docker 基础镜像之上添加两个自定义层，包含 E2B 运行所需的所有系统文件；这个函数是 E2B 系统的"定制化层"，将通用的 Docker 镜像改造成可以运行 E2B 沙箱的完整系统

## 详细步骤分析

### ⚙️ 第 1 步：配置 envd 服务（187-206 行）

```187:206:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
memoryLimit := int(math.Min(float64(config.MemoryMB)/2, 512))
envdService := fmt.Sprintf(`[Unit]
Description=Env Daemon Service
After=multi-user.target

[Service]
Type=simple
Restart=always
User=root
Group=root
Environment=GOTRACEBACK=all
LimitCORE=infinity
ExecStart=/bin/bash -l -c "/usr/bin/envd"
OOMPolicy=continue
OOMScoreAdjust=-1000
Environment="GOMEMLIMIT=%dMiB"

[Install]
WantedBy=multi-user.target
`, memoryLimit)
```

**envd 服务配置解析**：

- **memoryLimit**：envd 的内存限制 = min(总内存/2, 512MB)
  - 如果虚拟机有 1GB 内存，envd 限制为 512MB
  - 如果虚拟机有 256MB 内存，envd 限制为 128MB
  
- **systemd 服务配置**：
  - `Type=simple`：标准守护进程
  - `Restart=always`：崩溃后自动重启
  - `User=root`：以 root 权限运行
  - `GOTRACEBACK=all`：Go 程序崩溃时输出完整堆栈
  - `LimitCORE=infinity`：允许生成 core dump 文件（用于调试）
  - `OOMPolicy=continue`：内存不足时继续运行
  - `OOMScoreAdjust=-1000`：**最重要**：降低被 OOM Killer 杀死的优先级（保护 envd）
  - `GOMEMLIMIT`：Go 运行时的内存限制

### 🔐 第 2 步：配置自动登录（208-211 行）

```208:211:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
autologinService := `[Service]
ExecStart=
ExecStart=-/sbin/agetty --noissue --autologin root %I 115200,38400,9600 vt102
`
```

- **agetty**：管理终端登录的程序
- `--autologin root`：自动以 root 用户登录（无需密码）
- `--noissue`：不显示 issue 文件（登录提示信息）
- 这样虚拟机启动后直接进入 shell，无需手动登录

### 🌐 第 3 步：配置主机名和 DNS（213-223 行）

```213:222:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
hostname := "e2b.local"

hosts := fmt.Sprintf(`127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::	ip6-localnet
ff00::	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
127.0.1.1	%s
`, hostname)
```

- 设置主机名为 `e2b.local`
- 配置标准的 `/etc/hosts` 文件（IPv4 和 IPv6 映射）

### 📦 第 4 步：读取 envd 二进制文件（224-227 行）

```224:227:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
envdFileData, err := os.ReadFile(storage.HostEnvdPath())
if err != nil {
	return nil, fmt.Errorf("error reading envd file: %w", err)
}
```

- 从宿主机读取 envd 二进制文件
- envd 是 E2B 的核心守护进程，用 Go 编写
- 这个文件会被复制到虚拟机的 `/usr/bin/envd`

### 📝 第 5 步：创建文件层（229-281 行）

这是最核心的部分，创建包含所有配置文件的 OCI 层：

#### 🌍 **网络配置**（232-234 行）

```232:234:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
// Setup system
"etc/hostname":    {Bytes: []byte(hostname), Mode: 0o644},
"etc/hosts":       {Bytes: []byte(hosts), Mode: 0o644},
"etc/resolv.conf": {Bytes: []byte("nameserver 8.8.8.8"), Mode: 0o644},
```

- 主机名设置
- 主机映射
- DNS 服务器（使用 Google DNS）

#### 🚀 **envd 守护进程**（236-237 行）

```236:237:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
storage.GuestEnvdPath:                                            {Bytes: envdFileData, Mode: 0o777},
"etc/systemd/system/envd.service":                                {Bytes: []byte(envdService), Mode: 0o644},
```

- `/usr/bin/envd`：envd 可执行文件
- `/etc/systemd/system/envd.service`：systemd 服务定义

#### 🖥️ **终端和系统服务配置**（238-240 行）

```238:240:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
"etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf": {Bytes: []byte(autologinService), Mode: 0o644},
"etc/systemd/system/systemd-journald.service.d/override.conf":    {Bytes: []byte(serviceWatchDogDisabledConfig), Mode: 0o644},
"etc/systemd/system/systemd-networkd.service.d/override.conf":    {Bytes: []byte(serviceWatchDogDisabledConfig), Mode: 0o644},
```

- **serial-getty@ttyS0**：串口终端自动登录配置
- **systemd-journald**：日志服务，禁用 watchdog（看门狗定时器）
- **systemd-networkd**：网络服务，禁用 watchdog

**为什么禁用 watchdog？** 在虚拟机环境中，watchdog 可能误判服务卡住而重启服务。

#### 📜 **Provision 脚本**（242-243 行）

```242:243:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
// Provision script
"usr/local/bin/provision.sh": {Bytes: []byte(provisionScript), Mode: 0o777},
```

- 用户自定义的初始化脚本
- 首次启动时运行（安装软件包、配置环境等）

#### 🔧 **BusyBox 初始化系统**（244-261 行）

```245:261:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
BusyBoxPath: {Bytes: systeminit.BusyboxBinary, Mode: 0o755},
// Set to bin/init so it's not in conflict with systemd
// Any rewrite of the init file when booted from it will corrupt the filesystem
BusyBoxInitPath: {Bytes: systeminit.BusyboxBinary, Mode: 0o755},
"etc/init.d/rcS": {Bytes: []byte(`#!/usr/bin/busybox ash
echo "Mounting essential filesystems"
# Ensure necessary mount points exist
mkdir -p /proc /sys /dev /tmp /run

# Mount essential filesystems
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /tmp
mount -t tmpfs tmpfs /run

echo "System Init"`), Mode: 0o777},
```

**BusyBox 的作用**：
- 精简的 Unix 工具集（包含 sh, mount, mkdir 等命令）
- 用于初始化阶段（在 systemd 启动之前）
- `/usr/bin/busybox`：工具集
- `/usr/bin/init`：作为 PID 1 的初始化进程
- `/etc/init.d/rcS`：系统初始化脚本，挂载必要的文件系统

**挂载的文件系统**：
- `/proc`：进程信息伪文件系统
- `/sys`：内核和设备信息
- `/dev`：设备文件
- `/tmp`：临时文件
- `/run`：运行时数据

#### ⚡ **inittab 配置**（262-276 行）

```262:276:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
"etc/inittab": {Bytes: fmt.Appendf(nil, `# Run system init
::sysinit:/etc/init.d/rcS

# Run the provision script, prefix the output with a log prefix
::wait:/bin/sh -c '/usr/local/bin/provision.sh 2>&1 | sed "s/^/%s/"'

# Flush filesystem changes to disk
::wait:/usr/bin/busybox sync

# Report the exit code of the provisioning script
::wait:/bin/sh -c 'echo "%s$(cat %s || printf 1)"'

# Wait forever to prevent the VM from exiting until the sandbox is paused and snapshot is taken
::wait:/usr/bin/busybox sleep infinity
`, provisionLogPrefix, ProvisioningExitPrefix, provisionResultPath), Mode: 0o777},
```

**inittab 启动流程**（按顺序执行）：

1. **`::sysinit:/etc/init.d/rcS`**
   - 系统初始化：挂载文件系统

2. **`::wait:/usr/local/bin/provision.sh`**
   - 运行用户的 provision 脚本
   - `2>&1`：将错误输出重定向到标准输出
   - `sed "s/^/%s/"`：给每行输出添加前缀（用于日志分类）

3. **`::wait:/usr/bin/busybox sync`**
   - 同步所有文件系统更改到磁盘
   - 确保数据持久化

4. **`::wait:/bin/sh -c 'echo "E2B_PROVISIONING_EXIT:..."'`**
   - 报告 provision 脚本的退出码
   - 如果成功读取退出码文件，输出退出码；否则输出 1（失败）

5. **`::wait:/usr/bin/busybox sleep infinity`**
   - **关键**：永久休眠，防止虚拟机退出
   - 此时系统会被暂停并创建快照
   - 后续启动沙箱时从这个快照恢复

### 🔗 第 6 步：创建符号链接层（283-293 行）

```283:293:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
symlinkLayer, err := oci.LayerSymlink(
	map[string]string{
		// Enable envd service autostart
		"etc/systemd/system/multi-user.target.wants/envd.service": "etc/systemd/system/envd.service",
		// Enable chrony service autostart
		"etc/systemd/system/multi-user.target.wants/chrony.service": "etc/systemd/system/chrony.service",
	},
)
if err != nil {
	return nil, fmt.Errorf("error creating layer from symlinks: %w", err)
}
```

**systemd 服务自启动**：
- 通过在 `multi-user.target.wants/` 目录创建符号链接
- **envd.service**：E2B 核心守护进程（必须启动）
- **chrony.service**：时间同步服务（保持系统时间准确）

### ✅ 第 7 步：返回两个层（295-298 行）

```295:298:d:\github\e2b\infra\packages\orchestrator\internal\template\build\core\rootfs\rootfs.go
return []containerregistry.Layer{
	filesLayer,
	symlinkLayer,
}, nil
```

返回两个 OCI 层：
1. **文件层**：包含所有配置文件和二进制文件
2. **符号链接层**：启用服务自启动

## 🎯 完整的虚拟机启动流程

基于这些配置，虚拟机的启动流程如下：

```
Firecracker 启动虚拟机
    ↓
[内核启动]
    ↓
[PID 1: /usr/bin/init (BusyBox)]
    ↓
[读取 /etc/inittab]
    ↓
Step 1: 运行 /etc/init.d/rcS
        └─ 挂载 /proc, /sys, /dev, /tmp, /run
    ↓
Step 2: 运行 /usr/local/bin/provision.sh
        └─ 用户自定义初始化（安装包、配置等）
    ↓
Step 3: 运行 sync
        └─ 将所有更改写入磁盘
    ↓
Step 4: 输出 provision 脚本退出码
        └─ "E2B_PROVISIONING_EXIT:0" (成功) 或 "E2B_PROVISIONING_EXIT:1" (失败)
    ↓
Step 5: sleep infinity
        └─ 虚拟机暂停，创建快照
    ↓
[后续沙箱启动时]
从快照恢复 → 启动 systemd → envd 自动启动 → 沙箱就绪
```

## 💡 关键设计要点

### 1. **双 init 系统**
- **BusyBox init**：用于 provision 阶段（轻量级）
- **systemd**：用于正常运行阶段（功能完整）

### 2. **OOM 保护**
- `OOMScoreAdjust=-1000`：确保 envd 不会被系统杀死
  - OOMScoreAdjust用于调整进程在内存不足时被 OOM Killer（Out-Of-Memory Killer）选中终止的优先级参数


### 3. **日志前缀**
- 给 provision 输出添加前缀，方便在日志系统中过滤和分类

### 4. **快照机制**
- provision 完成后 `sleep infinity`
- 系统暂停并创建快照
- 后续沙箱启动直接从快照恢复（秒级启动）

### 5. **内存限制**
- envd 内存限制为总内存的一半（最多 512MB）
- 确保用户代码有足够内存运行


