# BeagleboneXenomai2Rtnet
1.	下载内核源代码

cd /home/用户名/Documents/bbb(将用户名替换为实际路径，bbb名字可更改，以下路径以此为准)

git clone https://github.com/RobertCNelson/yakbuild

cd ./yakbuild/

2.	配置所需要下载的内核版本

cp recipe.sh.v3.8.x.sample recipe.sh

将所得到的 recipe.sh文件内的kernel_tag设置为kernel_tag="3.8.13-xenomai-r84" 

3.	./build_deb.sh下载内核代码后开始编译，首先会进入内核编译选项配置界面，按照如下选项进行设置：

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

4.	开始编译内核，时间大概需要20分钟，在/deploy文件夹下面可以找到如下几个deb包：

linux-firmware-image-3.8.13-xenomai-r83_1cross_armhf.deb

linux-headers-3.8.13-xenomai-r83_1cross_armhf.deb

linux-image-3.8.13-xenomai-r83_1cross_armhf.deb

linux-libc-dev_1cross_armhf.deb

5.	准备一张SD卡，将官网下载的debian系统镜像烧录至sd卡，然后从sd卡启动BBB板子（按住S2按钮后不放上电直至4盏LED灯全亮），通过SSH连接BBB板子，初始用户名密码一般为（debian:temppwd）或者（machinekit:machinekit）。然后将上一步骤得到的几个deb包通过SSH上传至BBB板子内，假设放在/home路径下，按照如下步骤进行安装:

cd /home

sudo dpkg -i *.deb

安装完之后reboot,重新连接BBB板子，至此完成BBB板子的实时内核更换，输入uname -r 可以看到3.8.13-xenomai-r83， 输入cat /proc/xenomai/version 可以看到2.6.3

6.	下载xenomai-2.6.3.tar.bz2，在BBB板子上进行安装，完成BBB下xenomai用户环境相关内容的安装与配置，安装成功后可运行/usr/xenomai/bin/latency 进行测试。

7.	从这一步骤开始安装rtnet模块。运行在BBB板子上的rtnet需要依赖CPSW网卡驱动的实时改造，这一部分的代码并不由rtnet官方实现，因此需要在github上找第三方提供的代码。首先将SD卡插入电脑，在unbuntu系统下会自动加载，然后执行：

cd /home/用户名/Documents/bbb/yakbuild (即第一步骤创建的文件夹)

git clone git://github.com/hiddeate2m/rtnet.git rtnet_bbb

cd rtnet_bbb

./configure --host=arm-linux-gnueabihf
 --build=i686-linux-gnu （根据自己电脑确定 uname -m）
 --prefix=/media/用户名/rootfs/usr/bbbrtnet_bin
 --with-linux=/home/用户名/Documents /bbb/yakbuild/KERNEL
 --with-rtext-config=/media/用户名/rootfs/usr/xenomai/bin/xeno-config
 --enable-ticpsw
 
8.	配置完成后开始编译：

	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- 
sudo make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- install

至此完成rtnet编译以及安装。将SD装至卡槽上电BBB板子，从SD卡启动。

9.	通过SSH连接BBB板子，执行以下命令：

cd /usr/bbbrtnet_bin/modules/

insmod rtnet.ko

insmod rt_smsc.ko

insmod rt_davinci_mdio.ko

insmod rt_ticpsw.ko

insmod rtpacket.ko

insmod rtipv4.ko

insmod rtudp.ko 

至此完成rt_ticpsw网卡驱动模块的加载

../sbin/rtifconfig eteth0 up

此时网卡可用，完成整个流程。




P.S. Github上有面向xenomai 3版本的cpsw网卡驱动代码，但是我浪费了一周时间测试了很多内核，都无法正常使用，因此放弃。
