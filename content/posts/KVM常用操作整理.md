---
title: KVM常用操作整理
date: 2026-10-12 10:00:00
tags:
  - KVM
categories:
  - Linux
description: 整理 KVM 虚拟化的日常操作:桥接网络配置、创建与导入虚拟机、磁盘格式转换与压缩、克隆、静态与热迁移、磁盘扩容、console 卡住的解决办法。
---

本文整理 KVM 虚拟化日常运维中最常用的操作,包括桥接网络配置、虚拟机的创建与导入、磁盘格式转换、克隆、迁移、磁盘扩容等,方便随时查阅。

<!-- more -->

## 一、网络桥接配置

桥接后,虚拟机可以像物理机一样直接获得局域网 IP。下面有两种配置方法。

### 方法 1:修改网卡配置文件

> 修改完成后**直接重启系统**,不要只重启网卡。

配置文件位于 `/etc/sysconfig/network-scripts/` 目录下(CentOS / RHEL 系)。

**创建桥接网卡配置 `ifcfg-br0`:**

```bash
vim ifcfg-br0
```

```ini
DEVICE="br0"
ONBOOT="yes"
TYPE="Bridge"
BOOTPROTO="none"
IPADDR="192.168.19.196"
GATEWAY="192.168.19.1"
STP="on"
DELAY="0"
DNS1=192.168.19.1
DNS2=202.96.128.86
```

**修改物理网卡配置 `ifcfg-enp6s0`**,让它加入网桥:

```bash
vim ifcfg-enp6s0
```

```ini
DEVICE=enp6s0
ONBOOT=yes
BRIDGE="br0"
```

> 注意:物理网卡名称(这里是 `enp6s0`)和 IP、网关、DNS 都要按实际环境替换。

### 方法 2:命令行操作

> 执行后网络会中断,**重启系统后恢复正常**。

```bash
# 创建网桥
virsh iface-bridge eth0 br0

# 删除 virbr0 虚拟交换机(默认 NAT 网络)
virsh net-destroy default
virsh net-undefine default
```

### 问题:虚拟机可以 ping 通 IP,但无法 ping 通域名

这通常是网桥的流量被宿主机的 iptables / 防火墙规则过滤导致的。在宿主机上关闭桥接流量过滤。

编辑 `/etc/sysctl.conf`,添加:

```ini
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-arptables = 0
```

使配置生效:

```bash
sysctl -p
```

如果提示 `No such file or directory`,说明 `br_netfilter` 模块未加载,先执行 `modprobe br_netfilter` 再 `sysctl -p`。

## 二、创建虚拟机

### 1. 创建虚拟机磁盘镜像

```bash
qemu-img create -f qcow2 /data/kvm/images/vm1.qcow2 300G
```

### 2. 通过 ISO 安装并启动虚拟机

```bash
virt-install \
--name webvirtmgr-192.168.19.80 \
--virt-type kvm \
--os-variant ubuntu22.04 \
--memory 8192 \
--vcpus 4 \
--disk /data/kvm/images/webvirtmgr.qcow2 \
--cdrom /data/kvm/iso/ubuntu-22.04.1-live-server-amd64.iso \
--network bridge=br0 \
--graphics vnc,password=123456,listen=0.0.0.0 \
--autostart
```

参数说明:

| 参数 | 说明 |
| --- | --- |
| `--name` | 虚拟机名称 |
| `--virt-type kvm` | 虚拟化类型 |
| `--os-variant` | 操作系统类型,用于优化默认配置 |
| `--memory` | 内存大小,单位 MB |
| `--vcpus` | CPU 核数 |
| `--disk` | 磁盘镜像路径 |
| `--cdrom` | 安装用的 ISO 镜像 |
| `--network bridge=br0` | 使用 `br0` 网桥 |
| `--graphics vnc,...` | 开启 VNC 远程控制台,设置密码和监听地址 |
| `--autostart` | 宿主机开机时自动启动该虚拟机 |

### 3. 用已有镜像创建虚拟机

磁盘镜像已存在时,加上 `--import` 参数,跳过安装步骤:

```bash
virt-install \
--virt-type kvm \
--name dev-goclick-192.168.19.73 \
--os-variant ubuntu22.04 \
--memory 8192 \
--vcpus 4 \
--disk /data/kvm/images/dev-goclick.qcow2 \
--network bridge=br0 \
--graphics vnc,password=123456,listen=0.0.0.0 \
--autostart \
--import
```

## 三、磁盘格式转换与压缩

### 1. 转换磁盘格式为 qcow2

```bash
qemu-img convert -f raw -O qcow2 /opt/kvm/vm1.raw /opt/kvm/vm1.qcow2
```

### 2. qcow2 镜像压缩

**第一步**:在虚拟机内,对剩余空间做零写操作:

```bash
dd if=/dev/zero of=/zero.dat
rm /zero.dat
```

**第二步**:关闭虚拟机,通过 `qemu-img convert` 压缩:

```bash
qemu-img convert -c -O qcow2 /path/old.img.qcow2 /path/new.img.qcow2
```

**第三步**:编辑虚拟机配置文件,搜索 `disk`,把磁盘路径改成压缩后的新文件:

