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
### 硬盘分区
```bash
# 不加密分区方案
gdisk /dev/sdx
# 输入: x (进入专家模式)
# 输入: z (清空所有分区表)
# 输入: o (创建 GPT 分区格式)
# 输入: n (创建新分区)
# 结构: EFI 分区：ef00, 主分区：8300
# 输入: p (查看分区情况)
# 输入: w (写入磁盘更改)
# 格式化分区
mkfs.fat -F 32 /dev/sdx1  # EFI 分区
mkfs.ext4 /dev/sdx2       # 系统根分区
# 挂载分区
mount /dev/sdx2 /mnt
mount --mkdir /dev/sdx1 /mnt/boot
```
```bash
# 加密分区方案
lsblk
gdisk /dev/sdx
# 输入: x (进入专家模式)
# 输入: z (清空所有分区表)
# 输入: o (创建 GPT 分区格式)
# 输入: n (创建新分区，hex code ef00)
# 输入: n (余下所有空间 hex code 8309）
# 输入: w (写入磁盘更改)
# 格式化LUKS分区
cryptsetup luksFormat /dev/sdx2
# 打开LUKS
cryptsetup open /dev/sdx2 cryptlvm
# 设置lvm
pvcreate /dev/mapper/cryptlvm
vgcreate macvg /dev/mapper/cryptlvm
lvcreate -l 100%FREE macvg -n macroot
# 格式化并挂载分区
mkfs.fat -F 32 /dev/sdx1
mkfs.ext4 /dev/macvg/macroot
mount /dev/macvg/macroot /mnt
mount --mkdir /dev/sdx1 /mnt/boot
```
### 配置镜像源
```bash
vim /etc/pacman.d/mirrorlist
```
>
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch  
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch  
Server = https://mirror.csclub.uwaterloo.ca/archlinux/$repo/os/$arch  
Server = https://geo.mirror.pkgbuild.com/$repo/os/$arch  
Server = https://fastly.mirror.pkgbuild.com/$repo/os/$arch    
### 安装基础系统
```bash
pacstrap -K /mnt base linux linux-firmware linux-headers intel-ucode git base-devel dkms lvm2 cryptsetup(对于采用lvm LUKS加密方案）
```
### 生成 fstab 文件
```bash
genfstab -U /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab  # 检查fstab文件
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
### locale和hostname设置
```bash
vim /etc/locale.gen
# 取消注释: en_US.UTF-8 UTF-8 和 zh_CN.UTF-8 UTF-8  
echo 'LANG=en_US.UTF-8' > /etc/locale.conf  
echo 'arch' > /etc/hostname  
locale-gen  
```
### 用户配置
```bash
passwd # 设置 Root 密码
useradd -m asurada
passwd asurada # 设置用户密码
EDITOR=vim visudo # 配置 sudo
```
### 配置 mkinitcpio.conf（采用磁盘加密方案按需配置）
```bash
vim /etc/mkinitcpio.conf
# 重新生成initramfs
mkinitcpio -P
# 样式：
HOOKS=(base systemd autodetect keyboard modconf block (sd-encrypt lvm2) filesystems fsck)
```
### 配置引导
```bash
# 不加密磁盘方案采用GRUB引导
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```
```bash
# 加密磁盘方案采用systemd-boot引导
bootctl install
vim /boot/loader/loader.conf
# 写入：
default  arch.conf  
timeout  3  
console-mode max  
editor   no 
# 获取UUID
blkid /dev/sdx2
vim /boot/loader/entries/arch.conf
# 写入：
title   Arch Linux  
linux   /vmlinuz-linux  
initrd  /intel-ucode.img  
initrd  /initramfs-linux.img  
# 磁盘加密写法
options rd.luks.name=<UUID>=cryptlvm root=/dev/macvg/macroot rw
# 不加密磁盘写法
options root=UUID=<UUID> rw
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
```bash
sudo systemd-cryptenroll --tpm2-device=auto /dev/sda2
vim /etc/crypttab
```
>
找到对应 cryptlvm 的那一行（如果没有，需要手动创建它），并在其选项末尾添加 tpm2-device=auto :
cryptlvm UUID=your-luks-uuid none discard
改为：
cryptlvm UUID=your-luks-uuid none discard,tpm2-device=auto
### sudo免密时间长度
```bash
EDITOR=vim visudo
# 写入：
Defaults env_reset,timestamp_timeout=60
```
### 配置/swapfile 文件
```
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon --show
sudo nano /etc/fstab
# 写入：
/swapfile none swap defaults 0 0
```
### MacBookPro优化
#### 电源管理
```bash
sudo pacman -S tlp tlp-rdw
yay -S mbpfan-git
sudo systemctl enable --now tlp
sudo systemctl enable --now mbpfan
```
#### 声卡
```bash
sudo pacman -S alsa-utils
sudo pacman -S pipewire-pulse pipewire-alsa
systemctl --user enable --now pipewire-pulse.service
alsamixer
# 声音图标
sudo pacman -S plasma-pa
```
#### 蓝牙
```bash
sudo pacman -S bluedevil bluez bluez-utils
sudo systemctl enable --now bluetooth.service
```
#### MacBook启动项管理
> 针对于重装Linux系统之后，启动时MacBook转圈等待时间过长，此处方案对于使用systemd-boot引导
```bash
# 确定问题所在
sudo pacman -S efibootmgr
systemd-analyze
efibootmgr
# 查看输出里的BootOrder。如果排在前面的 BootXXXX 条目不是指向systemd-boot（通常叫 Linux Boot Manager，路径类似 \EFI\systemd\systemd-bootx64.efi），就需要把它调到第一位。
假设systemd-boot条目是Boot0001，可以这样调整：
sudo efibootmgr -o 0001,其他的编号...
# 用efibootmgr删除Boot0080
sudo efibootmgr -b 0080 -B
```
  - efibootmgr：操作UEFI启动项的命令行工具
  - -b 0080：-b 是 --bootnum，指定要操作的启动项编号。这里 0080 就是efibootmgr输出里的Boot0080
  - -B：--delete-bootnum，删除-b指定的那个启动项
