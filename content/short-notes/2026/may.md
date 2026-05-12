+++
title = "简记 2026-05"
date = 2026-05-01T20:00:00+08:00
tags = ['linux', 'debian']
showtoc = true
tocopen = true
+++

## 把 Linux ISO 镜像写入 USB 设备的单个分区并启动

在常见情况下，在使用 dd 命令制作 linux 安装盘（启动盘）时，把 iso 文件写入整个 usb 设备，比如 `/dev/sdb` 可能是一块U盘，这时使用的命令如下。

**！！注意：必须要反复确认 dd 命令会写入的设备名称，否则会丢失数据！**

```sh
$ sudo dd if=/path/to/debian-13.4.0-amd64-DVD-1.iso of=/dev/sdb status=progress
```

虽然上面的命令制作了一个安装盘，但是实际上只使用了 usb 设备上的大约前 4 GB 空间。即使 usb 还有很多的空闲空间，也不能添加第二个安装镜像到 usb 里面了。

经过尝试发现，把 iso 镜像写入单个硬盘分区是可以工作的。经过一些附加操作，能够启动 iso 并完成系统安装流程。

下面以 `debian-13.4.0-amd64-DVD-1.iso` 为例子，把它写入 usb 设备的一个分区并启动。

### 把 ISO 写入单个硬盘分区

文件 `debian-13.4.0-amd64-DVD-1.iso` 大小是 `3985178624` 字节。

- `3985178624 = 512 * 7783552`，一个扇区是 512 字节，一共是 `7,783,552` 个扇区。

使用 fdisk 命令，在 usb 设备中创建一个分区。分区的大小可以等于 iso 文件的大小，`7,783,552` 个扇区。也可以选择一个更大的数字，比如 `8,000,000` 个扇区。
下面创建一个 `8,000,000` 个扇区大小的分区。

``` text
$ sudo fdisk /dev/sdb

Command (m for help): n
Partition number (1-128, default 1): 65
First sector (2048-124735454, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-124735454, default 124733439): +7999999

Created a new partition 65 of type 'Linux filesystem' and of size 3.8 GiB.

Command (m for help): p
Disk /dev/sda: 59.48 GiB, 63864569856 bytes, 124735488 sectors
Disk model: USB3.0 CRW      
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 52D7A562-A5A4-4F0B-B0C6-3E8065E08C79

Device     Start     End Sectors  Size Type
/dev/sda65  2048 8002047 8000000  3.8G Linux filesystem
```

使用 dd 把 iso 文件写入刚才创建的分区。

```sh
$ sudo dd if=/path/to/debian-13.4.0-amd64-DVD-1.iso of=/dev/sdb65 status=progress
```

### 启动安装盘

启动单个分区内的 debian dvd iso，需要执行一些步骤：

1. 在 UEFI 管理界面（BIOS 界面）中选择从文件里启动。
2. 找到 iso 镜像所在的分区，找到其中的 EFI 文件夹，从 `/EFI/boot/bootx64.efi` 之类的文件启动。
3. 这时大概会进入 grub shell，因为 grub 没有找到 `grub.cfg` 配置文件。在 grub shell 中，
    - 执行 `ls`，查看有哪些分区。假设 iso 写入的分区对应 `(hd0,gpt65)`，第 65 个分区。
    - 执行 `configfile (hd0,gpt65)/EFI/debian/grub.cfg`，切换配置文件。
    - 这样应该可以启动了。

### 执行安装步骤

对于 debian 安装盘，它通过 debian installer 完成安装。

在安装的过程中，可能会找不到安装盘，这时需要进入 shell 中，手动挂载安装盘到合适的位置。

```sh
mount /dev/sdb65 /cdrom

mount /dev/sdb65 /target/media/cdrom
```


## grub 命令

几个有用的 grub 命令，用于处理出问题的情况。

- `ls` 查看设备情况。第一个设备是 `(hd0)`。假设使用 GPT 分区表，第一个设备的第一个分区是 `(hd0,gpt1)`。
- `configfile (hd0,gpt1)/efi/boot/grub.cfg` 切换到第一个设备的第一个分区的指定目录里的 `grub.cfg` 配置文件。可以用于 grub 没有找到配置文件的情况。
- `set root=(hd0,gpt1)` 设置 grub 使用的根分区。
- `cat /efi/boot/grub.cfg` 查看根分区下指定文件的内容。

制作 loopback 回环设备。可以用于从 iso 文件启动安装器。

```text
loopback loop0 (hd0,gpt3)/debian-live-13.4.0-amd64-kde.iso
set root=(loop0)
# 切换 root 到 loop 设备中，使用 iso 里面存放的 grub 配置文件。
configfile /boot/grub/grub.cfg
```

关于支持 loopback 的 iso，可以参考 <https://github.com/Mexit/MultiOS-USB/blob/master/docs/Supported_OS.md>。