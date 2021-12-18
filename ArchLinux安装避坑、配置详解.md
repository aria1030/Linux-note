# ArchLinux安装避坑、配置详解

### 一、安装前准备

开机，按F2，进入引导选择，选择安装U盘的选项，回车。等一会，这时候可以按“上下左右”的“上下”，不然可能直接进入shell，那个是不能用来安装的。
选择第一个，回车。等加载，然后出现root@archis~ #，开始安装！

#### 1、验证启动模式
这一步用来验证机器能不能用UEFI启动。 

	ls /sys/firmware/efi/efivars   
	
出现一大堆，就可以，什么都没有，就是不行。

#### 2、连接到因特网（本次在无线网环境下安装，有线网需要装的工具不同）
	Ip link
用来确保系统启用了网络接口。

	dhcpcd
动态IP地址和DNS服务器分配，由systemd-networkd和systemd-resolved提供功能,对于有线以太网、无线局域网（WLAN）和无线广域网（WWAN）网络接口来说开箱即用

无线网络用iwd 

	iwctl
	device list
	station device scan
	station device get-networks
	station connect SSID（只用这一句连接也可以）
	
以上device 是wlan0，SSID是aria
连接上之后
	Station device show
	ping -c 3 baidu.com
检查网络连接，看看网络的延迟

#### 3、更新系统时间
	timedatectl set-ntp true
	timedatectl status 		#用以检查时间是否有误。

#### 4、建立分区
	fdisk -l    lsblk  
/dev/sda 是windows固态硬盘   
/dev/sdb是linux系统固态盘

/dev/sda1为256M的efi分区，后面把这个分区挂载到linuxefi分区中，linux就不需要重新分efi分区了   
/dev/sda2为剩下的Windows分区

linux需要/根目录/dev/sdb1，/home家目录/dev/sdb2和swap交换分区/dev/sdb3
##### 用cfdisk分区简单易懂

#### 5、格式化分区
	mkfs.ext4 /dev/sdb1
	mkfs.ext4 /dev/sdb2
	mkswap /dev/sdb3
	swapon /dev/sdb3

#### 6、挂载分区
挂载，就是把设备依附到目录树上，linux下一切皆文件，挂载之后，这个分区就成了一个文件夹了，注意挂载顺序。

首先挂载根分区/ 

	mount /dev/sdb1 /mnt

由于需要使用UEFI引导，所以需要创建其挂载点并挂载。

	mkdir -p /mnt/boot/efi
	mount /dev/sda1 /mnt/boot/efi

还创建了home目录，所以也要挂载

	mkdir /home
	mount /dev/sdb2 /mnt/home

### 二、安装

####1、选择镜像
默认镜像速度很慢，需要切换到中国的镜像源。

	reflector --country China --sort rate --latest 5 --save /etc/pacman.d/mirrorlist
	
或

	/vim /etc/pacman.d/mirrorlist
	
/tuna 回车 yy g 下移到空白处上的一个#，p
/ustc 回车(查找功能) yy g 下移到之前复制的一行，p
基本就够了，还可以用163阿里云，重复上述操作
Esc，:wq，回车

更新镜像
	pacman -Syy

#### 2、安装必须的软件包

这里用pacstrap脚本，有些软件可以在后边用pacman命令安装，我一般把能想到的全装了，网络工具必须在这一步安装好，要不以后不能联网安装了。

	pacstrap /mnt base base-devel linux linux-firmware man-db man-pages texinfo  vim iwd dhcpcd reflector 
	
等待，安装完。

#### 3、生成fstab

	genfstab -U /mnt >> /mnt/etc/fstab
	
把之前的挂载信息写入fstab文件，然后要检查一下这个文件，并修改。

	vim /mnt/etc/fstab
	
找到efi的那一行，把最后的2改成0(不用fdisk检测)，就是光标移到那，点i，删除，输入0就可以。esc，wq，回车。
记得挂载windows启动分区磁盘，要不os-prober探测不到

	UUID=”aaaa” none挂载点	文件系统格式 default挂载参数	0是否dump备份	0是否dsck检测

