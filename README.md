# orangepi-action-build

    利用github的action来编译香橙派
    编译操作系统和生成镜像文件是非常消耗时间的,而且自己个人的机器上面往往要配置开发环境也比较麻烦.
    github的action可以指定不同的操作系统来运行,这样对编译环境的依赖就降低了.
    还能节省自己的固态硬盘的使用寿命
    缺点是action运行速度还是挺慢的,而且中间数据也不会保存,编译等待的时间很长.

## doc 目录

doc里面会存放一些各种开发板的相关资料

## OrangePi Linux SDK编译参数

下面是我编译成功之后(Repeat Build Options ),最后日志记录的编译命令.
这个命令是可以完全在cli下面无阻塞的编译整个镜像文件的.不需要再选择编译参数.
编译包含了桌面系统,有时候需要用到桌面又需要自己精简一些安装包的时候就需要这样的编译参数了.
这样编译出来的镜像文件不到4G,比完整的桌面要节省1-2G的空间,在资源紧张的情况下有点用.
资源紧张一般是因为穷,很多时候充分的利用资源是为了提高生产效率.

```text
sudo ./build.sh  BOARD=orangepi3-lts BRANCH=next BUILD_OPT=image \
RELEASE=jammy BUILD_MINIMAL=no BUILD_DESKTOP=yes KERNEL_CONFIGURE=no \
DESKTOP_ENVIRONMENT=xfce DESKTOP_ENVIRONMENT_CONFIG_NAME=config_base \
DESKTOP_APPGROUPS_SELECTED="desktop_tools remote_desktop" \
COMPRESS_OUTPUTIMAGE=sha,gpg,img  ext
```

## 关于orangepi-build编译脚本的一些说明

### 镜像文件的尺寸
```
#file:configuration.sh
# Let's set default data if not defined in board configuration above
[[ -z $OFFSET ]] && OFFSET=4 # offset to 1st partition (we use 4MiB boundaries by default)

#file:debootstrap.sh
local imagesize=$(( $rootfs_size + $OFFSET + $BOOTSIZE )) # MiB
# stage: calculate boot partition size
	local bootstart=$(($OFFSET * 2048))
	local rootstart=$(($bootstart + ($BOOTSIZE * 2048)))
	local bootend=$(($rootstart - 1))

    parted -s ${SDCARD}.raw -- mkpart primary ${parttype[$ROOTFS_TYPE]} ${rootstart}s 100%
```

### 镜像中分区的UUID

echo "UUID=$(blkid -s UUID -o value ${LOOP}p${bootpart}) /boot ${mkfs[$bootfs]} defaults${mountopts[$bootfs]} 0 2" >> $SDCARD/etc/fstab

### 镜像文件的分区信息

```
# prepare_partitions
#
# creates image file, partitions and fs
# and mounts it to local dir
# FS-dependent stuff (boot and root fs partition types) happens here
#
prepare_partitions()

```
这里只是初始化镜像的分区

```
# create_image
#
# finishes creation of image from cached rootfs
#

```
这函数是实际的往镜像文件中写入数据-

```
# stage: write u-boot
	write_uboot $LOOP
```
将uboot的bin写入镜像

## 编译镜像时 初始化用户密码和账户
在 orange-build\scripts\configuration.sh中,把几个变量在编译前设置成你自己想要的账户和密码了.
```
[[ -z $ROOTPWD ]] && ROOTPWD="orangepi" # Must be changed @first login
[[ -z $OPI_USERNAME ]] && OPI_USERNAME="orangepi" 
[[ -z $OPI_PWD ]] && OPI_PWD="orangepi" 

[[ -z $MAINTAINER ]] && MAINTAINER="Orange Pi" # deb signature
[[ -z $MAINTAINERMAIL ]] && MAINTAINERMAIL="leeboby@aliyun.com" # deb signature
```
也可以在你自己的配置脚本orange-build\userpatches\config-default.conf中设置你要的参数
在文件的后面加上你自己的配置信息,这个会覆盖上面的配置信息,以你的配置来构建
将值改成你自己的然后再编译就OK
```
#users info init
#root password
ROOTPWD="123456"
#username
OPI_USERNAME="yourname"
#password
OPI_PWD="123456"
#company
MAINTAINER="your company name"
#mail
MAINTAINERMAIL="youemail@you.com"
```



## bootlogo 开机和关机动画
如果是定制产品,开机动画是很有必要的,否则开机弹出控制台界面,一行行的代码闪过去,对于最终用户的体验会比较糟糕.
linux内核的开机动画需要uboot和内核共同配置才能完美呈现.
uboot启动之后会加载一个静态画面,内核被uboot加载启动之后,要接力在屏幕绘制动画.
将 /boot/orangepiEnv.txt 文件中的 bootlogo=false 修改成 bootlogo=true.重启系统你就可以看到开机动画了.
armbian中的开机动画设置也是类似的,只不过配置文件是 /boot/armbianEnv.txt
而开机动画文件位置在uboot的启动脚本 boot.cmd 里面指定

```
if test "${bootlogo}" = "true"; then setenv consoleargs "bootsplash.bootfile=bootsplash.orangepi ${consoleargs}"; fi
```
注意,你直接在系统中修改boot.cmd并不会使得修改生效,需要用构建工具构建好才能使用.参考下面的脚本来构建.
```
	# recompile .cmd to .scr if boot.cmd exists
	[[ -f $SDCARD/boot/boot.cmd ]] && \
		mkimage -C none -A arm -T script -d $SDCARD/boot/boot.cmd $SDCARD/boot/boot.scr > /dev/null 2>&1
```
下面的脚本函数boot_logo是用来构建开机动画文件 bootsplash.orangepi
```
# 位置:orange-build\scripts\general.sh 可以参照这个脚本来订制你自己的开机动画
#--------------------------------------------------------------------------------------------------------------------------------
# Create kernel boot logo from packages/blobs/splash/logo.png and packages/blobs/splash/spinner.gif (animated)
# and place to the file /lib/firmware/bootsplash
#--------------------------------------------------------------------------------------------------------------------------------
function boot_logo ()

```
