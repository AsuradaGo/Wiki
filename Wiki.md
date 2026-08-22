# 前言-Markdown使用说明
## 1.标题
```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```
## 2.文本样式
```
加粗：**加粗内容** 或 __加粗内容__
斜体：*斜体内容* 或 _斜体内容_
加粗并斜体：***加粗斜体***
~~删除线~~：~~删除内容~~
高亮（部分渲染器支持）：==高亮内容==
```
## 3.段落与换行
```
换行：在一行结尾添加 两个空格 再回车，或直接连续敲两次回车（空一行）。
分割线：单独一行使用三个或以上的 ***、--- 或 ___。
```
## 4.列表
```
无序列表
使用 *、+ 或 - 紧跟一个空格：
  - 列表项A
  - 列表项B
    - 嵌套列表（缩进 2 或 4 个空格）
有序列表
使用数字加英文句号 . 再加一个空格
1. 第一步
2. 第二步
3. 第三步
```
## 5.任务列表
```
[x] 已完成事项
[ ] 未完成事项
```
## 6.引用
```
使用 > 符号，支持嵌套：
> 这是引用的文本。
>> 这是嵌套的引用。
```
## 7.链接与媒体
```
行内链接：[链接文本](https://example.com)
带标题的链接：[链接文本](https://example.com "鼠标悬停显示的标题")
```
# Linux-Wiki
## ArchLinux 安装与配置
### 连接网络
```bash
iwctl device list
iwctl station wlan0 scan
iwctl station wlan0 get-networks
iwctl station wlan0 connect "WiFi名称"
```
### 更新系统时间
```bash
timedatectl
```
### 创建硬盘分区
彻底清除磁盘分区信息
```bash
wipefs -a /dev/sda
blkdiscard /dev/sda
```
#### 不加密磁盘分区方案
```bash
gdisk /dev/sda
# 输入: x (进入专家模式)
# 输入: z (清空所有分区表)
# 输入: o (创建 GPT 分区格式)
# 输入: n (创建新分区)
# 结构: EFI 分区：ef00, 主分区：8300
# 输入: p (查看分区情况)
# 输入: w (写入磁盘更改)
# 格式化分区
mkfs.fat -F 32 /dev/sda1  # EFI 分区
mkfs.ext4 /dev/sda2       # 系统根分区
# 挂载分区
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```
#### 加密磁盘分区方案
```bash
1. gdisk创建分区
lsblk
gdisk /dev/sda
# 输入: x (进入专家模式)
# 输入: z (清空所有分区表)
# 输入: o (创建 GPT 分区格式)
# 输入: n (创建新分区，hex code ef00)
# 输入: n (余下所有空间 hex code 8309）
# 输入: w (写入磁盘更改)
2. 设置LUKS加密容器
# 格式化LUKS分区
cryptsetup luksFormat /dev/sda2
# 打开LUKS容器，命名为cryptlvm
cryptsetup open /dev/sda2 cryptlvm
# 设置lvm（卷组名称dellinuxvg）
1. 在打开的 LUKS 容器上创建物理卷(PV)
pvcreate /dev/mapper/cryptlvm
2. 创建卷组(VG)，命名为 dellinuxvg
vgcreate dellinuxvg /dev/mapper/cryptlvm
3. 创建逻辑卷(LV)，只创建一个 root
lvcreate -l 100%FREE dellinuxvg -n dellinuxroot
# 格式化并挂载分区
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/dellinuxvg/dellinuxroot
mount /dev/dellinuxvg/dellinuxroot /mnt
mount --mkdir /dev/sda1 /mnt/boot
```
### 配置 Pacman 镜像源
```bash
vim /etc/pacman.d/mirrorlist
```
### 安装基础系统
```bash
pacstrap -K /mnt base linux linux-firmware linux-headers intel-ucode git base-devel dkms（+lvm2，对于采用lvm LUKS加密方案）
```
### 生成 fstab 文件
```bash
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab  # 检查文件
```
### chroot进入新系统
```bash
arch-chroot /mnt
```
### 安装必要软件包
```bash
pacman -S plasma-login-manager plasma-desktop kwalletmanager kscreen krdp konsole dolphin sudo vim networkmanager networkmanager-openvpn plasma-nm noto-fonts-cjk
```
### 设置时区与时间
```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```
### 区域与本地化
```bash
vim /etc/locale.gen
# 取消注释: en_US.UTF-8 UTF-8 和 zh_CN.UTF-8 UTF-8
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
locale-gen
```
### 主机名与用户
```bash
echo 'your-hostname' > /etc/hostname
passwd # 设置 Root 密码
useradd -m asuraarch
passwd asuraarch # 设置用户密码
EDITOR=vim visudo # 配置 sudo
```
### 对于加密磁盘方案，配置 mkinitcpio.conf
```bash
vim /etc/mkinitcpio.conf
# 样式：
HOOKS=(base systemd autodetect keyboard modconf block sd-encrypt lvm2 filesystems fsck)
# 重新生成initramfs
mkinitcpio -P
```
### 配置引导
1. 不加密磁盘方案采用GRUB引导
```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```
2.加密磁盘方案采用systemd-boot引导
安装systemd-boot到boot分区
```bash
bootctl install
# 获取UUID
blkid /dev/sda2
vim /boot/loader/entries/arch.conf
# 写入：
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options rd.luks.name=<UUID>=cryptlvm root=/dev/dellinuxvg/dellinuxroot rw
```
### 启用系统服务
```bash
systemctl enable plasmalogin
systemctl enable NetworkManager
```
### 退出并卸载分区
```bash
exit
umount -R /mnt
```
### 配置TPM自动解锁
将 TPM 与 LUKS 分区绑定
```bash
sudo systemd-cryptenroll --tpm2-device=auto /dev/sda2
vim /etc/crypttab   # 找到对应 cryptlvm 的那一行（如果没有，需要手动创建它），并在其选项末尾添加 tpm2-device=auto :
cryptlvm UUID=your-luks-uuid none discard
改为：
cryptlvm UUID=your-luks-uuid none discard,tpm2-device=auto
```
### 镜像源 (MirrorList)
```bash
Server = https://mirror.csclub.uwaterloo.ca/archlinux/$repo/os/$arch
Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch
Server = https://fastly.mirror.pkgbuild.com/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
```
### NVIDIA驱动（930M）
```bash
sudo pacman -S git base-devel
git clone https://aur.archlinux.org/nvidia-580xx-utils.git
cd nvidia-580xx-utils
makepkg -si
```
### 虚拟机进一步配置 (Optional)
#### VMware 配置
```bash
sudo pacman -S open-vm-tools gtkmm3
sudo systemctl enable vmtoolsd vmware-vmblock-fuse
```
**注意**: 开启 Vmware 3D 图形支持，分配内存 4G。
#### Hyper-V 配置
```bash
sudo pacman -S hyperv
sudo systemctl enable hv_kvp_daemon.service hv_vss_daemon.service
```
**网络配置 (网络列表混乱需重新绑定)**
```bash
nmcli device # 列出所有连接配置文件
sudo nmcli connection delete OutNet # 删除现有配置
sudo nmcli connection add type ethernet con-name InNet ifname ens37
sudo nmcli connection add type ethernet con-name OutNet ifname ens37
sudo systemctl restart NetworkManager
### Fcitx5输入法
```bash
sudo pacman -S fcitx5 fcitx5-configtool fcitx5-chinese-addons fcitx5-gtk fcitx5-qt
```
## Debian13简单配置
### 普通用户加入sudo
```bash
su -
usermod -aG sudo asurada
```
### Debin13 USTC sources.list
```bash
sudo nano /etc/apt/sources.list
# USTC
deb http://mirrors.ustc.edu.cn/debian trixie main contrib non-free non-free-firmware
# deb-src http://mirrors.ustc.edu.cn/debian trixie main contrib non-free non-free-firmware
deb http://mirrors.ustc.edu.cn/debian trixie-updates main contrib non-free non-free-firmware
# deb-src http://mirrors.ustc.edu.cn/debian trixie-updates main contrib non-free non-free-firmware
# backports
# deb http://mirrors.ustc.edu.cn/debian trixie-backports main contrib non-free non-free-firmware
# deb-src http://mirrors.ustc.edu.cn/debian trixie-backports main contrib non-free non-free-firmware
# Security
deb http://mirrors.ustc.edu.cn/debian-security/ trixie-security main contrib non-free non-free-firmware
# deb-src http://mirrors.ustc.edu.cn/debian-security/ trixie-security main contrib non-free non-free-firmware
```
### 配置TPM自动解锁
```bash
# 确定加密分区
sudo lsblk -f
# 清除旧信息
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/sda3
# 安装clevis组件
sudo apt install clevis clevis-tpm2 clevis-luks clevis-initramfs
# 执行绑定
sudo clevis luks bind -d /dev/sda3 tpm2 '{"pcr_ids":"7"}'
# 更新 initramfs
sudo update-initramfs -u -k all
```
### 配置NVIDIA驱动
```bash
# 先行安装的条件
sudo apt install linux-headers-$(uname -r) dkms build-essential
# 安装NVIDIA包
sudo apt install nvidia-driver
```
### Dell鼠标/触摸板配置
psmouse serio1: Failed to enable mouse on isa0060/serio1
```bash
sudo nano /etc/default/grub
# 在 GRUB_CMDLINE_LINUX_DEFAULT 这一行，在引号内的末尾添加 i8042.nopnp=1 参数：
GRUB_CMDLINE_LINUX_DEFAULT="quiet i8042.nopnp=1"
# 更新grub
sudo update-grub
```
## Oracle服务器
### SSH 连接
```bash
# 通用连接
ssh -i ~/Oracle9092.key username@Public_IP
# 修改ssh默认端口
sudo nano /etc/ssh/sshd_config
找到：
#Port 22
把它改成（去掉 # 并修改端口号）：
Port xx
# Android连接 (Termux)
termux-change-repo
termux-setup-storage
pkg upgrade
ssh -i "Oracle.Key路径:~/storage/downloads/Oracle9092.key" username@Public_IP
```
### 安装xfce桌面与远程桌面
```bash
sudo apt install xfce4 xfce4-goodies xrdp fonts-noto-cjk fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3 fcitx5-frontend-qt5 fcitx5-config-qt
sudo systemctl enable xrdp
sudo adduser xrdp ssl-cert
echo "xfce4-session" > ~/.xsession
sudo passwd debian
```
### 设置中文语言
```bash
sudo vim /etc/locale.gen
# 取消注释 en_US.UTF-8 和 zh_CN.UTF-8
sudo locale-gen
sudo update-locale LANG=zh_CN.UTF-8
```
## sudo免密时间长度
```bash
# 使用visudo编辑
EDITOR=vim visudo
# 添加内容，表示60分钟内免密执行
Defaults env_reset,timestamp_timeout=60
```
## 配置/swapfile 文件
```
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show
sudo nano /etc/fstab
#Debian
# /swapfile
/swapfile none swap sw 0 0
# Arch Linux
# /swapfile
/swapfile none swap defaults 0 0
```
## 安装kvm/qemu虚拟机
```bash
# Debian
sudo apt install qemu-system-x86 libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
sudo apt install virtiofsd
# Arch Linux
sudo pacman -S qemu-desktop virt-manager virt-viewer libvirt edk2-ovmf dnsmasq openbsd-netcat
---
# 加入用户组
sudo adduser $USER libvirt
sudo adduser $USER kvm
# 启动服务并设置开启自启动
sudo systemctl enable --now libvirtd
# 设置虚拟机网络自动启动
sudo virsh net-list --all
sudo virsh net-autostart default
```
KVM虚拟机与宿主机传输文件（virtiofs）
前提条件
- 虚拟机已关闭
- 宿主机为 Linux 系统
1. 打开 `virt-manager`，选择目标虚拟机，点击 **"打开"** 进入详情界面--内存，勾选共享内存
2. 点击左下角的 **"添加硬件"**，选择 **"文件系统"**，**驱动程序** 选择 `virtiofs`，**源路径** 点击 **"浏览"**，选择宿主机上要共享的文件夹，**目标路径** 填入一个挂载标签，例如 `sharedhost`
3. 自动挂载配置
```bash
vim /etc/fstab
# <挂载标签> <挂载点>  <文件系统类型>  <挂载选项>                                <dump> <pass>
Tab_Virtiofs /mnt/sharefolder  virtiofs          rw,noatime,nofail,x-systemd.automount 0 0
```
3. 手动挂载
```bash
mkdir -p /sharefolder   # 创建挂载点
sudo mount -t virtiofs sharehost ~/sharefolder
```
## 安装OpenVPN
```bash
# Debian
sudo apt install network-manager-openvpn
# Arch Linux
sudo pacman -S networkmanager-openvpn
```
## 磁盘相关操作
### 加密设备挂载/卸载
```bash
# 挂载设备
mkdir -p ~/mount
sudo cryptsetup luksOpen /dev/sdx sdcard
sudo mount /dev/mapper/sdcard ~/mount
# 卸载设备
sudo umount ~/mount
sudo cryptsetup luksClose sdcard
# 权限设置
sudo chown -R adurada:asurada ~/TestFolder
# 断开设备电源
sudo udisksctl power-off -b /dev/sdx
```
### smartctl查看磁盘健康状态
```bash
sudo smartctl -H /dev/sda
sudo smartctl -a /dev/sda
```
## Docker
### Docker安装
```bash
# Arch Linux
sudo pacman -S docker
sudo systemctl start docker
sudo systemctl enable docker.socket
# Debian
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
sudo systemctl enable --now docker
# 用户加入Docker组
sudo usermod -aG docker $USER
```
### Docker基础操作
彻底删除某个 Compose 项目
```bash
cd ＂docker项目目录＂
docker compose down -v --rmi all --remove-orphans
# 检查残留数据卷
docker volume ls  
# 删除项目文件
rm -rf ~/docker/项目文件夹名称 
```
## 常用Docker项目部署
### Docker-OpenVPN
```bash
# 开启 IP 转发
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/net_openvpn.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/net_openvpn.conf
sudo sysctl -p /etc/sysctl.d/net_openvpn.conf
# 运行容器
docker run -d \
  --name openvpn \
  --restart=always --network host \
  -v openvpn-data:/etc/openvpn \
  -e VPN_PORT=9092 -e VPN_PROTO=tcp \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  hwdsl2/openvpn-server