#### 4、进入新系统
	arch-chroot /mnt

#### 5、配置时区

	ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
	hwclock –systohc	#以生成 /etc/adjtime
	
稍等一会，用date命令看看时间是不是对的。

#### 6、本地化（本地化个鬼，不能在此配置中文，会导致 tty 乱码）

	vim /etc/locale.gen
	
/en_US.UTF-8 回车，取消注释，用x键删除就好了
/zh_CN 回车，取消UTF-8和GBK的注释
Esc，:wq，回车

	locale-gen
	vim /etc/locale.conf
	
i，输入：

	LANG=en_US.UTF-8
	
Esc，:wq，回车

#### 7、配置网络

	vim /etc/hostname
	
i 输入：
myhostname（随意写，这是你的计算机名aria）
保存退出

	vim /etc/hosts
	
i 输入：

	127.0.0.1 localhost
	::1  localhost
	127.0.1.1 lolcalhost.localdomain localhost
	
保存退出

然后安装网络管理工具(上面安装完了就不需要安装了)，启动服务

	pacman -S iwd networkmanager dhcpcd 	
	systemctl enable dhcpcd
	systemctl enable iwd networks
	
使网络管理服务开机自启，要不重启系统后需要enable并start这些服务

#### 8、 设置root密码、增加用户

	passwd    输入（不会显示出来）回车，再输入，回车，就好了。

##### 增加用户

	useradd -m -G wheel aria     # -m    创建家目录 -G    用户所属的组    
	
##### 设置 aria 用户密码

 	passwd arch
 	#修改(arch)用户权限
 	vim /etc/sudoers        # 编辑sudoer file
	
	去掉“%wheel ALL=(ALL) ALL”前面的注释，保存退出


#### 9、 安装引导
使用grub作为引导，还要安装能Windows塞进去的软件

	pacman -S grub efibootmgr os-prober 
	
其中GRUB是启动引导器，efibootmgr被GRUB脚本用来将启动项写入 NVRAM。

	grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=archlinux
	
想让grub-mkconfig探测其他已经安装的系统并自动把他们添加到启动菜单中，需安装软件包os-prober并挂载包含其它系统引导的磁盘分区。
编辑/etc/default/grub并取消下面这一行的注释，如果没有相应注释的话就在文件末尾添加上：

	GRUB_DISABLE_OS_PROBER=false
	
然后运行

	grub-mkconfig -o /boot/grub/grub.cfg
	
#### 注意： 
>1、记得每次运行grub-mkconfig之前都把包含其他操作系统的分区挂载上，以免忽略了这些操作系统的启动项。
>
>2、如果你得到以下输出：

	Warning: os-prober will not be executed to detect other bootable partitions
	
需要编辑/etc/default/grub并取消下面这一行的注释，如果没有相应注释的话就在文件末尾添加上：    

	GRUB_DISABLE_OS_PROBER=false	#2.6以后都需要添加
	
然后运行 grub-mkconfig再试一次
>3、os-prober通常能自动发现包含 Windows 的分区，当然在载入默认的 Linux驱动的情况下，NTFS分区也不是总能够被探测到。如果 GRUB 没能发现它，尝试安装NTFS-3G，然后重新挂载这个分区再试一次。
		
		pacman -S ntfs-3g 
>4、上述安装完成后 GRUB 的主目录将位于/boot/grub/grub-install,还将在固件启动管理器中创建一个条目，名叫archlinux。如果你的启动条目已满，这个命令会执行失败；你需要使用 efibootmgr 来删除不必要的条目。
>
>5、--efi-directory和--bootloader-id是 GRUB UEFI 特有的。
--efi-directory替代了已经废弃的--root-directory。

