# LightNVM安装
## 0.0 宿主机(不重要)
- 由于我懒，就直接重装系统成ubuntu了。刻盘时，直接用UltraISO刻的ubuntu16.04，出现
```
Failed to load ldlinux.c32
Boot failed: please change disks and press a key to continue.
```
网上说UltraISO不支持新版了，所以使用另一个软件工具重新刻盘Win32DiskImager。
- 刻好盘以后呢，第二个磁盘挂了，LSI rebuild第二个磁盘，还是帮别人rebuild的。。。
常用键：
space 选择， F10 配置，一直F10后accept，save configuration为yes即可。
- 安装ubuntu16.04， 命黑，装了好久以后出现
```
Unable to install GRUB in /dev/md126p1.
Executing 'grub-install /dev/md126p1' Failed
This is a fatal error.
```
重刻PE盘后，删除整个分区重建MBR也没用（放弃~~)
- ubuntu 16.04 desktop装成功了，但是，不知道什么原因很慢，内存有20几G，不过磁盘比较小总共只用100多G，特别的卡
- 直接还是用centos7，找到一个在centos上LightNVM的安装教程：https://hyunyoung2.github.io/2016/10/04/LightNVM_With_OpenChannelSSD_On_QEMU/

<hr />
**安装教程：**
1. [LighNVM官网](http://openchannelssd.readthedocs.io/en/latest/gettingstarted/)
2. [LightNVM with OpenchannelSSD on QEMU-NVMe](https://hyunyoung2.github.io/2016/10/04/LightNVM_With_OpenChannelSSD_On_QEMU/)

**教程2很全**

有时软件装不上，yum update或者clean一下。

##  1. 宿主机准备
由于只是底层qemu-NVMe不同，我感觉宿主机的内核不需要升级，所以还是3.10。
### 1.1 安装qemu-kvm
```
$ sudo yum install -y qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer
$ reboot
$ sudo yum groupinstall "Development Tools"
```
如果virt-manager没有出现图形界面，那是少了某些库，比如libSDL，VNC等。
### 1.2 安装qemu-NVMe
```
$ git clone https://github.com/OpenChannelSSD/qemu-nvme.git
$ cd qemu-nvme
$ ./configure --python=/usr/bin/python2 --enable-kvm --target-list=x86_64-softmmu --enable-linux-aio --prefix=$HOME/qemu-nvme
$ make
$ make install
```
中间会报错缺少某些库
```
$ sudo yum install libaio-devel
```
重新执行安装

### 1.3 网络配置（不重要）
由于感觉在xshell下复制粘贴比较方便，所以另外想将虚拟机配置成bridge模式。
首先将/etc/sysconfig/network-scripts/下的enp1s0f1备份
- 创建ifcfg-br0文件：
```
BOOTPROTO=static
DEVICE=br0
TYPE=Bridge
NM_CONTROLLED=no
IPADDR=192.168.3.207
NETMASK=255.255.255.0
GATEWAY=192.168.3.215
DNS1=202.114.0.242
DNS2=8.8.8.8
```
- 修改原来的ifcfg-enp0s0内容如下：
```
BOOTPROTO=none
DEVICE=enp0s0
NM_CONTROLLED=no
ONBOOT=yes
BRIDGE=br0
```
- 重启网络服务
```
systemctl restart network
```
brctl命令：
```
brctl show
brctl addbr br0
brctl addif br0 eth0
brclt delbr br0
```
- 在virt-manager中虚拟机的配置中选择bridge
- 虚拟机网络配置：
/etc/sysconfig/network-scripts/中下的enp1s0f1
```
BOOTPROTO=static
DEVICE=enp0s0
NM_CONTROLLED=no
ONBOOT=yes
IPADDR=192.168.3.241
NETMASK=255.255.255.0
GATEWAY=192.168.3.215
DNS1=202.114.0.242
```
重启网络。

<hr/>
## 2. 客户机准备

### 2.1 虚拟机系统安装
先把虚拟机装好，我是直接用virt-manager安装的centos7，版本3.10。
### 2.2 升级内核版本
LightNVM is directly supported in Linux kernel 4.4+.我这里直接使用的是LightNVM提供的linux，安装的时候版本为4.14+。
- 安装必要软件包
```
yum update
yum install git -y
yum groupinstall "Development Tools"
sudo yum install -y gcc bc openssl-devel ncurses-devel pciutils libudev-devel
```
- 编译内核
```
clone https://github.com/OpenChannelSSD/linux.git
cd linux
make menuconfig
make -j8
sudo make modules_install
sudo make install
$ grub2-editenv list
$ grub2-set-default 0
```
配置选项
```
CONFIG_NVM=y
# Expose the /sys/module/lnvm/parameters/configure_debug interface
CONFIG_NVM_DEBUG=y
# Target support (required to expose the open-channel SSD as a block device)
CONFIG_NVM_PBLK=y
# For NVMe support
CONFIG_BLK_DEV_NVME=y
```
为了测试能否模块化我的配置改为了
```
CONFIG_NVM_PBLK=m
CONFIG_BLK_DEV_NVME=m
```


### 2.3 安装lnvm和nvme-cli
```
$ git clone https://github.com/OpenChannelSSD/lnvm.git  
$ make
$ make install
$ lnvm devices
```
```
$ git clone https://github.com/linux-nvme/nvme-cli.git
$ cd nvme-cli
$ make
$ make install
```
<hr/>

## 3. 初步整合运行
### 3.1 创建一个空的NVMe设备
```
dd if=/dev/zero of=blknvme bs=1M count=8196
```
用qemu-nvme启动虚拟机,qemu-system-x86_64是之前安装的qemu-nvme编译的bin中的，不是kvm的。

### 3.2 用qemu-nvme启动虚拟机
```
qemu-system-x86_64 -m 4G -smp 2 --enable-kvm
-hda $LINUXVMFILE
-drive file=blknvme,if=none,id=mynvme
-device nvme,drive=mynvme,serial=deadbeef,namespaces=1,lver=1,nlbaf=5,lba_index=3,mdts=10,lnum_lun=4,lnum_pln=1,lsecs_per_pg=8
```

### 3.3 模块加载
```
# 到drivers底下一口气把nvm全insmod了
$ insmod nvm*
# 或者modprobe nvm*
$ modprobe pblk
```
### 3.4 初始化pblk
```
nvme lnvm list   #可看到我们模拟的设备
# 初始化设备
nvme lnvm create -d nvme0n1 --lun-begin=0 --lun-end=3 -n mydevice -t pblk
```
在/dev/下可以看到mydevice
```
mkfs.ext4 /dev/mydevice
```

以上，基本配置完成~~~
<hr/>

## 4. 模块化pblk
目标：printk一个hello world。
### 4.1 禁止Linux开机自动加载pblk
1. 编辑文件/etc/modprobe.d/blacklist.conf， 添加需要禁止加载的内核模块：
```
blacklist bridge
```
2. 在/etc/modprobe.conf或/etc/modules.conf 中加入一行
```
alias $module_name off
```

## 5. 模块化编译pblk
到目录drivers/lightnvm/下，根据覃博Makefile重写目录下的makefile，在pblk_init()中加入printk，打印helloworld。
重新编译即可。insmod，rmmod，可以看到pblk工作正确。
