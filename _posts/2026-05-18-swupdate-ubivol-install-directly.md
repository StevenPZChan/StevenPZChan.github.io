---
title: Swupdate工具使用ubivol流式安装注意事项
layout: post
categories: linux
tags:
  - linux
last_modified_at: 2026-11-03T11:00:00+08:00
---

分享一个 swupdate 工具使用 ubivol 流式安装的注意事项。

### 背景

当今存储价格疯涨，项目只能选择容量小的 NAND flash 和 RAM，文件系统还得选择 ubi，在 OTA 的时候把 .swu 升级包下载下来之后根本就没有多少空闲的容量进行解压，只得考虑 swupdate 本身自带的流式安装的方式，在读取 .swu 升级包的同时就以流式的方式直接写 flash，免去解压的步骤。

### 难点

1. 由于文件系统是 ubi 的，还不能直接使用 swupdate 的 flash 直写模块，需要用到 ubivol 模块
1. ubivol 模块要配置好参数，以应对各种可能出现的情况
1. swupdate 的流式安装是读取 .swu 升级包时就进行的早期行为，像 preinstall 这类的脚本都是在解压后执行的，如果要做前处理，需要使用 embedded-script 的方式
1. ubi 卷的一些前后处理也很重要，一方面是要确保流式安装能成功写入，另一方面是要保障重启完仍能加载，这些并没有包含在 ubivol 模块的义务里

### ubivol 模块

ubivol 是 swupdate 自带的 UBI 卷写入 handler，即 `type = "ubivol"`。它只负责把镜像内容写进 UBI 卷，不改动 UBI 布局，行为与 mtd-utils 的 ubiupdatevol 一致，底层复用的也是同一个库 libubi：按卷名找到目标卷后发起 volume update，把数据流式写入。

sw-description 中 ubivol 相关的参数：

| 参数 | 说明 |
| --- | --- |
| volume | 目标 UBI 卷名，必填 |
| device | MTD 设备节点，如 /dev/mtd12。使用 auto-resize 时必须指定 |

另外两个 property 用来控制写入前对卷的处理：

| property | 说明 |
| --- | --- |
| always-remove | 写入前无条件删除目标卷再重建，保证旧数据不残留。官方建议与 auto-resize 搭配使用，因为删除之后要按镜像大小重建 |
| auto-resize | 写入前把卷大小调整为恰好装下镜像 |

默认不删卷、就地 update，与 ubiupdatevol 一样；always-remove 则是先删后建，多一次卷操作，换来的是目标卷的干净状态。

### sw-description

由于要使用 embedded-script，为了具有更好排版的可读性，我们要使用 libconfig 的格式，大致结构如下：

```conf
software = {
    version = "1.0.0";
    description = "UBI streaming update with embedded precheck";

    embedded-script =
        "...";

    stable = {
        nowa_nextb = {
            images = (
                {
                    filename = "app";
                    device = "/dev/mtd12";
                    type = "ubivol";
                    volume = "app";
                    installed-directly = true;
                    properties = {
                        always-remove = "true";
                        auto-resize = "true";
                    };
                }
            );
            scripts = (
                {
                    filename = "postinstall";
                    type = "postinstall";
                    data = "12 app_b";
                }
            );
        };
        nowb_nexta = {
            images = (
                {
                    filename = "app";
                    device = "/dev/mtd11";
                    type = "ubivol";
                    volume = "app";
                    installed-directly = true;
                    properties = {
                        always-remove = "true";
                        auto-resize = "true";
                    };
                }
            );
            scripts = (
                {
                    filename = "postinstall";
                    type = "postinstall";
                    data = "11 app_a";
                }
            );
        };
    };
};
```

其中流式安装必须写死 `device` 位置，ubivol 模块的相关参数如上文所述设置，最后要把 MTD 的设备号传参给后处理脚本做可靠性处理，见下文。

#### 前处理

Swupdate 在启动时会默认扫描一遍所有的 MTD 设备，并且做 ubiattach 操作，而在结束的时候是不会进行 ubidetach 操作的。因此如果在 swupdate 运行的过程中有一些对 ubi 卷的操作不当，数据可能就没有成功保存。这里分几种情况：

1. 要写入的分区没有进行 ubiattach，则 swupdate 运行时自动进行 ubiattach，然后 ubivol 删除卷、重建卷、写入数据，这个过程是正常的
1. 要写入的分区已经进行了 ubiattach，则 swupdate 运行时自动进行 ubiattach，发现已经 attach 了就忽略默认成功，读取当前 ubi 卷的信息，然后 ubivol 删除卷、重建卷、写入数据，这个过程是依赖于当前 ubi 的卷信息是否还正确，如果在 swupdate 运行前有人 ubiattach 了并且错误操作了 ubi 卷，就有可能导致 ubivol 写入数据后重启就失效了
1. 要写入的分区损坏了，swupdate 运行时自动进行 ubiattach 失败，后续的 ubivol 也无法成功

因此，我们的前处理关键是要检查出来分区是否有损坏，如有损坏要使用 ubiformat 工具重新格式化，然后还要保证 ubivol 操作之前这个分区是刚刚 ubiattach 的。操作上的方案就是在前处理中先强制 ubidetach 再 ubiattach，这里的情况就有：

