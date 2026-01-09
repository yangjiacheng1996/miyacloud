Debian13通过iso最小化安装后，没有apt 镜像源，系统中没有vim命令，没有sshd服务，没有桌面。
# 修改grub
```
vim /etc/default/grub
GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"

update-grub
reboot
````

# 配网络
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet static
        address 10.1.227.200/24
        #gateway 10.1.227.248
        # dns-* options are implemented by the resolvconf package, if installed
        #dns-nameservers 10.9.28.80 10.9.28.81

allow-hotplug usb0
iface usb0 inet dhcp
````

# apt源

```
# 官方源
deb http://ftp.us.debian.org/debian/ trixie main non-free non-free-firmware contrib 
deb-src http://ftp.us.debian.org/debian/ trixie main non-free non-free-firmware contrib 
deb http://ftp.us.debian.org/debian/ trixie-updates main non-free non-free-firmware contrib 
deb-src http://ftp.us.debian.org/debian/ trixie-updates main non-free non-free-firmware contrib 
deb http://ftp.us.debian.org/debian/ trixie-backports main non-free non-free-firmware contrib 
deb-src http://ftp.us.debian.org/debian/ trixie-backports main non-free non-free-firmware contrib


# 清华源
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie main non-free non-free-firmware contrib
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie main non-free non-free-firmware contrib
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie-updates main non-free non-free-firmware contrib
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie-updates main non-free non-free-firmware contrib
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie-backports main non-free non-free-firmware contrib
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ trixie-backports main non-free non-free-firmware contrib
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security trixie-security main non-free non-free-firmware contrib
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security trixie-security main non-free non-free-firmware contrib


# 更新
apt update
apt upgrade -y

```

# SSH
```
apt install -y vim ssh git
vim /etc/ssh/sshd_config
------------------------------
PermitRootLogin  yes

# 重启sshd
systemctl restart sshd
````

# 安装桌面
```
tasksel
# 按空格键，勾选Debian Desktop Environment 和 Gnome
# 按Tab键到OK，开始安装

# tasksel命令运行完成后，无任何返回。如果有报错，重新运行一次tasksel
echo $?

# 允许root用户登录桌面
vim /etc/pam.d/gdm-password
-------------------------------------
auth    requisite       pam_nologin.so
#auth   required        pam_succeed_if.so user != root quiet_success

vim /etc/pam.d/gdm-autologin
-----------------------------------
auth    requisite       pam_nologin.so
#auth   required        pam_succeed_if.so user != root quiet_success


# 重启gdm服务
systemctl restart gdm.service

# 防止休眠
systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target

# 安装远程桌面
apt install -y xrdp
systemctl enable xrdp --now

````

# 安装virt-manager
```
apt install -y virt-manager  bridge-utils

````

