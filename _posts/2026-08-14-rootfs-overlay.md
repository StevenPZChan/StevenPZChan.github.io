---
title: 根文件系统只读+overlay方案
layout: post
categories: linux
tags:
  - linux
  - systemd
  - overlay
last_modified_at: 2026-11-03T15:00:00+08:00
---

分享一个简单实用的根文件系统只读 + overlay 方案。

### 背景

有些时候我们开发的嵌入式系统的根文件系统（rootfs）也需要在线升级，这个时候会遇到一些问题，比如用户调整过一些系统配置，如果是使用完全覆盖的方式，这些配置就会回到默认值，可能会影响使用。另外，系统运行会产生一些日志文件，如果是完全覆盖的方式也有可能会使日志丢失，难以排查问题。

### 方案

这个时候我们就会想到能不能在升级时只覆盖核心的部分，而用户使用时产生的新数据是保留下来的。很自然的一个想法是新开一个数据分区，将要产生的新数据都导向这个数据分区。但是这样处理的话会非常复杂：

1. 会产生新数据的目录或者文件有很多，逐一使用软链接覆盖会很复杂
2. 对于目录来说只能实现全部替换，比如放服务的目录，如果新 rootfs 新增了某个服务，有可能用户会运行不到那个服务
3. 删除文件或者目录很难处理

事实上，内核就提供了一种叫 overlayfs 的文件系统，可以在现有的文件目录上进行叠加，可以支持文件修改、新增或者删除。当原有的文件目录中有更新，如果 overlay 层没有记录相应的变化，是可以正常反映到用户使用上的。因此这个文件系统完美解决了 rootfs 的升级问题，并且由于修改都是写入的 overlay 层，rootfs 完全可以只读加载，还能大大延长 flash 的寿命。

### Overlay 目录

我们使用 systemd 系统，系统中需要修改或者存放数据的目录或者文件大致如下表：

| 目录/文件 | 说明 |
| --- | --- |
| `/etc/passwd` `/etc/shadow` `/etc/group` `/etc/gshadow` | 用户账号信息，修改密码或新增用户后会变化 |
| `/etc/hostname` `/etc/hosts` `/etc/resolv.conf` | 主机名、网络和 DNS 配置 |
| `/etc/systemd/system/` | 自定义服务单元及开机自启链接 |
| `/etc/machine-id` | 系统唯一 ID，首次启动时生成 |
| `/etc/ssh/` | SSH 主机密钥，首次启动时生成 |
| `/var/log/` | 系统日志，包括 journald 和 syslog |
| `/var/lib/` | 服务运行数据，如 systemd random-seed、dbus、时间同步状态等 |
| `/var/cache/` | 程序产生的缓存数据 |
| `/var/tmp/` | 需要跨重启保留的临时文件 |
| `/root/` | root 用户主目录，如命令历史、authorized_keys 等 |

汇总一下事实上只需要处理 `/etc` `/var` `/root` 三个目录。我们可以考虑只用一个分区来保存 overlay 层数据。

### 分区

在做分区方案时预留一块为 overlay，然后在 rootfs 文件系统中预创建 `/overlay` 目录为 overlay 分区挂载提供挂载点。

### Init 脚本

为了保证系统运行时产生的新数据都保存到了 overlay 层，我们需要在 systemd （init进程）运行之前就把 overlayfs 挂载好。这里我们选择使用 init 脚本，因为它简单灵活，也是可以随着 rootfs 升级的。

Overlay-init 脚本的内容其实也非常简单，首先检查 overlay 分区是否存在，然后挂载 overlay 分区，然后将 `/etc` `/var` `/root` 目录挂载成 overlayfs，最后启动原来的 init 进程：

```shell
#!/bin/bash
# Early init: set up OverlayFS for /etc, /var and /root before systemd

echo "[overlay-init] Starting overlay setup..."

# Mount required kernel filesystems
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /run

# Find a partition by name and print its device node. udev is not running
# at this early stage, so /dev/disk/by-partlabel does not exist yet;
# look up PARTNAME in sysfs instead.
find_part() {
    local ue
    for ue in /sys/class/block/*/uevent; do
        if grep -qx "PARTNAME=$1" "$ue" 2>/dev/null; then
            echo "/dev/$(basename "${ue%/uevent}")"
            return 0
        fi
    done
    return 1
}

# Wait up to 10s for the overlay partition to appear
timeout=100
while ! overlay_dev=$(find_part overlay) && [ $timeout -gt 0 ]; do
    sleep 0.1
    timeout=$((timeout - 1))
done
if [ -z "$overlay_dev" ]; then
    echo "[overlay-init] ERROR: overlay partition not found, booting without overlay"
    exec /sbin/init
fi

# Repair and mount the overlay partition (flash may suffer power-loss corruption)
e2fsck -y "$overlay_dev" 2>/dev/null
mount -t ext4 "$overlay_dev" /overlay || {
    echo "[overlay-init] ERROR: failed to mount overlay partition, booting without overlay"
    exec /sbin/init
}

# Factory reset: if .factory_reset marker exists, remove the whole overlay dir (recreated by setup)
if [ -f /overlay/.factory_reset ]; then
    echo "[overlay-init] Factory reset requested, clearing overlay..."
    find /overlay -mindepth 1 -delete
    echo "[overlay-init] Overlay cleared."
fi

# Set up overlay (/etc, /var and /root)
for dir in etc var root; do
    mkdir -p "/overlay/${dir}/upper" "/overlay/${dir}/work"
    mount -t overlay overlay \
        -o "lowerdir=/${dir},upperdir=/overlay/${dir}/upper,workdir=/overlay/${dir}/work" \
        "/${dir}" || {
        echo "[overlay-init] WARNING: /${dir} overlay setup failed"
    }
done

echo "[overlay-init] Overlays ready, starting systemd..."
exec /sbin/init
```

这里还包含了一个恢复出厂设置的入口，可以在用户触发恢复出厂设置的应用代码里创建一个 `/overlay/.factory_reset` 文件，下次系统启动时就会把 overlay 层清空，以应对某次升级的 rootfs 跟用户当前 overlay 层数据有冲突导致程序运行不正常的情况。

### 内核启动参数

把上述脚本放在 `/sbin/overlay-init`，并赋予可执行权限。然后修改设备树里的 `chosen -> bootargs` 启动参数，添加 `ro init=/sbin/overlay-init`，这样一来内核加载 init 进程的时候就会调用这个脚本，先设置好 overlayfs 之后，再启动 systemd，完美。

补充说明，如果要让 rootfs 为只读，还需要修改 `/etc/fstab` 里的挂载参数，以免系统启动后会被重新以 `rw` 读写挂载。
