## 前言
导师让我跑一个[RoboMimic_Deploy](https://github.com/ccrpRepo/RoboMimic_Deploy/tree/master) 项目，其中代码用了pygame来调用手柄。手头正好有PS5的手柄，但是用WSL2默认不支持USB以及没有驱动，网上有很多WSL连接xbox手柄的教程但ps5手柄却没有，虽然没什么区别，但中间还是费了点功夫，分享一下需要注意的点。
# 步骤
WSL中的Ubuntu不能连接手柄有2个原因：
1. [WSL不提供本机连接USB设备的支持](#USB连接)
2. [WSL的Ubuntu内核没有编译手柄的驱动](#重新编译内核添加手柄驱动)

因此需要一步步来

# USB连接
参考：[微软官方文档-连接 USB 设备](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)

1.  win中下载安装[usbipd-win](https://github.com/dorssel/usbipd-win/releases)
2.  使用**管理员模式**打开powershell，查看USB设备
	```powershell
	usbipd list
	```
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e02939f6f08c4e79a312d6f87f51986a.png)
3. 找到手柄的BUSID，用`bind`进行共享
	```powershell
	usbipd bind --busid <busid>
	```
	这里可能会提醒需要重启，
4. 用`attach`将手柄添加到WSL中
	```powershell
	usbipd attach --wsl --busid <busid>
	```
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e6728c0a0d7a42a1a24e00a8fcc2b2a6.png)
如果提示正在使用，可以在`usbipd bind`加上`--force`强制共享

5. 在Ubuntu中使用`lsusb`查看当前USB设备
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f1ddf8b3cc3048a0b65bec029a4e4fad.png)
这样就成功把手柄添加到WSL中了，但是此时还没有驱动。

# 重新编译内核，添加手柄驱动
其实5.12版本之后的内核源代码中都带有PS5手柄的驱动代码，但是WSL里安装的内核都没有编译，因此需要重新编译个内核。
在翻遍全网后，终于找到了驱动的方法：[Gentoo Wiki - Sony DualShock](https://wiki.gentoo.org/wiki/Sony_DualShock#Kernel)，但还是有不少坑。

1. 下载微软官方WSL内核源码：[WSL2-Linux-Kernel](https://github.com/microsoft/WSL2-Linux-Kernel/releases)，搬到Ubuntu中并解压
2. 安装编译工具
	
	```bash
	sudo apt install build-essential flex bison dwarves libssl-dev libelf-dev flex bison bc ncurses 	
	```
3. **进入文件夹，使用图形化界面 menuconfig 编辑配置**
	```bash
	sudo make menuconfig KCONFIG_CONFIG=Microsoft/config-wsl
	```
	在menuconfig中，将以下选项按 y 选中：
	```
	Networking support --->
	   <M>   Bluetooth subsystem support --->
	      [*]   Bluetooth Classic (BR/EDR) features
	         <*>   HIDP protocol support
	Device Drivers --->
	   Input device support --->
	      -*- Generic input layer (needed for keyboard, mouse, ...)
	      <*>   Joystick interface
	      <*>   Event interface
	      [*]   Miscellaneous devices  --->
	         <*>   User level driver support
	   HID bus support --->
	      [*]   Battery level reporting for HID devices
	      [*]   /dev/hidraw raw HID device support
	      <*>   User-space I/O driver support for HID subsystem   
	      <*>   Generic HID driver
	            Special HID drivers --->
	              <*> Sony PS2/3/4 accessories
	              [*]   Sony PS2/3/4 accessories force feedback support
	              <*> PlayStation HID Driver
	              [*]   PlayStation force feedback support
	      USB HID support --->
	         <*> USB HID transport layer
	   [*] LED Support --->
	      <*>   LED Class Support
	      <*>   LED Multicolor Class Support
	
	
	   [*] USB support  ---> 
	      <*>   Support for Host-side USB
	
	      <*>   USB Mass Storage support                                                 
	      [*]     USB Mass Storage verbose debug                                         
	      <*>     Realtek Card Reader support                                            
	      [*]       Realtek Card Reader autosuspend support                              
	      <*>     Datafab Compact Flash Reader support                                   
	      <*>     Freecom USB/ATAPI Bridge support                                       
	      <*>     ISD-200 USB/ATA Bridge support                                         
	      <*>     USBAT/USBAT02-based storage support                                    
	      <*>     SanDisk SDDR-09 (and other SmartMedia, including DPCM) support         
	      <*>     SanDisk SDDR-55 SmartMedia support                                     
	      <*>     Lexar Jumpshot Compact Flash Reader                                    
	      <*>     Olympus MAUSB-10/Fuji DPC-R1 support                                   
	      < >     Support OneTouch Button on Maxtor Hard Drives                          
	      <*>     Support for Rio Karma music player                                     
	      <*>     SAT emulation on Cypress USB/ATA Bridge with ATACB                     
	      <*>     USB ENE card reader support                                            
	      <*>     USB Attached SCSI       
	
	      <*>   USB/IP support
	      <*>     VHCI hcd         
	```
	其中*USB support*部分是我后来根据这个[issue](https://github.com/dorssel/usbipd-win/issues/1010)加的，否则`usbipd attach`会报错，其他都来自[Gentoo Wiki - Sony DualShock](https://wiki.gentoo.org/wiki/Sony_DualShock#Kernel)
	
	## **注意：最恶心的地方来了，请一定要按照一下顺序来选中！！！**
		1. Networking support
		2. Input device support
		3. USB support
		4. LED Support
		5. HID bus support ---USB HID support
		6. HID bus support 其他选项和 Special HID drivers
		  （这个一定要保证上面的都选中了，否则可能没有 PlayStation HID Driver选项或者只能选择module）
		
	我知道你肯定嫌麻烦，所以我把编译好的内核传到[github](https://github.com/tq3940/WSL-Ubuntu-DualSense)了（内核版本6.6.87.2），欢迎来玩呀━(*｀∀´*)ノ亻!
	
	想手动配置可以参考下图（~~多图警告~~ ）：
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/84fa7853e1944218b9cc0beaa1ea47ed.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/352b328099124758ba9a11d3fe149e95.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/b80ae1efb91846d4b54c9c0da99bf832.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/cf7c0a207a4e4f3b8d8b2d03186307c6.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/a70923c74cf946219e553ed78f1027a4.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/6fffe6863388432896643f5e61cd2e82.jpeg)
	![请添加图片描述](https://i-blog.csdnimg.cn/direct/efc8e3e2e87f4cd594c33ba89f52b31f.jpeg)
	
	如果有找不到的或者无法选中只能选module的，可以输入`/`进入搜索页面，搜索位置以及选中所需的依赖。



4.  编译内核

	```bash
	sudo make -j$(nproc) bzImage KCONFIG_CONFIG=Microsoft/config-wsl
	```
	编译完的内核位置在`源代码文件夹/arch/x86/boot/bzImage`
	
5. 更换内核
	这里有两种方法：
	1. **直接替换**WSL使用的内核文件，地址：`C:\WINDOWS\System32\lxss\tools\kernel`，可以参考[这篇文章](https://blog.csdn.net/weixin_43408232/article/details/129960452).
		
		但是由于我把我的Ubuntu搬到了D盘，这个目录下没东西了，用Listary都搜不到kernel在哪
		
	2. **修改`.wslconfig`配置文件**
	
		先将`bzImage`复制到windows中
		```bash
		cp arch/x86/boot/bzImage /mnt/d/WSL/Ubuntu-20.04/
		```
		再在windows的个人目录下新建个`.wslconfig`文件，写入：
		```
		[wsl2]
		kernel=D:\\WSL\\Ubuntu-20.04\\bzImage
		```
		即可指定WSL使用的内核，关闭Ubuntu，再关闭WSL
		```powershell
		wsl --shutdown
		```
		**等待8秒后**再启动Ubuntu
				
		> [配置更改的 8 秒规则](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config#the-8-second-rule-for-configuration-changes)
		必须等到运行你的 Linux 发行版的子系统完全停止运行并重启，配置设置更新才会显示。 这通常需要关闭发行版 shell 的所有实例后大约 8 秒。


6. 测试

	查看内核信息
	```bash
	 uname -a
	```
	
	此时可能需要重新共享一遍USB设备
	```powershell
	usbipd detach --busid <bushid>	# 断开连接
	```
	

	重新连接后就能在`/dev/input/`下看到`js0`设备了
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/184c168d68d443cd805b6ff3d5190c38.png)
测试手柄
	
	```bash
	sudo apt-get install joystick -y
	sudo jstest /dev/input/js0
	```
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1ecf01e33f6a49d595b36ef4773d63dc.png)
收工撒花 ✿✿ヽ(°▽°)ノ✿

参考文章：
[WSL2下连接XBOX手柄详细教程](https://blog.csdn.net/hl1526885155/article/details/131376854)

[WSL2编译内核并更改替换内核版本](https://blog.csdn.net/weixin_43408232/article/details/129960452)

[微软官方文档-连接 USB 设备](https://learn.microsoft.com/zh-cn/windows/wsl/connect-usb)

[Gentoo Wiki - Sony DualShock](https://wiki.gentoo.org/wiki/Sony_DualShock#Kernel)
