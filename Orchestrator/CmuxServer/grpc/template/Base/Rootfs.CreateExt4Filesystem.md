---
tags:
  - Sandbox
  - E2B/Orchestrator
date: 
date created: 2025-11-28, 14:34:08
date modified: 2025-11-28, 16:22:53
---

CreateExt4Filesystem() 用于从 Docker 镜像创建 ext4 根文件系统，供 Firecracker 等虚拟机使用。
位于 `packages/orchestrator/internal/template/build/core/rootfs/rootfs.go` 80 - 178

## 创建流程

### 1. 获取基础镜像

```go
var img containerregistry.Image  
var err error  
if r.template.FromImage != "" {  // 配置了 FromImage，从 Docker Hub 获取公共镜像
    img, err = oci.GetPublicImage(childCtx, r.dockerhubRepository, r.template.FromImage, r.template.RegistryAuthProvider)  
} else {  // 否则从 artifact registry 获取模板镜像
    img, err = oci.GetImage(childCtx, r.artifactRegistry, r.template.TemplateID, r.metadata.BuildID)  
}
```

### 2. 设置系统文件层

```go
logger.Debug("Setting up system files")  
// 创建 OCI 层
layers, err := additionalOCILayers(childCtx, r.template, provisionScript, provisionLogPrefix, provisionResultPath)  
if err != nil {  
    return containerregistry.Config{}, fmt.Errorf("error populating filesystem: %w", err)  
}  

// 将层追加到镜像
img, err = mutate.AppendLayers(img, layers...)  
if err != nil {  
    return containerregistry.Config{}, fmt.Errorf("error appending layers: %w", err)  
}  
telemetry.ReportEvent(childCtx, "set up filesystem")
```

### 3. 将 OCI 镜像转换为 ext4 文件系统

```go
// 转换为 ext4 文件系统
ext4Size, err := oci.ToExt4(ctx, logger, img, rootfsPath, maxRootfsSize, r.template.RootfsBlockSize())

...

// 修改读写权限，使可写
err = filesystem.MakeWritable(ctx, rootfsPath)

...

// 调整文件系统大小
// 获取剩余空间
rootfsFreeSpace, err := filesystem.GetFreeSpace(ctx, rootfsPath, r.template.RootfsBlockSize())
// 计算需要增加的大小
// 需要从 ext4 文件大小中减去剩余空间rootfsFreeSpace，因为这是创建文件系统时无法收缩的残留空间，但仍可用
diskAdd := r.template.DiskSizeMB<<constants.ToMBShift - rootfsFreeSpace
// 扩容
if diskAdd > 0 {  
    _, err := filesystem.Enlarge(ctx, rootfsPath, diskAdd)  
    if err != nil {  
       return containerregistry.Config{}, fmt.Errorf("error enlarging rootfs: %w", err)  
    }  
}
```

### 4. 检查文件系统完整性

```go
ext4Check, err := filesystem.CheckIntegrity(ctx, rootfsPath, true)
```

`CheckIntegrity()`使用 e2fsck 检查 ext4 完整性，确保文件系统可用

```
packages/orchestrator/internal/template/build/core/filesystem/ext4.go
```

### 5. 返回镜像配置

```go
config, err := img.ConfigFile()  
if err != nil {  
    return containerregistry.Config{}, fmt.Errorf("error getting image config file: %w", err)  
}  
  
return config.Config, nil
```

---

## 附录

### 关于additionalOCILayers()

`additionalOCILayers` 创建两个 OCI 层：

#### 1. 文件层 (`filesLayer`)

通过 `oci.LayerFile()` 创建

```go
func LayerFile(filemap map[string]File) (containerregistry.Layer, error) {}
```

##### 1.1 网络配置

```go
	// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
	"etc/hostname":    {Bytes: []byte(hostname), Mode: 0o644},
	"etc/hosts":       {Bytes: []byte(hosts), Mode: 0o644},
	"etc/resolv.conf": {Bytes: []byte("nameserver 8.8.8.8"), Mode: 0o644},
```

- `hostname`: 设置为 `"e2b.local"`
- `hosts`: 包含 localhost 和 IPv6 映射

```
127.0.0.1   localhost  
::1 localhost ip6-localhost ip6-loopback  
fe00::  ip6-localnet  
ff00::  ip6-mcastprefix  
ff02::1 ip6-allnodes  
ff02::2 ip6-allrouters  
127.0.1.1  {{ hostname }}
```

- `resolv.conf`: DNS 服务器为 8.8.8.8

##### 1.2 Systemd 服务配置