- - 整条命令的意思：删除编号为Boot0080的UEFI启动条目,Boot0080是当前BootOrder里唯一指systemd-boot的条目。删掉它之后,NVRAM里就没有有效的启动项了。这时Mac 固件会回退到默认的后备路径\EFI\BOOT\BOOTX64.EFI去寻找可启动文件
### KVM/QEMU虚拟机
#### 安装KVM/QEMU
```bash
sudo pacman -S qemu-desktop virt-manager virt-viewer libvirt edk2-ovmf dnsmasq
# 加入用户组
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
# 启动服务并设置开启自启动
sudo systemctl enable --now libvirtd
# 设置虚拟机网络自动启动
sudo virsh net-list --all
sudo virsh net-autostart default
```
#### KVM虚拟机与宿主机传输文件（virtiofs）
- 虚拟机已关闭
- 宿主机为 Linux 系统
  - 打开 `virt-manager`，选择目标虚拟机，点击 **"打开"** 进入详情界面--内存，勾选共享内存
  - 点击左下角的 **"添加硬件"**，选择 **"文件系统"**，**驱动程序** 选择 `virtiofs`，**源路径** 点击 **"浏览"**，选择宿主机上要共享的文件夹，**目标路径** 填入一个挂载标签，例如 `SharedHost`
  - 自动挂载配置
```bash
mkdir -p /ShareFolder
vim /etc/fstab
# 写入：
Virtiofs /mnt/sharefolder  virtiofs  rw,noatime,nofail,x-systemd.automount 0 0
```
  - 手动挂载
