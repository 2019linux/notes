# Tiny210笔记

[TOC]

------



## 一.开发环境

### 	1.开发板/主机电脑/虚拟机/ 三者处于相同的网段。

​	虚拟机修改方法:

![1563777359135](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1563777359135.png)





![1563777376146](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1563777376146.png)



​	选择本地网络的网卡。

### 		2.设置开发板网络

```c
	gatewayip=192.168.10.3			
    ipaddr=192.168.10.119
	netmask=255.255.255.0
	serverip=192.168.10.110
```
​			gatewayip  :网关

​			ipaddr ：本地IP地址

​			netmask：掩码

​			serverip： 服务器的IP地址

​		设置的IP地址，以本地网络的物理地址为准，上面选择的网卡也需要是本地网路的网卡，需要和WIFI的网络的网卡区分开。



## 二：文件系统

### 		1.nfs网络文件系统

#### 				1.安装nfs服务器和客户端

​					sudo apt-get install nfs-kernel-server nfs-common portmap

#### 				2.配置portmap 两种方式任选一种

​						（1）sudo vim /etc/default/portmap 去掉 -i 127.0.0.1

​						（2）sudo dpkg-reconfigure portmap 选择"否"

#### 				3.配置挂载目录和权限

​							vim /etc/exports

​							在最后添加 /nfsroot 	*(rw,sync) #nfsroot为nfs服务器根目录

​							/workspace/ftpboot/mini_rootfs  *(sync,rw,no_root_squash)

#### 				4.更新exports文件

​						sudo exportfs -r

#### 				5.重启nfs服务

​							sudo /etc/init.d/portmap start

​							sudo /etc/init.d/nfs-kernel-server

#### 				6.开发板UBOOT命令设置：

​					1.1 setenv bootargs 'root=/dev/nfs nfsroot=192.168.1.102:/workspace/ftpboot/mini_rootfs ip=192.168.1.3 init=linuxrc console=ttySAC0,115200'

​					解释：192.168.1.102  虚拟的IP地址，网络文件系统放在虚拟机上。

​					/workspace/ftpboot/mini_rootfs 网络文件系统的路径

​					192.168.1.3   开发版的IP地址，查看开发板IP地址printenv, 设置，setenv命令

​						

​					1.2 setenv bootcmd 'tftpboot 20008000 uImage;bootm 20008000'

​					解释：加载kernel



### 		2.安装TFTP服务器

​				 apt-get install tftp-hpa    //客户端软件

​				 apt-get install tftpd-hpa  //是服务程序

​				 apt-get install xinetd  //是新一代进程守护程序



​				vim /etc/xinetd.d/tftp，看 server_args的地址是否正确

			service tftp{ 
	            disable = no
	            socket_type = dgram
	            protocol = udp
	            wait = yes
	            user = root
	            server = /usr/bin/in.tftpd
	            server_args = -s /workspace/ftpboot -c
	            per_source = 11
	            cps = 100 2
	            flags = IPv4
	        }
​			最后重启xinetd服务。输入命令：“sudo /etc/init.d/xinetd restart”.到这里，TFTP服务器就搭建好了。



### 		3.检查防火墙是否关闭

​			tftp服务是否打开，/etc/init.d/iptables stop



​			

## 三：内核的编译加载

### 1.编译

​	拷贝arch/arm/congfig下的defincong 到 顶级目录下.config

​	make ARCH=arm menucofnig配置

​	make ARCH=arm CROSS_COMPILE= xxx 

​				xxx交叉编译工具：opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-

### 		2.zImage制作成uImage

​		./mkimage -A arm -O linux -T kernel -C none -a 30008000 -e 30008000 -n linux-3.0.8 -d zImage uImage

​		mkimage 目录在 uboot的tools 目录下面

​				直接编译uImage:

​				make uImage LOADADDR=0X40008000

### 	   3.查看启动文件的依赖项

​		readelf -d linuxrc  | grep NEEDED

### 4.uboot烧写

​	sudo dd iflag=dsync oflag=dsync if=tiny210-uboot1.bin of=/dev/sdb seek=1

​	将uboot写入SD第一扇区。 uboot的启动从第一扇区开始。



## 四：调试过程遇到的问题

### 	1.Uncompressing Linux... done, booting the kernel.

​			修改机器码和UBOOT不对应：

​			linux-3.0.8/arch/arm/tools/mach-types

​				smdkv210        MACH_SMDKV210        SMDKV210        2456

​				2456应该同Uboot中的一致，应改为3466

### 	2.tftp超时

​				修改uboot源码中

​			在u-boot for tiny210 源码

​			1.net/tftp.c18：

​			#define TIMEOUT         50000UL 

​			2.在net/net.c中

​			#define ARP_TIMEOUT         50000UL 

### 	3.tiny210 uboot下载地址

​			[git@github.com:2019linux/tiny210-u-boot-version4.0.git]()

### 	4.kernel下载地址

​	       [git@github.com:2019linux/linux3.0.8-youshanzhibi.git]()

### 5.交叉编译工具

​			[git@github.com:2019linux/tools.git]()



​	