>6、在grub-install命令中没有一个 <device_path> 选项，例如/dev/sda。事实上即使提供了 <device_path>，也会被 GRUB 安装脚本忽略，因为 UEFI 启动加载器不使用 MBR 启动代码或启动扇区。
>
>7、确保grub-install命令是在你想要用 GRUB 引导的那个系统上运行的。
也就是说如果你是用安装介质启动进入了安装环境中，你需要在chroot之后再运行grub-install。
如果因为某些原因不得不在安装的系统之外运行grub-install，在后面加上--boot-directory= 选项来指定挂载/boot目录的路径，例如

	--boot-directory=/mnt/boot。

完工！

#### 10、卸载与重启
	exit	
	
退出chroot

	umount -R /mnt/home
	Umount -R /mnt/boot/efi
	umount -R /mnt
	reboot 

### 三、部署archlinux使用环境

安装好了arch重启后进入Windows，甚至没有arch的启动项。
因为WindowsBootManager是无法引导archlinux的，但是grub可以引导Windows，所以在开机得时候按F2选择arch启动。
进入arch，还是只有文字界面哦。

先不着急安装图形界面，有些东西可以先搞。

#### 1、先确定一下网络能不能用

	ping -c 3 baidu.com
	
没有的话

	Ip link
	dhcpcd
	ping -c 3 baidu.com
	
出现了就：

	systemctl enable NetworkManager
	systemctl enable dhcpcd
	systemctl start	NetworkManager dhcpcd

#### 2、没有AUR还能叫arch？
添加archlinuxcn源，这里有国内常用的软件，AUR的辅助工具也在这里，不然等编译，等到天荒地老。

	vim /etc/pacman.conf
	
g 去掉[multilib]部分的#
i写入：

	[archlinuxcn]
	Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
	
保存退出

	pacman -Syy
	pacman -S archlinuxcn-keyring
	pacman -S yay 		#这个就是AUR的辅助工具

#### 3、	在安装图形界面之前先打驱动

	pacman -S pulseaudio pulseaudio-alsa sudo latte-dock
	
下边只是以上软件的注释，不需要输入的。

•	pulseaudio pulseaudio-alsa 	 #声音相关
•	sudo				 #sudo命令
•	latte-dock 			 #KDE的dock栏插件
•	pulseaudio-bluetooth 		 #蓝牙耳机
•	xf86-input-libinput		 #触摸板

#启动蓝牙服务：

	#systemctl enable bluetooth

#### 4、 装桌面
这里要装的东西不多，就两

 	pacman -S plasma kde-applications

然后回车回车，两个只是大包名，大包里边有小包，等待

然后安装桌面登录器：（lightdm可选）

	pacman -S sddm
	systemctl enable sddm

#### 5、创建普通用户
root账户是不能登录sddm的，需要创建一个普通账户：

	useradd -m -g users -G wheel -s /bin/bash 用户名
	
上边的用户名自己填，然后设置密码

	passwd 用户名
然后

	visudo
	
去掉%wheel ALL=(ALL)ALL的#，普通用户可使用sudo命令获取root权限

#### 6、进入桌面继续开搞

	exit
	
用户名（普通用户的用户名），回车，密码，回车

	sddm
	
不出意外会出现简陋的sddm界面，进去了，如果进了桌面，就继续，没有，就重启，用普通用户登录。
进去之后，win键，applications，system，konsole，打开终端。
如果是Intel+NVIDIA双显卡，

	sudo pacman -S xf86-video-intel nvidia bbswitch optimus-manager-qt-kde
	
装好去设置成开机自启动，搜一下就有了。

7、省电优化

	sudo pacman -S tlp tlp-rdw smartmontools x86_energy_perf_policy ethtool
	sudo systemctl enable tlp
	sudo systemctl enable NetworkManager-dispatcher
	sudo systemctl mask systemd-rfkill.service
	sudo systemctl mask systemd-rfkill.socket

#### 8、中文化
上边都是英文操作，也可以先弄成中文，然后没差，win键，第二个，system config（大概是叫这个吧），往下拉，找到一个region什么的，点进去，add，找到中文添加，再置顶，应用重启一下就好了。（其实重新登录就好，我知道怎么重新登录，不保证其他人知道）

