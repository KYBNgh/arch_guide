
---
**日期** 2026-02-08 ~ 2026-02-17

**测试环境**
- **CPU** Intel Core i3-4150
- **GPU** Intel HD Graphics 4400
- **主板** ASUS Z97 （UEFI）
- **内存** Kingston 8G
- **网卡** 主板内置，有线网络
- **磁盘** Sandisk SATA SSD 120G
---

# Arch Linux 全盘加密安装指南

> [!NOTE]
> 固件: UEFI
> 引导加载器: systemd-boot
> 文件系统: LUKS, btrfs
> 网络: systemd-networkd, systemd-resolved
> 交换空间: zram
## I 准备工作

### 1.1 烧录启动介质

以下方法任选其一
- `dd` （类 UNIX 系统）
- `Rufus` （Microsoft Windows）
- `ventoy` （全平台通用）

> [!IMPORTANT]
> 如果你不知道如何制作启动介质，也许你不应该现在尝试安装 Arch Linux
> 最好去选择一个新手友好的发行版: Fedora、Linux Mint

## 1.2 验证固件

开机进主板固件，以华硕主板为例

 - 在 `启动` 选项卡禁用 `CSM 兼容性模块`，令固件为纯 `UEFI` 模式
 - 在 `安全` 选项卡将 `操作系统类型` 改为 `其他操作系统`
 - 在 `安全` 选项卡中的 `密钥管理` 内 `删除所有PK`

> [!TIP]
 Arch ISO 是Legacy BIOS + UEFI 双启动
 Legacy BIOS 用的是 `GRUB` （较精美的启动菜单）
 UEFI 用的是 `systemd-boot` （黑底白字）
 
查看固件类型：

```sh
ls /sys/firmware/efi
```

有输出目录与文件，代表以 UEFI 启动。
"No such file or directory" 代表 Legacy BIOS
### 1.3 启动 `Arch ISO`

保存，重启计算机，从 `USB` 介质启动

> 华硕主板按 `F8` 调出启动菜单，`Ventoy 启动盘`的启动项一般以 `FAT32` 开头

以 `Arch ISO`内`systemd-boot` 启动菜单的第一项启动: 

```console
Arch Linux install medium (x86_64，UEFI)
Arch Linux install medium (x86_64，UEFI) with speech
				Memtest86+			
				EFI Shell			
		Reboot into firmware Interface
```
### 1.4 环境配置

```sh
# 启用 vi 按键模式
set -o vi

# 增大字体
setfont ter-132b
```
## II 配置磁盘：LUKS, btrfs

### 2.1 磁盘分区

使用 `lsblk` 查看当前磁盘布局（此处省略部分参数）

```sh
lsblk
```

在我的计算机上，其输出

```console
NAME         MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda            8:0    0 119.2G  0 disk  
```

我有一块 `SATA SSD`，设备名为 `sda`，即第一块 `SCSI 子系统` 管理的磁盘
若你的磁盘是 `nvme SSD`，设备名可能为 `nvme0n1`，即第一个 `NVMe 控制器` 上的第一个 `命名空间`

> [!NOTE]
> `nvme0` 指第一个 `NVMe 控制器`， `n1` 指第一个命名空间
> 消费级 `NVMe` 磁盘一般只有一个命名空间
> 若你有两块 `NVMe` 磁盘，则存在 `nvme0n1`、`nvme1n1` 

我的磁盘很小、使用 intel 核显、没有过于复杂的磁盘阵列
故`initramfs` 很小， `boot` 分区给 `256M` 已经足够
若你的磁盘空间阔绰，又使用 `NVIDIA` 显卡与复杂的硬件，这导致 `initramfs` 膨胀，强烈建议为 `boot` 分配 `1G` 大小

此分区方案无 `SWAP` 交换分区，我将使用 `zram`

- `/dev/sda1` ── `ESP` + `/boot` 合并( 256 M， FAT32 文件系统 )
- `/dev/sda2` ── LUKS 加密容器（剩余空间）
  ├─ Btrfs 文件系统
  ├─  `@` → `/` 
  ├─  `@home` → `/home` 
  └─  `@var` → `/var` 

### 2.2 使用 `fdisk` 进行分区

> [!CAUTION]
> 确认重要数据已转移
> 分区操作**不可撤销**

```sh
fdisk /dev/sda
```

##### 2.2.1 创建 GPT 分区表

```console
Command (m for help): g
Created a new GPT disklabel (GUID: 105D5E2A-B918-4933-A228-080FF8ABADE8).
```

##### 2.2.2 创建 EFI 分区（sda1）

```console
Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-249980485, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-249980485, default 249978879): +256M
Created a new partition 1 of type 'Linux filesystem' and of size 256 MiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): 1
Changed type of partition 'Linux filesystem' to 'EFI System'.
```

