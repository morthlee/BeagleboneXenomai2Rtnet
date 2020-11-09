Set up RTOS for EtherCAT master on Beaglebone Black Board under the config : Linux3.8.13 + Xenomai 2.6.3 + Rtnet

一 实时补丁准备

1.1: 下载https://github.com/RobertCNelson/bb-kernel/tree/3.8.13-xenomai, 解压至~/bbbkernel/bb-kernel-3.8.13-xenomai；

1.2： 修改~/bbbkernel/bb-kernel-3.8.13-xenomai/system.sh, 设置LINUX_GIT=~/bbbkernel/bb-kernel-3.8.13-xenomai/ignore/linux/；

1.3： 解压~/bbbkernel/bb-kernel-3.8.13-xenomai/ignore/linux.zip. linux.zip由git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git得到

1.4： 修改~/bbbkernel/bb-kernel-3.8.13-xenomai/scripts/git.sh，修改xenomai_2_6="https://gitlab.denx.de/Xenomai/xenomai.git"

1.5： 执行~/bbbkernel/bb-kernel-3.8.13-xenomai/build_deb.sh，开始编译，首先会进入内核编译选项配置界面，按照如下选项进行设置：

Power management options  --->

	Power Management Debug Support 选择为N

CPU Power Management  --->

	CPU Frequency scaling  --->
	
		CPU Frequency scaling 选择为N
		
Device Drivers  --->

Network device support  --->

Ethernet driver support  --->

	Texas Instruments (TI) devices 选择为N(取消所有TI CPSW网卡相关驱动)
	
PHY Device support and infrastructure  --->

	Drivers for SMSC PHYs 选择为N
	
配置完毕，按ESC直至退出，选择保存

1.6：	开始编译内核，时间大概需要20分钟，在/deploy文件夹下面可以找到如下几个deb包：

linux-firmware-image-3.8.13-xenomai-r86_1cross_armhf.deb

linux-headers-3.8.13-xenomai-r86_1cross_armhf.deb

linux-image-3.8.13-xenomai-r86_1cross_armhf.deb

linux-libc-dev_1cross_armhf.deb


二 主系统准备

2.1 将镜像文件bone-debian-8.7-machinekit-armhf-2017-02-12-4gb.img.xz烧录到 sd 卡。在 ubuntu 下面使用命令：
    sudo -s
    xz -dkc bone-debian-8.7-console-armhf-2017-01-23-2gb.img.xz > /dev/sdX (X替换为具体设备号)
    exit
2.2 默认烧录之后只分配了 2GB 的空间，剩余的没有分配，所以需要用 gparted 调整一下sd 卡的空间，没有安装的就 sudo apt-get install gparted 即可
2.3 将 SD 卡插入 bbb,按住 s2 按钮，上电直到 4 盏 led 灯全亮再松开 s2 按钮，此时 bbb 应该能够从 SD 卡正常启动

三 主系统更换实时内核

3.1 通过 SSH 连接 bbb: (默认用户名和密码：machinekit machinekit. 之后再设置Root 用户密码)

3.2 设置wifi:

sudo connmanctl

 tether wifi disable
 
 enable wifi
 
 scan wifi
 
 services
 
 agent on
 
 connect  <WIFI信息>
 
 Passphrase? <输入WIFI密码>
 
 quit
 
3.3 然后将上一步骤得到的几个deb包通过SSH上传至BBB板子内，假设放在/home路径下，按照如下步骤进行安装:

cd /home

sudo dpkg -i *.deb

安装完之后reboot,重新连接BBB板子，至此完成BBB板子的实时内核更换，输入uname -r 可以看到3.8.13-xenomai-r86， 输入cat /proc/xenomai/version 可以看到2.6.3

3.4 下载xenomai-2.6.3.tar.bz2，在BBB板子上进行安装，完成BBB下xenomai用户环境相关内容的安装与配置，安装成功后可运行/usr/xenomai/bin/latency 进行测试。

四 实时主系统安装rtnet模块

运行在BBB板子上的rtnet需要依赖CPSW网卡驱动的实时改造，这一部分的代码并不由rtnet官方实现，因此需要在github上找第三方提供的代码。首先将SD卡插入电脑，在unbuntu系统下会自动加载，然后执行：

cd /home/用户名/Documents/bbb/yakbuild (即第一步骤创建的文件夹)

git clone git://github.com/hiddeate2m/rtnet.git rtnet_bbb

cd rtnet_bbb

./configure --host=arm-linux-gnueabihf
 --build=i686-linux-gnu （根据自己电脑确定 uname -m）
 --prefix=/media/用户名/rootfs/usr/bbbrtnet_bin
 --with-linux=/home/用户名/Documents /bbb/yakbuild/KERNEL
 --with-rtext-config=/media/用户名/rootfs/usr/xenomai/bin/xeno-config
 --enable-ticpsw
 
配置完成后开始编译：

	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- 
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- install

至此完成rtnet编译以及安装。将SD装至卡槽上电BBB板子，从SD卡启动。

通过SSH连接BBB板子，执行以下命令：
cd /usr/bbbrtnet_bin/modules/

insmod rtnet.ko

insmod rt_smsc.ko

insmod rt_davinci_mdio.ko

insmod rt_ticpsw.ko

insmod rtpacket.ko


至此完成rt_ticpsw网卡驱动模块的加载

../sbin/rtifconfig rteth0 up

此时网卡可用，完成整个流程。