##### 中文字体

	sudo pacman -S ttf-dejavu wqy-microhei wqy-zenhei

wiki就说的字体较少不够，就这几个字体会让你很难受。
所以复制Windows的字体，你得先确定能够进Windows的磁盘，文件管理的图标是很好找的，进去找到那个磁盘，点了，会让你输密码那就是能进。不行，

	sudo pacman -S ntfs-3g
	
重启吧，反正按wiki配置这个，老报错，我没搞懂，但是重启了就能进。
魔法一步

	sudo cp /run/media/eralm/02306FF8306FF159/Windows/Fonts（这里请自行去文件夹里复制） /usr/share/fonts/
	
#如果不喜欢命令行，在sddm的设置里换一个样子之后，再退出，用root账户登录，不然没有权限，复制不了

	sudo  fc-cache -vf
	
Windows的字体就被安装到linux里了。

##### 中文输入法

你要按着wiki走，能出一个好的输入法算我输。我在输入法上花了最多的时间，来这么一个重新规划的诱因，也是之前配置输入法，把系统的语言环境搞崩了。
先来康康wiki咋说的，安装fcitx，你要不装搜狗，这一步一点问题都没有，sunpinyin、libpinyin、rime、googlepinyin都能用，然后是配置，在~/.pam_environment里添加

	GTK_IM_MODULE=fcitx
	QT_IM_MODULE=fcitx
	XMODIFIERS=@im=fcitx
	
我明白地告诉你，这样没用，在wps和浏览器里都没法用输入法打字。
正确的做法（不装搜狗）：
	
	sudo pacman -S fcitx kcm-fcitx fcitx-**pinyin 可选（fcitx-im）
	sudo vim .xprofile
	
先复制如下内容：

	export LC_ALL=zh_CN.UTF-8
	export LANGUAGE=zh_CN:en_US
	export GTK_IM_MODULE=fcitx
	export QT_IM_MODULE=fcitx
	export XMODIFIERS=@im=fcitx
	
按ctrl+shift+v，点删除不能识别字符，保存退出。重启。
如果装搜狗输入法：

	sudo pacman -S fcitx fcitx-sogouimebs kcm-fcitx
	sudo vim .xprofile
先复制如下内容：
	
	export LC_ALL=zh_CN.UTF-8
	export LANGUAGE=zh_CN:en_US
	export GTK_IM_MODULE=fcitx
	export QT_IM_MODULE=fcitx
	export XMODIFIERS=@im=fcitx
	
按ctrl+shift+v，点删除不能识别字符，保存退出。重启。

#### 9、美化及软件安装
美化去设置里找，慢慢搞，这个很简单。
软件，首先要知道几个pacman命令：

•	查找软件pacman -Ss
•	安装pacman -S
•	卸载pacman -R
•	更新源pacman -Sy
•	更新系统pacman -Syu
•	全面更新系统pacman -Syyu
•	移除无用包pacman -Sc
•	查找已安装的包pacman -Q
•	删除软件及依赖pacman -Rsn

好，可以开始安装了。以下均为S后填的内容，建议将sudo pacman改为yay，例：yay -S firefox。

浏览器 
	firefox或google-chrome
文字编辑 
	wps-office-cn wps-office-mui-zh-cn ttf-wps-fonts（三个都要，不然只有英文版的wps，默认是组件分开显示的，要去wps里设置成整合模式）
音乐 
	netease-cloud-music
视频 
	VLC（自带）
微信 
	先yay装deepin-wine，然后装wechat，在前面都弄好了应该不会蹦出英文版的微信出来
QQ 
	deepin-qq-eim（企业版QQ，我没成功打开，怪怪用手机QQ吧） qq-linux（这个08画风，还得扫码登录的假QQ）
python
	pycharm pycharm-community-jre
最后，部署完工！