```
	// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
	"etc/systemd/system/envd.service":                                {Bytes: []byte(envdService), Mode: 0o644},
	"etc/systemd/system/serial-getty@ttyS0.service.d/autologin.conf": {Bytes: []byte(autologinService), Mode: 0o644},
	"etc/systemd/system/systemd-journald.service.d/override.conf":    {Bytes: []byte(serviceWatchDogDisabledConfig), Mode: 0o644},
	"etc/systemd/system/systemd-networkd.service.d/override.conf":    {Bytes: []byte(serviceWatchDogDisabledConfig), Mode: 0o644},
```

- `envd.service`: 环境守护进程，内存限制为 `min(config.MemoryMB/2, 512)MB`

```
[Unit]  
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
Environment="GOMEMLIMIT={{ memoryLimit }}MiB"  
  
[Install]  
WantedBy=multi-user.target
```

- `autologin.conf`: 串口控制台自动登录 root

```
[Service]  
ExecStart=  
ExecStart=-/sbin/agetty --noissue --autologin root %I 115200,38400,9600 vt102
```

- `systemd-journald` 和 `systemd-networkd`: 禁用 watchdog

```
// serviceWatchDogDisabledConfig
[Service]
WatchdogSec=0
```

##### 1.3 预置脚本和初始化系统

```
	// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
	
	// Provision script
	"usr/local/bin/provision.sh": {Bytes: []byte(provisionScript), Mode: 0o777},
	
	// Setup init system
	BusyBoxPath: {Bytes: systeminit.BusyboxBinary, Mode: 0o755},
	
	// Set to bin/init so it's not in conflict with systemd
	// Any rewrite of the init file when booted from it will corrupt the filesystem
	BusyBoxInitPath: {Bytes: systeminit.BusyboxBinary, Mode: 0o755},
```

- `provision.sh`: 用户提供的预置脚本

```
packages/orchestrator/internal/template/build/phases/base/provision.sh
```

- `busybox`: 二进制文件，同时作为 `/usr/bin/busybox` 和 `/usr/bin/init`

```
packages/orchestrator/internal/template/build/core/systeminit/busybox_1.36.1-2
```

##### 1.4 BusyBox 初始化脚本

- `rcS`: 挂载 proc、sys、dev、tmp、run

```go
// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
"etc/init.d/rcS": {Bytes: []byte(`【init】`, provisionLogPrefix, ProvisioningExitPrefix, provisionResultPath), Mode: 0o777},
```

```
#【init】初始化脚本内容：

#!/usr/bin/busybox ash
echo "Mounting essential filesystems"
# Ensure necessary mount points exist
mkdir -p /proc /sys /dev /tmp /run

# Mount essential filesystems
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /tmp
mount -t tmpfs tmpfs /run

echo "System Init"
```

- `inittab`: 定义启动顺序

```go
// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
"etc/inittab": {Bytes: fmt.Appendf(nil, `【init】`, provisionLogPrefix, ProvisioningExitPrefix, provisionResultPath), Mode: 0o777}
```

```
# 【init】初始化脚本内容：

# Run system init  
::sysinit:/etc/init.d/rcS  
  
# Run the provision script, prefix the output with a log prefix  
::wait:/bin/sh -c '/usr/local/bin/provision.sh 2>&1 | sed "s/^/%s/"'  
  
# Flush filesystem changes to disk  
::wait:/usr/bin/busybox sync  
  
# Report the exit code of the provisioning script  
::wait:/bin/sh -c 'echo "%s$(cat %s || printf 1)"'  

# Wait forever to prevent the VM from exiting until the sandbox is paused and snapshot is taken  
::wait:/usr/bin/busybox sleep infinity
```

1. 运行系统初始化 (`rcS`)
2. 执行预置脚本（输出带前缀）
3. 同步文件系统
4. 报告预置脚本退出码
5. 保持运行直到得到快照

#### 2. 符号链接层 (`symlinkLayer`)

通过 `oci.LayerSymlink` 创建

```go
func LayerSymlink(symlinks map[string]string) (containerregistry.Layer, error) {}
```

用于启用 systemd 服务自启动：

```go
// packages/orchestrator/internal/template/build/core/rootfs/rootfs.go
symlinkLayer, err := oci.LayerSymlink(
	map[string]string{
		
		// Enable envd service autostart
		"etc/systemd/system/multi-user.target.wants/envd.service": "etc/systemd/system/envd.service",
		
		// Enable chrony service autostart
		"etc/systemd/system/multi-user.target.wants/chrony.service": "etc/systemd/system/chrony.service",
	},
)
```

- 创建 systemd wants 目录下的符号链接，使 `envd` 和 `chrony` 在 multi-user.target 时自动启动
