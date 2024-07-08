# ubuntu 系统分区方案

| 目录  | 建议大小     | 格式 | 描述                                                         |
| :---- | :----------- | :--- | :----------------------------------------------------------- |
| /     | 150G-200G    | ext4 | 根目录                                                       |
| swap  | 物理内存两倍 | swap | 交换空间：交换分区相当于Windows中的“虚拟内存”，如果内存低的话（1-4G），物理内存的两倍，高点的话（8-16G）要么等于物理内存，要么物理内存+2g左右， |
| /boot | 1G左右       | ext4 | **空间起始位置** 分区格式为ext4 **/boot** **建议：应该大于400MB或1GB** Linux的内核及引导系统程序所需要的文件，比如 vmlinuz initrd.img文件都位于这个目录中。在一般情况下，GRUB或LILO系统引导管理器也位于这个目录；启动撞在文件存放位置，如kernels，initrd，grub。 |
| /tmp  | 5G左右       | ext4 | 系统的临时文件，一般系统重启不会被保存。（建立服务器需要？） |
| /home | 尽量大些     | ext4 | 用户工作目录；个人配置文件，如个人环境变量等；所有账号分配一个工作目录。 |
| /usr  | 100G         | ext4 | 安装软件的文件夹，安装时会占用较大硬盘容量的目录，可分配相对大一些的空间 |
| /var  | 20G+         | ext4 | 系统运行过程中渐渐占用硬盘容量的目录。包括缓存cache，日志log，以及某些软件运行所产生的文件，包括[程序文件](https://www.zhihu.com/search?q=程序文件&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A2726241058})（lock file, run file） |

**Linux 各目录含义**

/boot/： 启动文件，所有与系统启动有关的文件都保存在这里

/boot/grub/：grub引导器相关的配置文件都在这里

/dev/：此目录中保存了所有设备文件，例如，使用的分区：/dev/hda，/dev/cdrom 等。

/proc/：内核与进程镜像

/mnt/：此目录主要是作为挂载点使用

/media/： 挂载媒体设备 包括软盘，光盘，DVD等设备文件

/root/ root用户的HOME目录

/home/user名 /：普通用户的HOME目录，创建一个一般用户账号时，默认的用户主文件夹就在该目录下

/bin/：此目录中放置了所有用户能够执行的命令

/sbin/：此目录中放置了一般是只有root用户才能执行的命令

/lib/： 系统程序库文件目录

/etc/：系统程序和大部分应用程序的全局配置文件都在这个目录

/etc/init.d/： SystemV风格的启动脚本

/etc/rcX.d/：启动脚本的链接，定义运行级别

/etc/network/： 网络配置文件

/etc/X11/： 图形界面配置文件

/lost+found：包含了系统修复时的恢复文件

/proc：这个目录本身是一个虚拟文件系统。它放置的数据都是在内存当中，例如系统内核，进程等

/sys：一个虚拟的文件系统，主要也是记录与内核相关的信息。这个目录同样不占硬盘容量

/usr：usr并不是user的缩写，而是Unix Software Resource的缩写，即“Unix 操作系统软件资源”放在该目录，而不是用户的数据。这个目录
相当于Windows操作系统的“C:\Windows\”和“C:\Program files\”这两个目录的综合体，系统安装完毕后，这个目录会占用最多的硬盘容量

/usr/bin ：用户可使用的大部分命令都放在这里

/usr/include ：存放C/C++等程序语言的头文件（head）和目标文件（include）

/usr/lib ：包含各应用软件的函数库，目标文件（object file），比如它下面有jvm目录，就是java

/usr/local ：系统管理员在本机自行下载自行安装的软件（非Ubuntu发行版默认提供的软件）一般放在该目录。该目录下也有bin,etc, include, lib等子目录。

/usr/sbin：非系统正常运行所需要的系统命令。最常见的就是某些网络服务器软件的daemon命令，如nginx, ntpd, mysqld

/var：如果/usr 是安装时会占用较大硬盘容量的目录，那么/var 就是在系统运行过程中渐渐占用硬盘容量的目录。包括缓存cache，日志log，以及某些软件运行所产生的文件，包括程序文件（lock file, run file）。mysql的数据库文件也是放置在这个目录下，具体为/var/lib/mysql/目录下

/var/cache： 应用程序缓存目录

/var/lib：存放程序执行过程中，需要使用到的数据文件

/var/lock：它是/run/lock目录的软链接，某些设备或文件一次只能被一个应用所使用

/var/log ：日志文件目录