# 管理客户端
docker exec openvpn ovpn_manage --addclient Oracle02 # 新建OpenVPN客户端文件
docker cp openvpn:/etc/openvpn/clients/Oracle02.ovpn . # 从docker复制文件到主机目录
```
### Docker-Alist
```bash
mkdir -p ~/docker/alist # 创建项目目录
docker run -d \
--name=alist \
--restart=always --network=host \
-v ~/docker/alist:/opt/alist/data \
-e PUID=1000 -e PGID=1000 \
xhofe/alist:latest
# 设置密码
sudo docker exec -it alist ./alist admin set your-password
```
### Docker-Aria2
```bash
mkdir -p ~/docker/aria2/config # 创建项目目录
docker run -d \
--name aria2-pro \
--net=host \
--restart unless-stopped \
-e PUID=1000 \
-e PGID=1000 \
-e RPC_SECRET=your-secret \
-e RPC_PORT=6800 \
-v ~/docker/aria2/config:/config \
-v ~/docker/aria2/:/downloads \
p3terx/aria2-pro
```
### Docker-Code-Server(Web Coding)
```bash
mkdir -p ~/docker/code-server #创建项目目录
docker run -d \
--name=code-server \
--restart=always \
--network=host \
-v ~/docker/code-server:/config \
-e PUID=1000 \
-e PGID=1000 \
-e TZ=Asia/Shanghai \
-e PASSWORD="your-password" \
lscr.io/linuxserver/code-server:latest
```
### Docker-Chromium
```bash
mkdir -p ~/docker/chromium # 创建项目目录
docker run -d \
--name=chromium \
--net=host \
--security-opt seccomp=unconfined \
-e PUID=1000 \
-e PGID=1000 \
-e TZ=Asia/Shanghai \
-v /home/asurada/docker/chromium:/config --shm-size=2g \
--restart unless-stopped \
lscr.io/linuxserver/chromium:latest
```
### Metasploit
```bash
mkdir -p ~/docker/metasploit
docker run -it -d \
--name metasploit \
--net=host \
-v ~/docker/metasploit:/root/.msf4 \
metasploitframework/metasploit-framework
# 预先在 PostgreSQL 中创建数据库
docker exec -it postgresql psql -U postgres
# CREATE USER user_msf WITH PASSWORD 'password';
# CREATE DATABASE data_msf OWNER user_msf;
# 连接 msf 并手动连接数据库
docker exec -it metasploit msfconsole
# db_connect user_msf:password@127.0.0.1:5432/data_msf
# db_status
```
### Docker-NetData (监控面板)
```bash
mkdir -p ~/docker/netdata #创建项目目录
docker run -d \
--name=netdata \
--pid=host \
--network=host \
-e NETDATA_IP=10.8.0.1 \
-v netdataconfig:/etc/netdata \
-v netdatalib:/var/lib/netdata \
-v netdatacache:/var/cache/netdata \
-v /:/host/root:ro,rslave \
-v /etc/passwd:/host/etc/passwd:ro \
-v /etc/group:/host/etc/group:ro \
-v /etc/localtime:/etc/localtime:ro \
-v /proc:/host/proc:ro \
-v /sys:/host/sys:ro \
-v /etc/os-release:/host/etc/os-release:ro \ -v /var/log:/host/var/log:ro \
-v /var/run/docker.sock:/var/run/docker.sock:ro \
-v /run/dbus:/run/dbus:ro \
--restart unless-stopped \
--cap-add SYS_PTRACE \
--cap-add SYS_ADMIN \
--security-opt apparmor=unconfined \
netdata/netdata:stable
# 访问
`http://10.8.0.1:19999`
```
### PostgreSQL
```bash
mkdir -p ~/docker/postgresql # 创建项目目录
docker run -d \
--name postgresql \
--net=host \
-e POSTGRES_PASSWORD=your-password \
-v ~/docker/postgresql:/var/lib/postgresql/ \
--restart unless-stopped \
postgres:latest
# 连接并操作
docker exec -it postgresql psql -U postgres
# 修改密码: ALTER USER postgres WITH PASSWORD 'new-password';
# 创建用户: CREATE USER user_asurada WITH PASSWORD 'password';
# 创建数据库: CREATE DATABASE data_asurada OWNER asurada;
# 退出: \q
# 连接新数据库: psql -h 127.0.0.1 -p 5432 -U user_asurada -d data_asurada
```
#### PostgreSQL 备份,恢复与导入
**备份**
```bash
pg_dump -U user_asurada -d data_asurada --section=pre-data --section=data -F c -Z 9 --file=~/asura_data.dump
# 验证: ls -lh /home/asuraarch/asura_data.dump && echo $? (输出 0 为成功)
# 列出内容: pg_restore -l ~/asura_data.dump
```
**恢复**
```bash
# 前提：已创建好目标用户和数据库
pg_restore --section=pre-data --section=data -v -U user_asurada -d data_asurada --noowner ~/asura_data.dump
# 检查: \dt, \dv, \ds, \d table_name, SELECT COUNT(*) FROM table_name;
```
**删除数据库**
```bash
psql -U user_asurada -d data_asurada -c "DROP DATABASE data_asurada;"
```
**PostgreSQL 数据导入示例**
```bash
# 创建表
CREATE TABLE table_2026 ( phone VARCHAR(20), uid VARCHAR(50) );
# 导入数据
\copy table_2026 (phone, uid) FROM '~/Test.txt' DELIMITER E'\t' CSV HEADER;
# 创建索引
ANALYZE table_2026;
CREATE INDEX idx_phone ON table_2026("phone");
CREATE INDEX idx_uid ON table_2026("uid");
# 查询: SELECT * FROM table_2026 WHERE "phone" = 'Your Phone';
```
## yay
```bash
# Arch Linux
sudo pacman -S git base-devel #如果未安装需提前安装
cd ~
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
```
## Google Chrome
```bash
# Arch Linux
yay -S google-chrome
```
## Chromium
```bash
# Debian
sudo apt install chromium chromium-l10n
# Arch Linux
sudo pacman -S chromium
```
## Fcitx5
```bash
# Arch Linux
sudo pacman -S fcitx5 fcitx5-configtool fcitx5-qt fcitx5-gtk fcitx5-chinese-addons
```
## Kwallet
建议设置空密码
## crunch
```bash
yay -S crunch
```
## swaks
```bash
sudo pacman -S swaks
```
## hashcat
```bash
yay hashcat
```
## wireshark
```bash
yay wireshark
```
## nmap
```bash
sudo pacman -S nmap
```
## Aircrack-NG
```bash
# Arch Linux
yay -S aircrack-ng
ifconfig                      # 查看当前网卡
sudo airmon-ng                # 查看外接网卡
sudo airmon-ng start wls35u1  # 开启监听模式 (wls35u1为接口名)
sudo airodump-ng wls35u1mon   # 扫描网络
# sudo airodump-ng -c [CH频道] --bssid [路由器MAC] -w cap文件名 wls35u1mon   # 抓取cap包
sudo aireplay-ng -0 10 -a [路由器MAC] wls35u1
sudo aireplay-ng -0 0 -a [路由器ID] -c [设备ID] wls35u1mon # 断网攻击 (Deauth)
aircrack-ng xxx.cap -w 字典路径 # 字典破解
sudo iw dev wls35u1mon set type managed # 停止监听模式 (推荐)
sudo airmon-ng stop wls35u1mon # 不推荐，遇到网卡、内核和桌面崩溃
```
## 手机 USB 共享网络
```bash
ip a                     # 查看设备接口
ip link set <接口名> up   # 启用接口
dhcpcd <接口名>           # 通过 DHCP 获取地址
ping debian.org -c 3     # 验证网络
ip route show            # 查看路由
ip route del default via 192.168.100.1 # 删除无效路由:
```
## 查看系统内核/硬件日志
```bash
sudo dmesg | tail -n 50
# 从内核日志中，过滤并高亮显示与 sda 硬盘、SATA 接口、错误（error）或失败（fail）相关的关键日志
sudo dmesg | grep -i -E "sda|ata|error|fail"
```
# Windows系统相关
## 系统设置
- **BIOS**: Dell 进入 F2，选择启动项 F12。
- **设置中心**: 设置密码、个性化、高性能电源模式，关闭 U 盘自动运行。
- **安全中心**: 关闭病毒防护、文件审查。
- **Windows 更新**: 关闭"从其他计算机下载"。
- **控制面板**:
  - 系统和安全: 关闭自动磁盘优化。
  - 电源设置: 关闭快速启动，高级设置中设置电池最小 CPU。
- **服务**: 禁用三个 Edge 更新服务。
- **组策略**: 关闭自动更新、系统还原、防病毒。
- **关闭休眠**: `powercfg -h off`
- **文件管理器**: 显示隐藏文件和后缀，修改EdgeUpdate文件夹权限。
## Hyper-V 管理 (PowerShell)
```powershell
# 启用嵌套虚拟化
Set-VMProcessor -VMName 虚拟机名称 -ExposeVirtualizationExtensions $true
# 关闭嵌套虚拟化
Set-VMProcessor -VMName 虚拟机名称 -ExposeVirtualizationExtensions $false
# Mac 地址欺骗 (可选)
Get-VMNetworkAdapter -VMName 虚拟机名称 | Set-VMNetworkAdapter -MacAddressSpoofing On
```
## 基于 GPT+UEFI 的磁盘分区命令
```cmd
list disk
select disk 0
clean
convert gpt
create partition efi size=200
format quick fs=fat32 label="EFI"
assign letter="S"
create partition msr size=16
create partition primary 
shrink minimum=1024
format quick fs=ntfs label="Windows"
assign letter="W"
create partition primary
format quick fs=ntfs label="Recovery"
assign letter="R"
set id="de94bba4-06d1-4d40-a16a-bfd50179d6ac"
gpt attributes=0x8000000000000001
list volume
exit
```
## DISM备份驱动
```cmd
DISM.exe /Online /Export-Driver /Destination:C:\Drivers\
# 安装系统后，在设备管理器中导入备份的驱动。
```
## 校验Hash值
```cmd
certutil -hashfile ~\xxx.iso SHA256
```
## 命令行复制文件
```cmd
# 复制文件夹
robocopy E:\Folder F:\Folder /E
# 复制单个文件
robocopy E:\ F:\ 文件名.后缀
```
