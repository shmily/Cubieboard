Cubieboard arm-linux 移植
=========================

1、u-boot 移植
--------------

获取Cubieboard的u-boot源代码

	git clone https://github.com/linux-sunxi/u-boot-sunxi.git --depth=1

编译u-boot，得到 u-boot 、spl/sunxi-spl.bin  
在 board/allwinner/ 目录下有该芯片对应的不同配置，我们使用的是cubieboard

	make cubieboard CROSS_COMPILE=arm-linux-gnueabihf-


2、script.bin 编译
------------------

这个文件对A10芯片进行配置的二进制文件，好像是内核启动后读取的，但是不清楚具体实现是怎么做到的。

下载sunxi-tools源码，编译

	git clone https://github.com/linux-sunxi/sunxi-tools.git --depth=1
	cd sunxi-tools
	make

得到 fex2bin 文件，这个是能把 *.fex 文件生成 *.bin文件。

下载sunxi-boards源码，得到cubieboard对于的fex文件

	git clone https://github.com/linux-sunxi/sunxi-boards.git --depth=1
	cd sunxi-boards

在 sys_config/a10 目录下，我们能找到 cubieboard_1gb.fex 文件，这就是我们需要的   
编译，得到 script.bin

	fex2bin cubieboard_1gb.fex script.bin

3、制作可启动的SD卡
-------------------

参考自:   
	[wiki]:http://linux-sunxi.org/Bootable_SD_card   
	[wiki]:http://linux-sunxi.org/FirstSteps

allwinner A10 芯片上电启动的时候，会读取SD卡最前面的 1M 内容，从得到 bootloader，   
所以我们需要把 u-boot 、spl/sunxi-spl.bin 写到SD卡的前1M区间。

假设在host上，我们的sd卡是 /dev/sdb

*	对SD卡进行分区处理

		dd if=/dev/zero of=/dev/sdb bs=1M count=1       # 把SD卡前1M的区域填充为0，预留给 u-boot 和 spl/sunxi-spl.bin
		
		sfdisk -R /dev/sdb								# 重新读取/dev/sdb，因为我们已经改变了sdb
		
		cat <<EOT | sfdisk -uM /dev/sdb 				# 对sd卡进行分区，具体请参照sfdisk命令手册
		1,16,c 											# 从SD卡的1M处开始，划分16M长度为第一个分区
		18,,L 											# 余下的空间为第二个分区 (参考网站上使用的是 17,,L，这是错误的，重叠了)
		EOT
		
		mkfs.vfat /dev/sdb1								# 格式化为fat
		mkfs.ext4 /dev/sdb2 							# 格式化为ext4

*	写入 bootloader
	
		dd if=spl/sunxi-spl.bin of=/dev/sdb bs=1024 seek=8
		dd if=u-boot.bin of=/dev/sdb bs=1024 seek=32

*	安装内核 uImage，设置启动参数

	*安装 uImage*  
		
		mount /dev/sdb1 /mnt
		mkdir /mnt/boot
		cp linux-sunxi/arch/arm/boot/uImage /mnt/boot
		cp script.bin /mnt/boot

	----------------------------------------------

	*设置启动参数*
	*新建 boot.cmd 文件，文件内容如下*

		setenv bootargs console=ttyS0,115200 noinitrd init=/init root=/dev/mmcblk0p2 rootfstype=ext4 rootwait panic=10 ${extra}
		fatload mmc 0 0x43000000 boot/script.bin
		fatload mmc 0 0x48000000 boot/uImage
		bootm 0x48000000

	----------------------------------------------

	*由 boot.cmd 生成 u-boot能读取的 boot.scr 文件*

		mkimage -C none -A arm -T script -d boot.cmd boot.scr

	----------------------------------------------

	*把 boot.scr 拷贝到sd卡的第一个分区*

		cp boot.scr /mnt/								# 之前我们已经把 /dev/sdb1 挂载在 /mnt 上

*	关于 uEnv.txt
	
	老版本的 u-boot 是可以直接读取boot.scr，但是新版本的貌似不支持，但是可以通过 uEnv.txt 间接读取boot.scr，  
	当然，也可以直接使用 uEnv.txt 编写启动参数。

	*我使用的 uEnv.txt*
	
		bootenv=boot.scr
		loaduimage=fatload mmc ${mmcdev} ${loadaddr} ${bootenv}
		mmcboot=echo Running boot.scr script from mmc ...; source ${loadaddr}


4、内核编译
-----------

	git clone https://github.com/linux-sunxi/linux-sunxi.git

	make ARCH=arm sun4i_defconfig
	make ARCH=arm menuconfig
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 uImage
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 INSTALL_MOD_PATH=output modules
	make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j4 INSTALL_MOD_PATH=output modules_install


5、根文件系统
-------------
	
我使用的是 buildroot-2012.08 建立的文件系统，只需要生成 rootfs 就行了，kernel 与 toolschain 使用外置的，  
不需要使用 buildroot 生成。