```bash
# 创建挂载点
mkdir -p /ShareFolder  
sudo mount -t virtiofs ShareHost ~/ShareFolder  
```
### 磁盘操作
```bash
清除磁盘
wipefs -a /dev/sda
blkdiscard /dev/sda
待补充完善
```
```bash
加密设备挂载/卸载
挂载设备
mkdir -p ~/mount
sudo cryptsetup luksOpen /dev/sdx sdcard
sudo mount /dev/mapper/sdcard ~/mount
卸载设备
sudo umount ~/mount
sudo cryptsetup luksClose sdcard
权限设置
sudo chown -R username:username ~/TestFolder
断开设备电源
sudo udisksctl power-off -b /dev/sdx
```
```bash
smartctl查看磁盘健康状态
sudo pacman -S smar?tool
sudo smartctl -H /dev/sda
sudo smartctl -a /dev/sda
```
### konsole SSH连接记录
```bash
# 对于已保存的SSH连接，如果连接的目标主机经过重装系统后连接不上，只需要删除该文件中对应的主机条目即可
sudo vim ~/.ssh/known_hosts
```
### yay
```bash
sudo pacman -S git base-devel 
# 如果未安装需提前安装
cd ~
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin
makepkg -si
```
### Fcitx5输入法
```bash
sudo pacman -S fcitx5 fcitx5-configtool fcitx5-chinese-addons fcitx5-gtk fcitx5-qt
```
### Google Chrome
```bash
yay -S google-chrome
```
### Chromium
```bash
sudo pacman -S chromium
```
### LibreOffice
```bash
sudo pacman -S libreoffice-still libreoffice-still-zh-cn
```
### Kwallet
```bash
建议设置空密码
```
### crunch
```bash
yay -S crunch
```
### swaks
```bash
sudo pacman -S swaks
```
### hashcat
```bash
yay hashcat
```
### wireshark
```bash
yay wireshark
```
### nmap
```bash
sudo pacman -S nmap
```
### Aircrack-NG
```bash
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
### 手机 USB 共享网络
```bash
ip a                     # 查看设备接口
ip link set <接口名> up   # 启用接口
dhcpcd <接口名>           # 通过 DHCP 获取地址
ping debian.org -c 3     # 验证网络
ip route show            # 查看路由
ip route del default via 192.168.100.1 # 删除无效路由:
```
### 查看系统内核/硬件日志
```bash
sudo dmesg | tail -n 50
# 从内核日志中，过滤并高亮显示与 sda 硬盘、SATA 接口、错误（error）或失败（fail）相关的关键日志
sudo dmesg | grep -i -E "sda|ata|error|fail"
```
### Docker
#### Docker安装
```bash
sudo pacman -S docker
sudo systemctl start docker
sudo systemctl enable docker.socket
# 加入组
sudo usermod -aG docker $USER
```
#### Docker基础操作
- 彻底删除某个Compose项目
```bash
cd docker项目目录
docker compose down -v --rmi all --remove-orphans
```
- 检查残留数据卷
```
docker volume ls
```
- 删除项目文件
```
rm -rf ~/docker/项目文件夹名称 
```
#### Docker-OpenVPN
- 可选配置
```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/net_openvpn.conf
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/net_openvpn.conf
sudo sysctl -p /etc/sysctl.d/net_openvpn.conf
```
```
docker run -d \
  --name openvpn \
  --restart=always --network host \
  -v openvpn-data:/etc/openvpn \
  -e VPN_PORT=9092 \
  -e VPN_PROTO=tcp \
  --cap-add=NET_ADMIN \
  --device=/dev/net/tun \
  hwdsl2/openvpn-server