##### 2.2.3 创建 LUKS 容器分区（sda2）

```console
Command (m for help): n
Partition number (2-128, default 2):
First sector (526336-249980485, default 526336):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (526336-249980485, default 249978879):
Created a new partition 2 of type 'Linux filesystem' and of size 118.9 GiB
```

##### 2.2.4 保存并退出

```console
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

### 2.3 创建加密容器

格式化 `/dev/sda2` 为 LUKS 容器，输入密码

> [!IMPORTANT]
> LUKS **没有**找回密码的功能，忘了数据就丢了
> LUKS 支持用文件作为密钥

```sh
cryptsetup luksFormat /dev/sda2
```

映射 LUKS 容器。起个名字，例如 `cryptsys`

```sh
cryptsetup open /dev/sda2 cryptsys
```

使用 `lsblk` 查看磁盘

```console
NAME         MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda            8:0    0 119.2G  0 disk
├─sda1         8:1    0   256M  0 part
└─sda2         8:2    0   119G  0 part
  └─cryptsys 253:0    0   119G  0 crypt
```

### 2.4 格式化文件系统

此处的 `-L` 指定 `label`（`标签`），这是可有可无的
`UEFI` 固件只识别 `FAT32` 格式的文件系统

```sh
mkfs.btrfs -L archlinux /dev/mapper/cryptsys
mkfs.fat -F32 /dev/sda1
```

### 2.5 创建 `btrfs` 子卷

临时挂载 `btrfs` 文件系统

```sh
mount /dev/mapper/cryptsys /mnt
```

创建子卷
- @: /
- @home: /home
- @var: /var

```sh
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
```

卸载文件系统

```sh
umount /mnt
```

### 2.6 挂载 `btrfs` 子卷与 `ESP`

添加 `ssd` 挂载选项
genfstab 默认就有 `space_cache=v2` `discard=async` ，无需手动设置

```sh
# 挂载根子卷
mount -o compress=zstd,ssd,noatime,subvol=@ /dev/mapper/cryptsys /mnt

# 创建挂载点
mkdir -p /mnt/{boot,home,var}
```

```sh
# 挂载其余子卷
mount -o compress=zstd,ssd,noatime,subvol=@home /dev/mapper/cryptsys /mnt/home
mount -o compress=zstd,ssd,noatime,subvol=@var /dev/mapper/cryptsys /mnt/var

# 挂载 ESP (兼 boot 分区)
mount /dev/sda1 /mnt/boot
```

我的 `/var` 主要放大体积虚拟机映像文件，我不用快照功能
我禁用子卷 `@var` 的 `CoW` 功能

```sh
chattr +C /mnt/var
```

## III 配置基本系统

### 3.1 `pacstrap` 安装基本系统

注意: 插网线的台式机/笔记本用户通常无需配置即可上网，此处不讲 Wi-Fi 连接方法
自行搜索 `iwctl`

#可选 编辑 `pacman.conf` 开启颜色

```sh
sed -i '/^#Color$/s/#//' /etc/pacman.conf
```

搜索 `ustc` `tuna` `bfsu` ，移到开头，刷新软件包缓存

```sh
vim /etc/pacman.d/mirrorlist
pacman -Syy
```

使用 `pacstrap` 安装最小系统，`-K` 选项表示初始化密钥
内核、头文件、编辑器任选其一

> [!NOTE]
> **基础系统**
> - `base` `base-devel` `cryptsetup`
> - `btrfs-progs` `zram-generator`
> 
> **内核与头文件**（三选一）
> - 内核: `linux` / `linux-lts` / `linux-zen`
> - 头文件: `linux-headers` / `linux-lts-headers` / `linux-zen-headers`
> 
> **硬件支持**
> - 固件: `linux-firmware`
> - 微码: `intel-ucode` 或 `amd-ucode`（按 CPU 选）
> 
> **工具**
> - 编辑器: `vim` / `neovim` / `emacs`
> - 建议: `bash-completion` `man-db` 

> [!]
> `linux-firmware` 是元包（meta package），以依赖的形式安装以下软件包:
> 
> - `linux-firmware-amdgpu` / `linux-firmware-radeon`（AMD 显卡）
> - `linux-firmware-intel`（Intel 网卡/显卡）
> - `linux-firmware-nvidia`（NVIDIA 显卡）
> - `linux-firmware-realtek`（Realtek 网卡）
> - `linux-firmware-broadcom` / `linux-firmware-atheros`（其他固件）
> 
> 图省心就装 `linux-firmware` 整个固件包，磁盘空间小就要精挑细选

我的计算机只需要 `linux-firmware-intel` 和 `linux-firmware-realtek` 

```sh
pacstrap -K /mnt \
base base-devel \
	cryptsetup btrfs-progs intel-ucode zram-generator \
	linux linux-headers linux-firmware-intel linux-firmware-realtek \
	vim bash-completion