```bash
virsh edit vm1
```

## 四、虚拟机克隆

语法:

```bash
virt-clone -o 源虚拟机名 -n 新虚拟机名 -f 新虚拟机磁盘文件名 --auto-clone
```

示例:

```bash
virt-clone -o template -n zhuxijiang-192.168.22.66 -f /opt/kvm/zhuxijiang-192.168.22.66.qcow2 --auto-clone
```

> 克隆前需要先关闭源虚拟机。

## 五、虚拟机迁移

### 1. 静态迁移

在源宿主机上:

```bash
# 查看虚拟机使用的磁盘文件
virsh domblklist centos7

# 导出虚拟机配置
virsh dumpxml centos7 > /opt/centos7.xml
```

然后把**磁盘文件**和**配置文件**拷贝到目标机的对应目录。

在目标机上注册虚拟机:

```bash
virsh define /opt/centos7.xml
```

### 2. 热迁移

前置条件:

- 两台宿主机设置好 hostname,并在 `/etc/hosts` 中互相绑定。
- 需要安装 `qemu-kvm-ev`。

```bash
virsh migrate --live zfweb-backup-192.168.19.79 \
  --copy-storage-all --unsafe --persistent --verbose --undefinesource \
  qemu+tcp://kvm-node02/system
```

参数说明:

| 参数 | 说明 |
| --- | --- |
| `--live` | 热迁移,虚拟机不停机 |
| `--copy-storage-all` | 同时迁移磁盘数据(没有共享存储时使用) |
| `--unsafe` | 忽略部分安全检查 |
| `--persistent` | 在目标机上永久保留虚拟机定义 |
| `--undefinesource` | 迁移完成后,在源机上删除虚拟机定义 |
| `--verbose` | 显示迁移进度 |

## 六、磁盘扩容(GPT + LVM + KVM)

> 扩容前建议先备份磁盘镜像。

**宿主机上**:扩大镜像文件(这里增加 400G):

```bash
qemu-img resize outbound-call-system.qcow2 +400G
```

**虚拟机内**:扩展分区、逻辑卷和文件系统:

```bash
# 扩展第 3 个分区
growpart /dev/vda 3

# 扩展逻辑卷,使用全部剩余空间
lvextend -l +100%FREE /dev/mapper/vg0-lv--0

# 扩展 XFS 文件系统
xfs_growfs /dev/mapper/vg0-lv--0
```

说明:

- `growpart` 来自 `cloud-utils-growpart` 软件包,没有的话需要先安装。
- `xfs_growfs` 只适用于 XFS 文件系统;ext4 请使用 `resize2fs /dev/mapper/vg0-lv--0`。
- 分区号、分区名、卷组名要按 `lsblk` 和 `lvs` 的实际结果替换。

## 七、调整虚拟机配置

修改 CPU、内存等配置的流程:

```bash
# 关闭虚拟机
virsh shutdown vm1

# 查看虚拟机信息
virsh dominfo vm1

# 编辑虚拟机配置
virsh edit vm1

# 重新加载配置(直接用 virsh edit 保存时通常会自动生效,手动改 XML 文件时需要执行)
virsh define /etc/libvirt/qemu/vm1.xml

# 启动虚拟机
virsh start vm1
```

## 八、virsh console 卡住的解决办法

`virsh console` 连接后没有反应,是因为虚拟机内没有启用串口控制台。

### 1. 修改 GRUB 配置

进入虚拟机,编辑 `/etc/default/grub`,找到 `GRUB_CMDLINE_LINUX` 这一行:

```ini
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap rhgb quiet"
```

在尾部增加 `console=ttyS0,115200`:

```ini
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap rhgb quiet console=ttyS0,115200"
```

然后重新生成 GRUB 配置(BIOS 启动):

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

> UEFI 启动的系统,输出路径一般是 `/boot/efi/EFI/<发行版>/grub.cfg`。

### 2. 启动并设置串口登录服务

这是 CentOS 7 才有的服务,默认没有启动:

```bash
sudo systemctl start serial-getty@ttyS0.service
sudo systemctl enable serial-getty@ttyS0.service
```

### 3. 重启后连接

```bash
virsh console vm1
```

退出 console 使用快捷键 `Ctrl + ]`。

## 九、使用提示

1. **VNC 安全**:示例中使用了 `listen=0.0.0.0` 和弱密码 `123456`,仅适合内网测试。实际使用请设置强密码,并用防火墙限制 VNC 端口的访问来源,或改为只监听 `127.0.0.1` 并通过 SSH 隧道连接。
2. **操作前先备份**:磁盘扩容、压缩、迁移前,先备份镜像文件和 XML 配置。
3. **XML 配置位置**:虚拟机配置默认保存在 `/etc/libvirt/qemu/` 目录,建议使用 `virsh edit` 修改,而不是直接改文件。
4. **常用查看命令**:

```bash
virsh list --all          # 查看所有虚拟机
virsh start vm1           # 启动
virsh shutdown vm1        # 正常关机
virsh destroy vm1         # 强制关机
virsh undefine vm1        # 删除虚拟机定义(不删磁盘)
virsh autostart vm1       # 设置开机自启
```