```
- 新建OpenVPN客户端文件
```
docker exec openvpn ovpn_manage --addclient Oracle
```
- 从docker复制文件到主机目录
```
docker cp openvpn:/etc/openvpn/clients/Oracle02.ovpn .
```
- 从云主机复制到本地设备
```
scp -i /storage/emulated/0/Download/Oracle/private.key username@Public:~/Oracle.ovpn /storage/emulated/0/Download/Oracle/
scp -i /home/asurada/Downloads/Oracle/private.key username@Public:~/Oracle.ovpn /home/asurada/Downloads/Oracle/
```
#### Docker-Alist
```bash
# 创建项目目录
mkdir -p ~/docker/alist  
# 运行
docker run -d \
--name=alist \
--restart=always --network=host \
-v ~/docker/alist:/opt/alist/data \
-e PUID=1000 -e PGID=1000 \
xhofe/alist:latest  
# 设置密码
sudo docker exec -it alist ./alist admin set your-password  
```
#### Docker-Aria2
```bash
# 创建项目目录
mkdir -p ~/docker/aria2/config
# 运行  
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
#### Docker-Code-Server(Web Coding)
```bash
# 创建项目目录
mkdir -p ~/docker/code-server  
# 运行
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
#### Docker-Chromium
```bash
# 创建项目目录
mkdir -p ~/docker/chromium 
# 运行 
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
#### Docker-NetData (监控面板)
```bash
# 创建项目目录
mkdir -p ~/docker/netdata 
# 运行 
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
#### Metasploit
```bash
# 创建项目目录
mkdir -p ~/docker/metasploit  
# 运行
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
#### PostgreSQL
```bash
# 创建项目目录
mkdir -p ~/docker/postgresql  
# 运行
docker run -d \
--name postgresql \
--net=host \
-e POSTGRES_PASSWORD=your-password \
-v ~/docker/postgresql:/var/lib/postgresql/ \
--restart unless-stopped \
postgres:latest
# 连接并操作
docker exec -it postgresql psql -U postgres
# 修改密码
ALTER USER postgres WITH PASSWORD 'new-password';
# 创建用户
CREATE USER user_asurada WITH PASSWORD 'password';
# 创建数据库
CREATE DATABASE data_asurada OWNER asurada;
# 退出
\q
# 连接新数据库
psql -h 127.0.0.1 -p 5432 -U user_asurada -d data_asurada  
```
#### PostgreSQL 备份,恢复与导入
**备份**
```bash
pg_dump -U user_asurada -d data_asurada --section=pre-data --section=data -F c -Z 9 --file=~/asura_data.dump  
# 验证
ls -lh /home/asuraarch/asura_data.dump && echo $?  
(输出 0 为成功)
# 列出内容
pg_restore -l ~/asura_data.dump  
```
**恢复**
```bash
# 前提：已创建好目标用户和数据库
pg_restore --section=pre-data --section=data -v -U user_asurada -d data_asurada --noowner ~/asura_data.dump  
# 检查
\dt, \dv, \ds, \d table_name, SELECT COUNT(*) FROM table_name;  
```
**删除数据库**
```bash
psql -U user_asurada -d data_asurada -c "DROP DATABASE data_asurada;"  
```
**PostgreSQL数据导入示例**
```bash
# 创建表
CREATE TABLE table_2026 ( phone VARCHAR(20), uid VARCHAR(50) );  
# 导入数据
\copy table_2026 (phone, uid) FROM '~/Test.txt' DELIMITER E'\t' CSV HEADER;  
# 创建索引
ANALYZE table_2026;  
CREATE INDEX idx_phone ON   table_2026("phone");  
CREATE INDEX idx_uid ON table_2026("uid");  
# 查询: SELECT * FROM table_2026 WHERE "phone" = 'Your Phone';
```
## Debian On Oracle&KVM/QEMU
### SSH 连接
```bash
# 通用连接
ssh -i ~/private.key username@Public_IP
# Android连接 (Termux)
termux-change-repo
termux-setup-storage
pkg upgrade
ssh -i "private.Key路径:~/storage/downloads/private.key" username@Public_IP
```
### 安装xfce桌面与远程桌面
> 安装完整的xfce桌面并默认启动后进入桌面
```bash
sudo apt install xfce4 xrdp fonts-noto-cjk
sudo systemctl enable xrdp
sudo adduser xrdp ssl-cert
echo "xfce4-session" > ~/.xsession
sudo passwd debian
```
> 最小化安装xfce桌面并默认启动后进入tty界面
```bash
sudo pacman -S xfce4 xorg-xinit xorg-server
# 创建配置文件
sudo vim ~/.xinitrc
- exec startxfce4
# 启动进入TTY界面后，如果需要登录桌面，输入：
startx
```
### 设置中文语言
```bash
sudo vim /etc/locale.gen
# 取消注释 en_US.UTF-8 和 zh_CN.UTF-8
sudo locale-gen
sudo update-locale LANG=zh_CN.UTF-8
```
### Fcitx5
```bash
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-gtk3 fcitx5-frontend-qt5 fcitx5-config-qt
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
### Debian安装Docker
```bash
sudo apt install ca-certificates curl
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
```bash
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
```
```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```bash
sudo systemctl status docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```
### Debian安装chromium
```bash
sudo apt install chromium chromium-l10n
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