```

生成挂载信息，打印到终端，同时写入 `fstab`

```sh
genfstab -U /mnt | tee /mnt/etc/fstab
```

`arch-chroot` 进入系统

```sh
arch-chroot /mnt
```

### 3.2 本地化配置

加载 `bash_completion` 补全，设定 vi 模式

```bash
. /usr/share/bash_completion/bash_completion

set -o vi
```

设置时区，同步硬件时钟

```bash
# 中国大陆用户请设置为上海
ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

设置 locale。取消 `/etc/locale.gen`中 `en_US.UTF-8 UTF-8` 与 `zh_CN.UTF-8` 注释。
生成 locale，设定 系统级 locale 为 `en_US.UTF-8`

```bash
vim /etc/locale.gen
locale-gen
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

设置主机名

```bash
echo 'arch-desk' > /etc/hostname
```

编辑 `/etc/hosts`

```conf
127.0.1.1		arch-desk.localdomain arch-desk
````

### 3.3 LUKS 配置

编辑 `/etc/mkinitcpio.conf`
在 `block` 和 `filesystems` 中加上 `sd-encrypt`
注意，目前 `initramfs` 分传统 `busybox`和 `systemd` 两种，姑且叫“传统派”和“systemd派”
这两个只能二选一！

我采用 `systemd` 的 `initramfs` 方案

 > [!IMPORTANT]
 > **传统派的 HOOKS**
 >`udev` `consolefont`  `encrypt`
 >```conf
 > HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
 >```


 > [!IMPORTANT]
 > **systemd 派的 HOOKS**
`systemd` `sd-vconsole` `sd-encrypt`
>```conf
> HOOKS=(base systemd autodetect microcode modconf kms keyboard keymap sd-vconsole block sd-encrypt filesystems fsck)
>```

> [!WARNING]
> Arch Linux 中，base 后默认是 systemd
> 如果你选择传统派，务必改成 udev
> 否则就会卡在根文件系统的挂载，且无法输入密码 

#可选 配置 `vconsole`

若想 `ter-132b` 字体始终可用，安装字体包

```bash
pacman -S terminus-font
```

编辑 `/etc/vconsole.conf`

```conf
KEYMAP=us
FONT=ter-132b
```

编辑 `/etc/mkinitcpio.conf` 或修改 `/etc/vconsole.conf` 后，需重新生成`initramfs`

```bash
mkinitcpio -P
```

默认地，SSD 的 `TRIM`  请求被 `LUKS` 拦截，SSD 无法回收空闲块，长期使用后性能下降
编辑 `/etc/crypttab` 加 `discard` 参数，允许透传 TRIM 指令
这会暴露加密块的空闲状态，可能降低安全性，一般可接受

```conf
cryptsys UUID=</dev/sda2 的 UUID> none luks,discard
```

### 3.4 `systemd-boot` 配置

`systemd-boot` 已经集成在 `base` 包，随 `systemd` 提供，不需单独安装
安装 `systemd-boot` 到 `ESP`

```bash
bootctl install --esp-path=/boot
```

编辑 `/boot/loader/loader.conf`

```conf
default 09-arch.conf
timeout 3
console-mode max
```

> [!TIP]
> 这里加上 09 前缀是让默认内核排在最前面
> 后续我会加其他内核/EFI 工具（如 UEFI Shell）
> 若你确定只用这个内核，可以改掉 `09-arch.conf` 这个名字

使用 `blkid` 获取 LUKS 分区的 UUID 并追加到 `09-arch.conf` 

```bash
echo '#'$(blkid | grep '/dev/sda2') >> /boot/loader/entries/09-arch.conf
```

`echo '#'` 目的是打一个注释避免语法错误，使用 grep 提取 blkid 输出中关于 `dev/sda2` 的行（含 UUID ）
编辑 `/boot/loader/entries/arch.conf`

`linux` - 加载内核
`initrd` - 加载初始化内存盘/微码
`options` - 设置内核参数

`cryptsys` 是 `/dev/sda2` 的映射名
`rootflags` 指定 root 子卷为`@`、启用 `zstd` 透明压缩、告知磁盘类型为 `SSD`、使用  `v2` 版本的空间缓存、启用异步 `TRIM` / `DISCARD` 支持
`nowatchdog` 禁用看门狗

```conf
title	Arch Linux
linux	/vmlinuz-linux
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
options	rd.luks.name=</dev/sda2 的 UUID>=cryptsys root=/dev/mapper/cryptsys rootflags=subvol=@,compress=zstd:3,ssd,space_cache=v2,discard=async rw loglevel=3 nowatchdog
```

注意！如果你是 “传统派”
你的 `options` 不能照抄上面，你要用 `cryptdevice` 参数，且在 **冒号** 写出解密后的映射名
正确的写法如下