1. swupdate 之前分区没有进行 ubiattach，则 swupdate 运行时自动进行了 ubiattach，那么增加 ubidetach + ubiattach 的操作之后还是一样的状态，完全没有风险
1. swupdate 之前分区已经进行了 ubiattach，则 swupdate 运行时自动进行的 ubiattach 会忽略默认成功，那么增加 ubidetach + ubiattach 的操作会调整为刚刚 ubiattach 的状态；同时，如果这一步 ubidetach 或者 ubiattach 出错了，说明分区是有损坏的，需要使用 ubiformat 进行修复
1. swupdate 运行时自动进行 ubiattach 失败，则 ubidetach 会出错，直接进行 ubiformat 修复；这个时候后续的 ubivol 是不会成功的，需要重新运行 swupdate 走自动 ubiattach 的流程

Embedded script 示例如下：

```conf
embedded-script =
    "function is_mtd_in_use(mtd_num)\n"
    "  local mounts = io.open(\"/proc/mounts\", \"r\")\n"
    "  if not mounts then\n"
    "    swupdate.error(\"Failed to open /proc/mounts\")\n"
    "    return false\n"
    "  end\n"
    "  for line in mounts:lines() do\n"
    "    local dev, _, fstype = line:match(\"^(%S+)%s+(%S+)%s+(%S+)\")\n"
    "    if dev and fstype == \"ubifs\" then\n"
    "      local ubi_num = dev:match(\"/dev/ubi(%d+)_%d+\")\n"
    "      if ubi_num then\n"
    "        local mtd_file = io.open(\"/sys/class/ubi/ubi\" .. ubi_num .. \"/mtd_num\", \"r\")\n"
    "        if mtd_file then\n"
    "          local mtd_num_in_use = mtd_file:read(\"*line\")\n"
    "          mtd_file:close()\n"
    "          if mtd_num_in_use == tostring(mtd_num) then\n"
    "            mounts:close()\n"
    "            return true\n"
    "          end\n"
    "        end\n"
    "      end\n"
    "    end\n"
    "  end\n"
    "  mounts:close()\n"
    "  return false\n"
    "end\n"
    "\n"
    "local _, mode = swupdate.get_selection()\n"
    "if not mode then\n"
    "  swupdate.error(\"No mode selected\")\n"
    "  return\n"
    "end\n"
    "local target_mtd = nil\n"
    "if mode == \"nowa_nextb\" then\n"
    "  target_mtd = 12\n"
    "elseif mode == \"nowb_nexta\" then\n"
    "  target_mtd = 11\n"
    "end\n"
    "if not target_mtd then\n"
    "  return\n"
    "end\n"
    "local mtd_dev = \"/dev/mtd\" .. target_mtd\n"
    "swupdate.info(\"Checking if \" .. mtd_dev .. \" is in use\")\n"
    "if is_mtd_in_use(target_mtd) then\n"
    "  swupdate.error(\"Cannot update \" .. mode .. \" because \" .. mtd_dev .. \" is in use\")\n"
    "  os.exit(1)\n"
    "end\n"
    "if not os.execute(\"sleep 1 && ubidetach -p \" .. mtd_dev .. \" && ubiattach -p \" .. mtd_dev) then\n"
    "  swupdate.warn(mtd_dev .. \" is corrupted, repairing...\")\n"
    "  if not os.execute(\"ubiformat -y \" .. mtd_dev) then\n"
    "    swupdate.error(\"Failed to format \" .. mtd_dev)\n"
    "    os.exit(1)\n"
    "  end\n"
    "  swupdate.warn(mtd_dev .. \" repaired, might need to re-run swupdate\")\n"
    "end\n"
    "swupdate.info(\"Protection passed\")\n";
```

#### 后处理

后处理也是一样的道理，需要用 ubidetach + ubiattach 的方式来确保重启后 ubi 文件系统还是正常的，不然就要用 ubiformat 修复并且要标记升级失败。还有一点需要处理的是，如果这个分区需要可写，要使用 ubirsvol 工具来进行容量扩展。最终调用 ubidetach 确保分区不在加载状态，以免再进行不当的操作（同时也可以再次运行 swupdate 来重新进行升级）。后处理脚本示例如下：

```shell
APP_MTD_NUM="$1"
APP_PARTITION="$2"
APP_DISK="/dev/mtd${APP_MTD_NUM}"

sleep 1
if ubidetach -p "${APP_DISK}" && ubiattach -p "${APP_DISK}" -d "${APP_MTD_NUM}"; then
    echo "UBI device ${APP_DISK} detached and re-attached successfully."
else
    ubiformat -y "${APP_DISK}"
    fw_setenv app_upgrade_available 0
    fw_setenv app_bootcount 0
    echo "Failed to detach and re-attach UBI device ${APP_DISK}, fixed, need retry."
    exit 1
fi
sleep 1
USED_BLOCKS=$(cat /sys/class/ubi/ubi${APP_MTD_NUM}_0/reserved_ebs)
AVAILABLE_BLOCKS=$(cat /sys/class/ubi/ubi${APP_MTD_NUM}/avail_eraseblocks)
ubirsvol /dev/ubi${APP_MTD_NUM} -n 0 -S $((USED_BLOCKS + AVAILABLE_BLOCKS))
ubidetach -p "${APP_DISK}"

fw_setenv app_partition "${APP_PARTITION}"
fw_setenv app_upgrade_available 1
fw_setenv app_bootcount 0

echo "Upgrade completed successfully, please reboot..."
```

### 总结

总的来说，流式安装省掉的是解压临时文件的空间开销，省不掉的是 flash 状态的保障：ubivol 只负责把镜像写进卷，分区是否干净、重启后能否正常加载，都要靠前处理和后处理来兜底。前处理用 embedded-script 把目标分区调整到刚刚 ubiattach 的干净状态，后处理用 postinstall 确认写入结果并设置好启动相关的环境变量，两头做扎实，这套方案在小容量 ubi 设备上就能稳定工作。