```conf
options cryptdevice=UUID=</dev/sda2 的 UUID>:cryptsys root=/dev/mapper/cryptsys rootflags=subvol=@,compress=zstd:3,ssd,space_cache=v2,discard=async rw loglevel=3 nowatchdog
```

### 3.5 开启 `systemd` 服务

```bash
# 自动更新 systemd-boot 引导程序
systemctl enable systemd-boot-update.service
```

若你是 Wi-Fi 用户、图方便、需要复杂网络功能( VPN )等，推荐使用 `NetworkManager`
且`GNOME` 桌面环境依赖 `NetworkManager`
`NetworkManager` 和 `systemd-networkd` 不能同时使用，否则引起冲突

若你使用 `NetworkManager`，请安装 `NetworkManager` 包，启用服务，即可上网，后面的都不用看了

```bash
pacman -S networkmanager
systemctl enable NetworkManager
```

我使用 `systemd-networkd` 和 `systemd-resolved`，适合简单网络，不需要安装额外软件包

```bash
# 启动网络、域名解析服务
systemctl enable systemd-{networkd,resolved}
```

查看物理网卡，一般是第二个

```bash
ip -br link
```

```console
lo          UNKNOWN     00:00:00:00:00:00   <LOOPBACK,UP,LOWER_UP> 
enp3s0      UP          aa:bb:cc:dd:ee:ff   <BROADCAST,MULTICAST,UP,LOWER_UP> 
```

多数有线网卡是 `enp3s0`，有些是 `eth0` 。替换 `enp3s0`为实际网卡
使用 `DHCP` 动态IP，`DHCP` 填 `yes`，否则需手动配置

编辑 `/etc/systemd/network/20-wired.network`

```ini
[Match]
Name=enp3s0

[Network]
DHCP=yes
```

配置 `DNS` 解析

```bash
ln -svf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

### 3.6 配置 `zram`

采用传统 `SWAP` 需要读写磁盘，加速 SSD 的磨损，占用一定磁盘空间
对小磁盘用户不友好

我不使用休眠，使用 `zram` 内存压缩
`zstd` 是较平衡的压缩算法，如果想降低 CPU 占用，可选压缩率低的 `lz4` 

编辑 `/etc/systemd/zram-generator.conf`
设置 `zram` 大小为物理内存的一半

```conf
[zram0]
zram-size = "ram / 2"
compression-algorithm = zstd
```

### 3.7 配置用户

`sudo` 已经包含在 `base-devel` 包组里
创建用户 `mmc0`，并添加到 `wheel` 组，设置密码

```bash
useradd -m -s /usr/bin/bash -G wheel mmc0
passwd mmc0
```

令 `/usr/bin/vi` 指向 `/usr/bin/vim`

```bash
ln -sv /usr/bin/vim /usr/bin/vi
```

配置 `/etc/sudoers`，赋予 `wheel` 组管理权限

```bash
visudo
```

```conf
# 取消此行注释
# %wheel ALL=(ALL:ALL) ALL

# 可选: 开启审计
Defaults logfile="/var/log/sudo.log"
```

为 root 账户设置密码，不和普通用户相同

```bash
passwd root
```

> [!WARNING]
若系统故障，进入 `rescue.target` ，`sulogin` 必须输入 root 密码
> 若没有 root 密码，你只能去 `Live ISO` 

## IV 清理

退出 `arch-chroot` 环境，同步磁盘，关闭映射，卸载文件系统，重启
~~并祈祷一切正常~~

```bash
exit
```

```sh
sync
cryptsetup close cryptsys
umount -R /mnt
reboot
```

## V 参考资料

Bilibili UP 主 unixchad
Arch Wiki
大语言模型 Deepseek、Qwen Chat

全盘加密、LUKS参考：
[ unixchad - 全盘加密安装Arch Linux的一切（困难模式）](https://www.bilibili.com/video/BV1DTT2zSE5R)
[ unixchad - arch-install-guide ](https://github.com/gnuunixchad/arch-install-guide)

Systemd 相关参考：
[ unixchad - 如何用systemd-boot替换现有系统的GRUB引导 ](https://www.bilibili.com/video/BV1kdLqzeEBA/)
[ Arch Wiki - Zram](https://wiki.archlinux.org/title/Zram)
[ Arch Wiki - Systemd-boot ](https://wiki.archlinux.org/title/Systemd-boot)
[ Arch Wiki - Systemd-networkd](https://wiki.archlinux.org/title/Systemd-networkd)
[ Arch Wiki - Systemd-resolved](https://wiki.archlinux.org/title/Systemd-resolved)

Btrfs 相关参考：
[ archlinux 简明指南 ](https://arch.icekylin.online/guide/)
 [ Arch Wiki - Btrfs](https://wiki.archlinux.org/title/Btrfs)
