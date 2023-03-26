# Linux实战技能100讲

## 系统操作篇

### 常见目录介绍
- / 根目录
- /root root用户的家目录
- /home/username 普通用户的家目录
- /etc 配置文件目录
- /bin 命令目录
- /sbin 管理命令目录
- /usr/bin /usr/sbin 系统预装的其他命令

### 万能的帮助命令

1. man、help、info 帮助

- man帮助用法演示
```BASH
man ls  # 获取ls命令的帮助
man man # 获取man命令的帮助，有很多重名的
man 7 man # 获取man命令第7章的命令
man -a passwd # 不清楚是passwd是配置、文件还是命令等，可以用-a列出所有情况
man -1 passwd # 获取passwd命令的帮助
```

man中相关操作
ctrl + F或者Page Down：向下翻页
ctrl + B或者Page Up：向上翻页

- help 帮助
shell 命令解释器自带的命令成为内部命令，其他的就是外部命令

如何区分内部、外部命令：
```
举例： 
type cd ->  cd is a shell builtin cd是内部命令
type ls -> ls is hashed(/bin/ls) ls是外部命令

举例：
help cd  内部命令使用help
ls --help  外部命令使用**--help 
```

- info 命令
info帮助比help更详细，作为help的补充，但info都是英文的。

### pwd和ls命令

1. pwd 显示当前的目录名称

切到超级管理员用户
- su - root

2. cd 更改当前的操作目录
- cd /path/to/.... 绝对路径
- cd ./path/to/... 相对路径
- cd ../path/to/... 相对路径

命令行补全功能，当输入大部分的命令时，按左边的tab键可命令补全。

3. ls 查看当前目录下的文件
ls [选项，选项] 参数

支持同时使用多个选项
- ls -l -r -t 
可集合为
- ls -lrt

常用参数
- -l 长格式显示文件
- -a 显示隐藏文件
- -r 逆序显示
- -t 按照时间顺序显示
- -R 递归显示

查看多个目录
- ls /root /

### 创建和删除目录

1. mkdir
- mkdir /a 建立在根目录下
- midkr a 建立在当前目录下
- mkdir a b c d 同时建立多个文件夹
- mkdir /a/b/c 建立多层目录
- mkdir -p /a/b/c/d/e/f/g 建立超多层目录
此时查看目录可使用： ls -R /a

删除目录
- rmdir /a 只能删除空白的目录
- rm -r /a 删除非空目录

2. cp
cp [原始目录] [目标目录]
- cp -r /root/a / 复制目录
- cp /root/a.txt /home 复制文件 

3. touch
touch [文件名]
- touch /filea
- cp -v /file /tmp 覆盖已有文件
- cp -p /file /tmp 复制的时候保留原有时间
- cp -a /file /tmp 保留权限、用户组、时间等参数设置，等同于-dpr

4. mv 
mv [原文件] [目标文件]
- mv /filea /fileb

5. 通配符
- file* 匹配全部字符，包含file
- file? 匹配任意一个字符
- [xyz] 匹配xyz任意一个字符
- [a-z] 匹配一个范围
- [!xyz]或[^xyz] 不匹配

### 文本查看命令

1. cat 文本内容显示到终端

2. head 查看文件开头
- head -n file 查看前几行内容

3. tail 查看文件结尾
常用参数-f文件内容更新后，显示信息同步更新

- tail -n file 查看末尾几行内容

4. wc 统计文件内容信息
- wc -l file 查看文件行数
按空格键进行分页查看

5. more 分屏查看内容，空格下一页

6. less

### 打包压缩和解压缩

Linux的备份压缩:
- 最早的Linux备份介质是磁带，使用的命令是tar
- 可以打包后的磁带文件进行压缩储存，压缩的命令是gzip和bzip2
- 经常使用的扩展名是 .tar.gz、tar.bz2、.tgz

1. tar打包命令
- c 打包
- x 解包
- f 指定操作类型为文件

- tar [cxf] <destination> <original> 文件存放到目标地址 源文件

2. 压缩和解压缩
可以使用gzip和bzip2命令单独操作，通常和tar命令配合操作
- z gzip 格式压缩和解压缩
- j bzip2 格式压缩和解压缩

- tar czf <destination> <original> 以gzip格式打包文件
- tar cjf <destination> <original> 以bzip2格式打包文件

- tar xf <original> -C <destination> 解压缩到指定目录
- tar zxf <original> -C <destination> 解压缩到指定目录
- tar jxf <original> -C <destination> 解压缩到指定目录

### Vim的四种模式
1. 正常模式（Normal-mode）
- 四个方向hjkl
h 左
l 右
j 下
k 上

- 复制、粘贴
yy 复制单行 p粘贴单行 3p粘贴3行
3yy复制3行（当前行往下三行，包括当前行）
单行无提示，多行有提示

y$ 复制当前光标位置到这一行的结尾字符
dd 剪切一整行
d$ 剪切当前位置到这一行的结尾
u 普通模式下，撤销，多次u多次撤销
u ctrl +r 重做，返回上一次撤销
x 删除指定字符，光标选中，按x
r+新字符 字符替换，光标选中按r再输入新字符

G 移动到指定行
:set nu 查看当前行
11G 移动到第11行
g 移动到第一行
G 移动到最后一行
^ 表示到这一行的开头
$ 表示到这一行的结尾（用于一行太长的情景）

2. 插入模式（Insert-mode）
i进入插入模式
I进去插入模式并且光标到当前行开头
a进去插入模式并且光标到当前光标的下一位
A进去插入模式并且光标到当前行的末尾
o进去插入模式并且光标到当前光标的下一行产生空行
O进入插入模式并且光标到当前行的上一行产生空行

: 表示末行模式

3. 命令模式（Command-mode）
:w + 文件名 保存到指定文件名，不接文件名表示保存到原始文件当中
:q 退出
:q! 强制退出
:wq! 强制写入退出
:! + 功能命令 如：! ipconfig，表示临时查看命令
/ +字符 表示查找某个字符n向下移动查找，shift n向上移动查找

:s/old/new 替换字符，默认所在行范围进行替换，整个文件范围替换使用：%s/old/new/g
（g表示全局）在指定范围替换使用 :起始行，结束行s/old/new/g（多次替换加/g，单次不需要）

:s +命令 表示单次修改设置生效，如nu，nonu，设置永久生效则需要去配置文件（/etc/vimrc）添加
set nu的配置
:set nohlsearch 取消搜索结果高亮

4. 可视模式（Visual-mode）
v 表示字符可视模式，以字符为单位选择
V 表示行可视模式，以行为单位选择
ctrl+v 进入可视模式，光标选中多行后按大写i，然后输入内容，再按两次ESC；
块删除ctrl+v进入可视模式，光标选中多行，按d进行删除

### 用户与权限管理

1. 用户与权限管理
- 多用户操作系统
- 用户管理常用命令
- 用户切换
- 用户配置文件
- 文件权限的表示方法
- 文件权限管理的常用命令

2. 多用户操作系统
- 多用户操作系统的目的是隔离
用户权限隔离、系统资源隔离、root用户与普通用户的区别

3. 用户管理常用命令
- useradd 新建用户
- userdel <username> 删除用户，userdel -r <username>，r可以连带home目录删除的干干净净
- passwd <username> 修改用户密码
- usermod -d /home/<oldname> <newname> 修改用户属性
- change 修改用户属性
- id [用户] 通过id查看系统已存在的用户
- root用户在/root下，非root用户在/home下
- 用户信息存放于 etc/passwd，用户密码存放于 etc/shadow

4. 组管理命令
- groupadd 新建用户组
添加用户到用户组
```bash
groupadd group1
useradd user1
usermod -g group1 user1
```
- groupdel 删除用户组

5. 用户切换
- su 切换用户
su - username 使用login shell方式切换用户
su - root 切换到管理员账户
- sudo 以其他用户身份执行命令
visudo 设置需要使用sudo的用户（组）

### 用户和用户组的配置文件介绍

- vim /etc/passwd（7个字段的含义）	
root:x:0:0:root:/root:/bin/bash
eagle:x:1000:1000:eagle:/home/wangpeng:/bin/bash
用户名:是否需要密码验证:uid:gid:注释:用户家目录:用户登录成功后用的命令解释器（如果设置成/sbin/nologin 表示不可登录）
	
- vim /etc/shadow
root:$6$......$......
master:$6$......$.....
用户名:用户密码（加密的）

- vim /etc/group
root:x:0:
组的名称:是否需要密码验证:组的gid:其他组设置
mail:x:12:postfix

### 查看文件权限的方法
1. 查看文件权限
```bash
-r w ------ 1 root root 1523 sep 28 12:00 xxx.cfg
# 类型 + 权限 + 所属用户和组 + 文件名
```

2. 文件类型
- - 普通文件
- d 目录文件
- b 块特殊文件
- c 字符特殊文件
- l 符号链接
- p 命名管道
- s 套接字文件 

3. 文件权限的表示方法
字符权限表示方法
- r 读
- w 写
- x 执行

数字权限的表示方法
- r = 4
- w = 2
- x = 1

示例
```bash
-rw-r-xr-- 1 username groupname mtime filename
```
- rw- 文件属主的权限
- r-x 文件属组的权限
- r-- 其他用户的权限
创建新文件有默认权限，根据umask值计算，属主和属组根据当前进程的用户来设定

4. 目录权限的表示方法
- x 进入目录
- rx 显示目录内的文件名
- wx 修改目录内的文件名

5. 修改权限命令
- chmod 修改文件、目录权限
chmod u+x /tmp/testfile 增加权限
chmod 755 /tmp/testfile
chmod g-r /tmp/testfile 用户组减少权限
chmod o=w <filename> 其他用户直接赋予权限
修改权限：u=user，g=group，o=other，a=all
- chown 更改属主、属组
chown :<ownname> <directory>
- chgrp 可以单独更改属组，不常用
chown <owngroup> <directory>

ctrl+r搜索之前输入的命令

6. 特殊权限
- SUID 用于二进制可执行文件，执行命令时取得文件属主权限 4
如 /usr/bin/passwd
chmod 4755 对应 rwsr-xr-x
- SGID 用于目录，在该目录下创建新的文件和目录，权限自动更改为该目录的属组 2
- SBIT 用于目录，该目录下新建的文件和目录，仅root和自己可以删除 1
如 /tmp
chmod 1777 对应drwxrwxrwt.

因为SUID对应八进制数字是4，SGID对于八进制数字是2，则“4755”表示设置SUID权限，“6755”表示同时设置SUID、SGID权限。

## 系统管理篇

### 网络管理

#### 网络状态查看
1. net-tools
ifconfig:
```
eth0 第一块网卡（网络接口）
第一个网络接口可能叫做下面的名字

eno1 板载网卡
ens33 PCI-E网卡
enp0s3 无法获取物理信息的PCI-E网卡
```

普通用户不能使用ifconfig则使用/sbin/ifconfig

2. 网卡接口命名修改：
- 网卡命名规则受biosdevname和net.ifnames两个参数影响
-编辑 /etc/default/grub文件，增加biosdevname=0 net.ifnames=0，如下：
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap net.ifnames=0 biosdevname=0 rhgb quiet"
- 更新grub
grub2-mkconfig -o /boot/grub2/grub.cfg
- 重启
reboot

网卡是ens33，修改biosdevname=0 net_ifnames=0，更新grubs重启，ifconfig的网卡还是ens33，需要手动修改 vim /etc/sysconfig/network-scripts/ifcfg-ens33
将 NAME= 和DEVICE= 两个设置项手动改为eth0 ，保存后重启即可

3. 查看网卡物理链接情况
- mii-tool eth0

4. 查看网关命令：
- route -n 使用-n参数不解析主机名

#### 网络配置
1. 网络配置命令：
- ifconfig <接口> <ip地址> [netmask 子网掩码]
- ifup <接口>
- ifdown <接口>

2. 网关配置命令
添加网关
- route add default gw <网关ip>
- route add -host <指定ip> gw<网关ip>
- route add -net <指定网段> netmask <子网掩码> gw<网关ip>

3. 网络命令集合
- ip addr ls: ifconfig
- ip link set dev eth0 up: ifup eth0
- ip addr add 10.0.0.1/24 dev eth1: ifconfig eth1 10.0.0.1 netmask 255.255.255.0
- ip route add 10.0.0/24 via 192.168.0.1: route add -net 10.0.0.0 netmask 255.255.255.0 gw 192.168.0.1

5. 网络配置文件
- ifcfg-eth0
- /etc/hosts

#### 网络故障排除
1. 网络故障排除命令
- ping 目标主机是否畅通
- traceroute 数据包经过中间路由的状态
```bash
traceroute -w 1 www.baidu.com
```
- mtr 数据包经过中间路由的状态（内容更丰富）
- nslookup 查看dns解析
- telnet 目标主机端口是否畅通
```
telnet <ip地址> <端口号> 
```
- tcpdump 网络抓包
```
tcpdump -i any -n port 80
tcpdump -i any -n host 10.0.0.1
tcpdump -i any -n host 10.0.0.1 and port 80 -w <destination>
```
- netstat 查看服务监听地址
```
netstat -ntpl 使用tcp方式抓包；p进程；tcp的状态
```
- ss 
```
ss -ntpl
```

#### 网络服务管理
1. 网络服务管理
网络服务管理程序分为两种，分别为SysV和systemd
- service network start|stop|restart
- chkconfig --list network
- systemctl list-unit-files NetworkManager.service
- systemctl start|stop|restart NetworkManager
- systemctl enable/disable NetworkManager

查看网络状态：
- service network status

网络配置存放位置
/etc/sysconfig/network-scripts

network只能service管理，network服务是centos6的网络默认管理工具；
centos7默认的服务管理工具换成了systemctl。

显示及更改SysV服务：
- chkconfig --list network
- chkconfig --level <level> network on|off

设置路由永久生效：
写入 /etc/rc.local

查看网卡配置文件NA查看主机名配置文件：
/etc/hostname

查看主机名到网络地址映射文件（如果hostname发生改变）：
/etc/hosts

永久更改主机名：
- hostnamectl set-hostname name.domain

查看网络服务报错具体信息：
- journalctl -xe

### 软件安装

#### 软件包管理器
包管理方便软件安装、卸载，解决软件依赖关系
- CentOS、RedHat使用yum包管理器，软件安装包格式为rpm
- Debian、Ubuntu使用apt包管理器，软件安装包格式为deb

#### rpm包和rpm命令
1. rpm包格式
vim-common<软件名称>-7.1.10-5<软件版本>.el7<系统版本>.x86_64<平台>.rpm

2. rpm命令
rpm命令常用参数
```
-q 查询软件包
-i 安装软件包
-e 卸载软件包
```

查看设备文件
- ls /dev -l

把光盘做成光盘镜像
- dd if=/dev/sr0 of=/xxx/xx.iso

挂载磁盘到指定目录
- mount /dev/sr0 /mnt

查询当前安装的rpm包，分屏查看，查看单个包
- rpm -qa
- rpm -qa | more
- rpm -q <packagename>

#### yum仓库
1. rpm包管理器的问题：
需要自己解决依赖关系；软件包来源不可靠。

2. yum配置文件
- /etc/yum.repos.d/CentOS-Base.repo
- wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
- 更新对应缓存 yum makecache

3. yum命令常用选项
- install 安装软件包
- remove 卸载软件包
- listl grouplist 查看软件包
- update 升级软件包

#### 源代码编译安装

安装方式
- 二进制安装
- 源代码编译安装
```
wget https://xxx.tar.gz
tar zxf xxx.tar.gz
cd directory
./configure --prefix=/usr/local/directory
make -j2
make install
```

#### 内核升级
1. rpm格式内核
- 查看内核版本 uname -r
- 升级内核版本 yum install kernel-3.10.0
- 升级已安装的其他软件包和补丁 yum update

2. 源代码编译安装内核
- 安装依赖包
yum install gcc gcc-c++ make ncurses-devel openssl-devel elfutils-libelf-devel
- 下载并解压缩内核
<https://www.kernel.org>
tar xvf linux-5.1.10.tar.xz -C /usr/src/kernels
- 配置内核编译参数
cd /usr/src/kernels/linux-5.1.10/
make menuconfig | allyesconfig | allnoconfig
- 使用当前系统内核配置
cp /boot/config-kernelversion.platform /usr/src/kernels/linux-5.1.10/.config
- 查看CPU
lscpu
- 编译
make -j2 all
- 安装内核
make modules_install
make install

df -h查看占用空间

3. 软件包仓库
- 安装新的软件仓库
yum install epel-release -y
- 安装指定kernel版本
yum install kernel-3.10.0

#### grub配置文件 
1. grub配置文件
- /etc/default/grub
- /etc/grub.d/ 引导模块
- /boot/grub2/grub.cfg
- grub2-mkconfig -o /boot/grub2/grub.cfg

2. 更改引导
grep ^menu /boot/grub2/grub.cfg 所有的内核
grub2-set-default 0 从第一个引导
grub2-editenv list 引导目录

挂载盘及赋予权限
mount -o remount,rw /sysroot
chroot /sysroot

bios界面修改密码
echo 123456 | passwd --stdin root

selinux配置文件
/etc/selinux/config

### 进程管理

#### 进程的概念与进程查看
1. 进程的概念
进程-运行中的程序，从程序开始运行到终止的整个生命周期是可管理的
终止分为正常终止和异常终止：
- 正常终止分为从main返回、调用exit等方式
- 异常终止分为调用abort、接收信号等

2. 进程的查看命令
进程是树形结构，进程和权限有密不可分的关系

查看命令：
- ps 查看进程（静态）
```
ps -e 相当于"-a"
ps -ef 显示更详细的信息
```
- pstree 查看进程树
- top 查看进程（动态）

3. 3个特殊的进程
idle进程(PID = 0), init进程(PID = 1)和kthreadd(PID = 2)
```
idle进程由系统自动创建, 运行在内核态

idle进程其pid=0，其前身是系统创建的第一个进程，也是唯一一个没有通过fork或者kernel_thread产生的进程。
完成加载系统后，演变为进程调度、交换

init进程由idle通过kernel_thread创建，在内核空间完成初始化后, 加载init程序, 并最终用户空间

由0进程创建，完成系统的初始化. 是系统中所有其它用户进程的祖先进程
Linux中的所有进程都是有init进程创建并运行的。首先Linux内核启动，然后在用户空间中启动init进程，再启动
其他系统进程。在系统启动完成完成后，init将变为守护进程监视系统其他进程。
```

#### 进程的控制命令
1. 进程的优先级调整
- 调整优先级
nice 范围从-20到19，值越小优先级越高，抢占资源就越多
```
nice -n 10 ./a.sh
```
renice 重新设置优先级
```
renice -n 15 <pid>
```
- 进程的作业控制
jobs [num] 查询在后台运行的进程
fg 1 可以将进程调回到前台
bg 1 可以将进程调回到后台

&符号
```
./a.sh & 程序保持在后台运行
ctrl + z 运行程序时，调回到后台且程序是停止状态，确认top指令s列为“T”
```

#### 进程的通信方式 - 信号
1. 进程间通信
信号是进程间通信方式之一，典型用法是：终端用户输入中断命令，通过信号机制停止一个程序的运行。

使用信号的常用快捷键和命令。
- kill -l 查看所有支持的信号
SIGINT 通知前台进程终止进程 ctrl + c
SIGKILL 立即结束程序，不能被阻塞和处理kill -9 pid

#### 守护进程和系统日志
1. 守护进程
- 使用nohup与&符号配合运行一个命令
```
nohup tail -f /var/log/messages &
```
nohup命令使进程忽略hangup命令

当前运行进程在/proc目录下
ls -l cwd 查看进程运行在哪个目录下
ls -l fd  查看文件描述符

- 守护进程（daemon）和一般进程差别
daemon是守护进程，是一种特殊的进程，没有控制终端也不和前台交互，一般用于服务的管理，
通过对daemon进程发不同的信号，控制它对应的进程起停。 daemon进程可以发送信号9结束，
但是1号进程无法结束，只能用halt来结束。

- 使用screen命令
screen 进入screen环境
ctrl+a d 退出（detached）screen 环境
screen -ls 查看screen的会话
screen -r sessionid 恢复会话

2. 系统日志
常见的系统日志
- /var/log 查看日志
- message 查看信息
- dmesg 内核运行相关信息
- cron 查看定时任务
- secure 查看安全日志

#### 服务管理工具systemctl
1. 服务（提供常见功能的守护进程）集中管理工具
- service 启动脚本放于/etc/init.d/
- systemctl 启动脚本放于/usr/lib/systemd/system/

2. systemctl常见操作
- systemctl start|stop|restart|reload|enable|disable 服务名称
enable/disable 随着开机运行或关闭
- 软件包安装的服务单元 /usr/lib/systemd/system

3. systemctl的服务配置
- [Unit]
Requires = 新的依赖服务
After = 新的依赖服务
```
After尾部新添加service，After下一行Requires=service
```

- [Service]
- [Install]
安装到哪个默认启动级别 /lib/systemd/system
systemctl get-default|set-default

查看启动的级别：
chkconfig --list

/lib/systemd/system查看软链接后的服务：
ls -l runlevel*.target
改变启动级别：
systemctl set-default xxx.target

#### SELinux简介
1. MAC（强制访问控制）与DAC（自主访问控制）

2. 查看SELinux的命令
- getenforce 分为enforcing、permissive、disabled三种状态
- selinux配置文件放置于/etc/selinux/config/
- /usr/sbin/sestatus
- ps -Z and ls -Z and id -Z

3. 关闭SELinux
setenforce 0 从enforcing临时改为permissive
/etc/selinux/sysconfig 永久更改状态，更新完重启

### 内存与磁盘管理

#### 内存和磁盘使用率查看
1. 内存使用率查看
- free 静态查看
free 显示总内存大小
free -m 按大小显示内存
free -g 按大小显示内存，按1024为整
- top 动态查看

2. 磁盘使用率的查看
- fdisk
fdisk -l 查看磁盘列表
fdisk -l /dev/sd? 查看磁盘类型
fdisk -l /dev/sd?? 属于哪块硬盘的哪块分区
- parted
parted -l 查看分区
- df
df -h 查看分区详细信息，作为fdisk的补充
- du
du <directory> 文件实际占用的空间
du -h <filename>
- du与ls的区别
ls -lh <directory> 查询空洞文件大小
du -lh <direcotry> 查询文件实际占用的大小
- dd 创建空文件
dd if=afile bs=4M count=10 of=afile 创建40M的afile
dd if=afile bs=4M count=10 seek=20 of=bfile 跳过20块（4 * 20 = 80M），创建一个120M的bfile

3. 杀进程
杀进程是根据/proc/进程id/oom_score和oom_adj文件来确定的。出现oom kill 的时机是
malloc()无法获得资源的时候触发的，选择kill进程时也会选择内存占用高，存活时间短的
进程--即多释放内存，少杀无辜进程为根本目标。

#### ext4 文件系统
1. 常见文件系统
Linux支持多种文件系统，常见的有ext4、xfs、NTFS

2. ext4文件系统
ext4文件系统基本结构
- 超级块
- 超级块副本
- i节点（inode）
ls -i
- 数据块（datablock）

echo写入和直接写入的区别：
```
echo 写入只会对文件的datablock作出改变
直接写入会对i节点作出改变
```

ext4文件系统深入
- 执行mkdir、touch、vi等命令后的内部操作
当前目录下 cp <file> <file1> 复制
当前目录下 mv <file> <file1> 更名
使用rm，就是把文件和i节点断开，好处是无论删除的文件多大删除的时间都是一致的
- 符号链接与硬链接
ln <file> <file1> 创建硬链接，使用同一个i节点
ln -s <file> <file1> 创建软连接，使用不同节点
- facl 文件访问控制列表
getfacl <file> 查看文件访问控制列表
setfacl -m u:user1:r <file> 用户/组赋予文件权限

#### 磁盘配额的使用

#### 磁盘的分区与挂载
1. 常用命令
- fdisk
fdisk -l 查看所有的磁盘
fdisk /dev/sdx 对空闲磁盘进行分区，主分区只能建立四个，超过要在扩展分区重新建立逻辑分区
- mkfs
mkfs. 做成指定格式的磁盘
mkfs.ext4 /dev.sdx 对指定磁盘进行指定格式初始化
- parted 分区大于2T
parted /dev/sdd
- mount
mkdir /mnt/sdx 创建一个目录
mount /dev/sdx /mnt/sdx 挂载磁盘到指定目录下
mount 查看日志最后一行确认挂载成功
进程操作最终会指向磁盘sdx

常见配置文件放于：
- /etc/fstab

2. 用户磁盘配额
取消挂载：
umount /mnt/sdx

xfs文件系统的用户磁盘配额quota：
mkfs.xfs /dev/sdb1
mkdir /mnt/disk1
mount -o uquota,gquota /dev/sdb1 /mnt/disk1
chmod 1777 /mnt/disk1
xfs_quota -x -c 'report -ugibh' /mnt/disk1
xfs_quota -x -c 'limit -u isoft=5 ihard=10 user1' /mnt/disk1

#### 交换分区（虚拟内存）的查看与创建
1. 增加交换分区的大小
mkswap <directory>
swapon 或者 swapoff <directory>

2. 使用文件制作交换分区
dd if=/dev/zero bs=4M count=1024 of=/swapfile
mkswap /swapfile
chmod 600 /swapfile 考虑安全问题
swapon /swapfile
/etc/fstab 挂载点和文件格式都选择swap

3. 有软件默认安装/opt/abc，但是空间不够，如何让它安装到挂载盘？
先对硬盘格式化，然后挂载到/opt/其他目录 ， 迁移/opt/abc 到 /opt/其他目录 ，
修改/etc/fstab确保下次启动依然生效

#### 软件RAID的使用
1. RAID的常见级别及含义
- RAID 0 striping 条带方式，提高单盘吞吐率
- RAID 1 mirroring 镜像方式
- RAID 5 有奇偶检验
只能坏一块磁盘
- RAID 10 是RAID 1与RAID 0的结合

2. mdadm创建阵列
mdadm -C /dev/md0(默认值) -a yes -l1(RAID级别) -n2(硬盘数) /dev/sda1 /dev/sda2(简略写法/dev/sd[b,c]1)
mdadm -D 查看磁盘阵列信息
echo DEVICE /dev/sd[b,c]1
echo DEVICE /dev/sd[b,c]1 >> /etc/mdadm.conf
mdadm -Evs >> /etc/mdadm.conf
mkfs.xfs /dev/md0
mount

#### 逻辑卷管理
- 逻辑卷和文件系统的关系
pvcreate /dev/sda /dev/sdb /dev/sdc 组成一个物理卷
pvs 查看所有的物理卷

vgcreate vg1 /dev/sdx 创建新的卷组
pvs

vgs 报告显示关于卷组的信息

lvs 虚拟服务器

- 为Linux创建逻辑卷
lvcreate -L 100M -n lv1 vg1 创建100M的lv1逻辑卷，卷组是vg1
- 动态扩容逻辑卷
xfs_growfs /dev/sdx 逻辑卷扩容

#### 系统综合状态查看
1. 使用sar命令查看系统综合状态
sar -u 1 10 CPU查看，每隔一秒做一次采样，采样10次
sar -r 1 10 内存查看
sar -b 1 10 IO查看
sar -b 1 10 磁盘读写查看
sar -q 1 10 进程查看

2. 使用第三方命令查看网络流量
- yum install epel-release
- yum install iftop
- iftop -P
监听eth0网络接口（第一个网卡）

## Shell篇

### 认识Bash
1. Linux的启动过程
BIOS-MBR-BootLoader(grub)-kernel-init-系统初始化-shell

dd if=/dev/sda of=mbr.bin bs=446 count=1
hexdump -C mbr.bin 查看硬盘的主引导记录

centos7的启动文件挂载在/boot/grub2下
grub2-editenv list查看引导的内核版本

centos6的启动文件挂载在/usr/sbin/init下
/etc/rc.d shell脚本存放初始化文件

查看启动脚本：
ls /sbin/grub2-
ls /sbin/grub2-mkconfig

2. Shell脚本
- UNIX的哲学：一条命令只做一件事

- 为了组合命令和多次执行，使用脚本文件来保存需要执行的命令
- 赋予该文件执行权限（chmod u+rx filename）

3. 标准的Shell脚本要包含的元素
- Sha-Bang
- 命令
- “#”号开头的注释
```
#!/bin/bash
#!/usr/bin/python
```
- chmod u+rx filename 可执行权限
- 执行命令
```
bash ./filename.sh 不需要chmod赋予执行权限
./filename.sh 需要在当前进程执行
source ./filename.sh 内建命令会对当前环境产生影响
. filename.sh 内建命令会对当前环境产生影响
```

4. 内建命令和外部命令的区别
- 内建命令不需要创建子进程
- 内建命令对当前Shell生效

5. 管道与重定向
- 管道与管道符
管道和信号一样，也是进程通信的方式之一
匿名管道（管道符）是Shell编程经常用到的通信工具
管道符“|”，将前一个命令执行的结果传递给后面的命令，如ps | cat

- 子进程与子Shell
子进程是Shell程序，称作子Shell
内部命令的结果不会传递给子Shell
在管道符两端放置内部命令，相当于打开了新的子shell。内部命令执行结束之后，子shell也会
跟着一起结束。因此在管道符两端放置内部命令，对当前的shell是不生效的。

- 重定向符号
一个进程默认会打开标准输入、标准输出、错误输出三个文件描述符
输入重定向符号“<”
```
read var < /path/to/a/file
```
输出重定向符号“>”（输出重定向，原有内容会清空） “>>”（追加重定向，追加内容） “2>”（错误重定向） “&>”（全部重定向）&1 引用重定向
```
echo 123 > /path/to/a/file
```
输入和输出重定向组合使用
```
cat > /path/to/a/file << EOF
I am $USER
EOF
```

6. 变量
- 变量的定义
变量名的命名规则：
字母、数字、下划线
不以数字开头

- 变量的赋值
为变量赋值的过程，称为变量替换
```
1. 变量名=变量值
a=123
2. 使用let为变量赋值
let a=10+20
3. 将命令赋值给变量
l=ls
4. 将命令结果赋值给变量，使用$()或者``
letc=$(ls -l /etc) 或letc=`ls /root`
5. 变量值有空格等特殊字符可以包含在""或''中
```

- 变量的引用
${变量名}称作对变量的引用
echo ${变量名}查看变量的值
${变量名}在部分情况下可以省略为 $变量名

- 变量的作用范围
变量的作用域：只运行在当前的shell终端
```
为避免变量失效，运行bash脚本由原来的bash xx.sh改为source xx.sh
```
变量的导出：export
变量的删除：unset

- 系统环境变量
环境变量：每个Shell打开都可以获得到的变量
env查询环境变量，set设置环境变量

将新的目录添加到环境变量，对当前shell的任意位置都生效的。
```
PATH=$PATH:/root
```

echo $? 可以确定上一条命令是否执行成功
echo $$ 显示当前进程的pid
echo $0 当前的进程名称

$1...$9数字不变，从第10个开始要变更为${10}

参数为空输出占位符：${1-_}

cd $_ 上一条命令的最后一个参数

- 环境变量配置文件
/etc/profile 系统环境变量和启动环境的环境变量，用于登录配置
/etc/profile.d/ 基于不同的shell执行不同的脚本
~/.bash_profile
~/.bashrc 当前用户特有的
/etc/bashrc 用于函数和别名

给命令书写路径增加一条新的路径：
export PATH=$PATH:/new/path

注意：不要在配置文件中echo内容，在配置中进行了echo，会导致scp
的时候返回这些echo，并认为是协议的一部分进行分析，从而出错。

7. 数组
- 定义数组
IPTS=(10.0.0.1 10.0.0.2 10.0.0.3)
- 显示数组的所有元素
echo ${IPTS[@]}
- 显示数组元素个数
echo ${#IPTS[@]}
- 显示数组的第一个元素
echo ${IPTS[0]}

8. 转义与引用
- 特殊字符
特殊字符：一个字符不仅有字面意义，还有元意（meta-meaning）
#注释
;分号
\转义符号
"和'引号

- 转义符号
单个字符前的转义符号
```
\n \r \t 单个字母的转义
\$\" \\ 单个非字母的转义
```

- 引用
"双引号 不完全引用在终端中包含对变量的解释
'单引号 完全引用，在终端中不包含对变量的解释
`反引号 执行命令

9. 运算符
- 赋值运算符
=赋值运算符，用于算数赋值和字符串赋值
使用unset取消为变量的赋值
=除了作为赋值运算符还可以作为测试操作符

- 算数运算符
+ - * / ** %
使用 `expr` 进行运算

- 数字常量
let "变量名=变量值""
变量值使用0开头为八进制
变量值使用0x开头为十六进制

- 双圆括号
是let命令的简化
((a = 10))
((a++))
echo $((10+20))

- 空指令
~空指令符

复制简化：
```
cp -v /etc/passwd /etc/passwd.bak
cp -v /etc/passwd{,.bak}
```

其他符号：
- #注释符
- ;命令分隔符，case语句的分隔符要转义;;
- :空指令
- .和source命令相同
- ~家目录
- ,分隔目录

10. 测试与判断
- 退出与退出状态
退出程序命令
exit
exit 10 返回10给Shell，返回值非0位不正常退出
$?判断当前Shell下前一个进程是否正常退出

- 测试命令test
test命令用于检查文件或者比较值
test可以做以下测试：文件测试、整数比较测试、字符串测试

- 使用if-then语句
test测试语句可以简化为[]符号
if-then语句的基本用法：
```
if [测试条件成立] 或命令返回值是否为0
then 执行相应命令
fi 结束
```

[]符号还有扩展写法[[]]支持&&、||、<、>

- 使用if-then-else语句
if-then-else语句可以在条件不成立时也运行相应的命令
```
if [测试条件成立]
then 执行相应命令
else 测试条件不成立，执行相应命令
fi 结束
```

- 使用if-elif-else语句
if-then-else语句可以在条件不成立时也运行相应的命令
```
if [ 测试条件成立 ]
then 执行相应命令
elif [ 测试条件成立 ]
then 执行相应命令
else 测试条件不成立，执行相应命令
fi 结束 
```

- 嵌套if的使用
if条件测试中可以再嵌套if条件测试
```
if [ 测试条件成立 ]
then 执行相应命令
   if [ 测试条件成立 ]
   then 执行相应命令
   fi
fi 结束
```
嵌套的结果和复合比较语句&&结果相同

11. 循环
- case语句和select语句可以构成分支
```
case "$变量" in
   "情况1" )
   命令... ;;
   "情况2" ）
   命令... ;;
   * )
   命令... ;;
   esac
```

- 使用for循环遍历命令的执行结果
for循环的语法
```
for 参数 in 列表
do 执行的命令
done 封闭一个循环
```
使用反引号或$()方式执行命令，命令的结果当作列表进行处理

- 使用for循环遍历变量和文件的内容
列表中包含多个变量，变量用空格分隔
对文本处理，要使用文本查看命令取出文本内容；默认逐行处理，出现空格会当作多行处理

- C语言风格的for命令
```
for((变量初始化；循环判断条件；变量变化))
do
	循环执行的命令
done
```

- while循环
```
while test测试是否成立
do
	命令
done
```

- 死循环
```
while test测试一直成立
do
	命令
done
```

- until循环
until循环测试为假时，执行循环，为真时循环停止

- break和continue语句
循环和循环可以嵌套
循环中可以嵌套判断，反过来也可以实现嵌套
循环可以使用break和continue语句在循环中退出
组合循环语句中使用continue会跳过该索引

- 使用循环对命令行参数的处理
命令行参数可以使用$1 $2 ... ${10}...$n进行读取
$0 代表脚本名称
$* 和 $@ 代表所有位置参数
$# 代表位置参数的数量
使用$1_方式代替$1避免变量为空导致的遗产

12. 函数
- 自定义函数
函数用于“包含”重复使用的命令集合
自定义函数
```
function fname(){
命令
}
```
函数的执行 fname
函数作用范围的变量 local变量名
函数的参数 $1 $2 $3 ... $n

- 系统脚本
系统自建了函数库，可以在脚本中引用，位于/etc/init.d/functions
自建函数库：使用source函数脚本文件“导入”函数

13. 脚本控制
- 脚本优先级控制
可以使用nice和renice调整脚本优先级
避免出现“不可控的”死循环：
死循环导致CPU占用过高，死循环导致死机

ulimit -a 查询当前终端的限制
 
fork炸弹

- 捕获信号
捕获信号脚本的编写（防止意外关闭脚本）：
kill默认会发送15号信号给应用程序
ctrl+c发送2号信号给应用程序
9号信号不可阻塞
```bash
#不建议在工作环境下测试
trap "echo sig 15" 15

echo $$

while :
do
  :
done
```

14. 计划任务
- 一次性计划任务
计划任务：让计算机在指定的时间运行程序
计划任务分为：一次性计划任务 周期性计划任务
一次性计划任务：at，没有shell交互，需要定向输出；编写完用ctrl+d结束，运行之后用atq去查询一次性任务
```
at 18:31
echo hello > /tmp/hello.txt
<EOT>
```

- 周期性计划任务
cron
配置方式：crontab -e
查看现有的计划任务：crontab -l
配置格式：分钟、小时、日期、月份、星期执行的命令；命令的路径问题
查看log是否被执行：/var/log/

- 计划任务加锁flock
加锁时的配置放在/etc/cron.d/和/etc/anarcrontab

如果不能按照预期时间运行：
anacontab 延时计划任务
flock 锁文件，保证任务只执行一次
```
flock -xn "/tmp/f.lock" -c "/root/a.sh" 排它锁放在/tmp下
```

## 文本操作篇

### 正则表达式与文本搜索

#### 元字符
- . 匹配除换行符外的任意单个字符
- * 匹配任意一个跟在它前面的字符
- [] 匹配方括号中的字符类中的任意一个
- ^ 匹配开头
- $ 匹配结尾
- \ 转义后面的特殊字符

```
让转义字符生效加引用
grep "\." anaconda-ks.cfg
```

#### 扩展元字符
- + 匹配前面的正则表达式至少出现一次 前面要加r
- ? 匹配前面的正则表达式出现零次或一次
- | 匹配它前面或后面的正则表达式

#### 文件的查找命令find
- find 路径 查找条件[补充条件]
```bash
find /etc -name passwd
find /etc -regex .*wd
```

#### 文本内容的过滤（查找）grep
- grep 选项 文本文件1 [...文本文件n]
- 常用选项

```
-i 忽略大小写
-r 递归读取每一个目录下的所有文件
```

分割并统计字符数目：
```
cut -d ":" -f7 /etc/passwd | uniq -c
```

### 行编辑器sed和AWK介绍

#### 行编辑器介绍
- Vim和sed、AWK的区别
交互式与非交互式
文件操作模式与行操作模式

- sed的基本用法
sed一般用于对文本内容做替换
```
sed `/user1/s/user1/u1` /etc/passwd
```

- AWK的基本用法
AWK一般用于对文本内容进行统计、按需要的格式进行输出
cut命令：cut -d:-f 1 /etc/passwd
AWK命令：awk -F:`/wd$/{print $1}` /etc/passwd

#### sed的替换命令
- sed的替换命令
sed的模式空间
替换命令s

- sed的模式空间
sed的基本工作方式是：
```
将文件以行为单位读取到内存（模式空间）
使用sed的每个脚本对该行进行操作
处理完成后输出该行
```

- 替换命令s
sed的替换命令s：
```
1. sed `s/old/new` filename
2. sed -e `s/old/new` -e `s/old/new/` filename 不替换原始文件
或 sed -e `s/old/new`;`s/old/new/` filename
3. sed -i `s/old/new/` `s/old/new/` filename 替换原始文件
4. sed 's!/!abc!' filename
```

- 使用正则表达式
带正则表达式的替换命令s:
```
sed `s/正则表达式/new/` filename
sed -r `s/扩展正则表达式/new/` filename
```

#### sed的替换命令加强版
sed的替换命令加强版
全局替换
标志位
寻址
分组
sed脚本文件

- 全局替换
s/old/new/g
g为全局替换，用于替换所有出现的次数
/如果和正则匹配的内容冲突可以使用其他符号，如：s@old@new@g

- 标志位
s/old/new/标志位
```
数字，第几次出现才进行替换
g，每次出现都进行替换
p打印模式空间的内容 sed -n `script` filename阻止默认输出
w file将模式空间的内容写入到文件
```

- 寻址
默认对每行进行操作，增加寻址后对匹配的行进行操作
```
/正则表达式/s/old/new/g
行号s/old/new/g 行号可以是具体的行，也可以是最后一行$符号
可以使用两个寻址符号，也可以混合使用行号和正则地址
```

- 分组 
寻址可以匹配多条命令
```
/regular/{s/old/new/;s/old/new/}
sed -r 's/(aa)|(bb)/!/' filename
sed -r 's/(a.*b)/\1:\1/' filename
```

- 脚本文件
可以将选项保存为文件，使用-f加载脚本文件
sed -f sedscript filename

#### sed的其他命令
- sed的其他命令
删除命令
追加、插入、更改
打印
下一行
读文件和写文件
退出命令

- 删除命令
[寻址]d
删除模式空间内容，改变脚本的控制流，读取新的输入行

- 追加插入和更改
追加命令a
插入命令i
更改命令c
```
sed '/ab/a hello' filename
sed '/ab/i hello' filename
sed '/ab/c hello' filename
```

- 打印
打印命令p 

- 下一行
下一行命令n
```
head -5 /etc/passwd | sed -n 's/root/!!!!/p'
```
打印行号命令=

- 读文件和写文件
读文件命令r
写文件命令w
```
head -5 /etc/passwd | sed -n 's/root/!!!!/w /tmp/a.txt'
```

- 退出命令
退出命令q

#### sed的多行模式
- sed的多行模式
多行模式的原因：
配置文件一般为单行出现，也有使用XML或JSON格式的配置文件，为多行出现

多行模式处理命令N、D、P：
N将下一行加入到模式空间
D删除模式空间中的第一个字符到第一个换行符
P打印模式空间中的第一个字符到第一个换行符
```
b.txt
hell
o bash hel
lo bash
sed 'N;s/\n//;s/hello bash/hello sed\n/;P;D' b.txt
```

交互式输入文本内容
```
cat > b.txt << EOF
```

#### sed的保持空间
什么是保持空间：
保持空间也是多行的一种操作方式；将内容暂存在保持空间，便于做多行处理
文本文件->模式空间->保持空间

保持空间命令
- h和H将模式空间内容存放到保持空间
- g和G将保持空间内容取出到模式空间
- x交换模式空间和保持空间内容
```
cat -n /etc/passwd | head -6 | sed -n '1h;1!G;$!x;$p'
cat -n /etc/passwd | head -6 | sed -n 'G;h;$p'
cat -n /etc/passwd | head -6 | sed -n '1!;G;h;$!d'
```

####  AWK
- AWK和sed的区别
AWK更像是脚本语言
AWK用于“比较规范”的文本处理，用于统计数量并输出指定字段
使用sed将不规范的文本，处理为“比较规范”的文本

- AWK脚本的流程控制
输入数据前例程BEGIN{}
主输入循环{}
所有文件读取完成例程END{}

- AWK的字段引用和分离
记录和字段：
每行称作AWK的记录
使用空格、制表符分隔开的单词称作字段
可以自己指定分隔的字段

字段的引用：
awk中使用$1 $2...$n表示每一个字段
```
awk '{print $1,$2,$3}' filename
```
awk可以使用-F选项改变字段分隔符，分隔符可以使用正则表达式
```
awk -F "," '{print $1,$2,$3}' filename
```

#### AWK的表达式
- 赋值操作符
=是最常用的赋值操作符：
```
var1= "name"
var2= "hello" "world"
var3= $1
```
其他赋值操作符
```
++ -- += -= *= /= %= ^=
```

- 算数操作符
算术操作符
```
+-*/%^
```

- 系统变量
F: 分隔符
FS和OFS字段分隔符，OFS表示输出的字段分隔符，用于变量内
RS记录分隔符
NR和FNR行数
NF字段数量，最后一个字段内容可以用$NF取出

```
读入文件之前要做的准备工作
head -5 /etc/passwd | awk 'BEGIN{FS=":"}{print NR}' 
head -5 /etc/passwd | awk 'BEGIN{FS=":";OFS="-"}{print NR}' 
head -5 /etc/passwd | awk '{print NR,$0}'
head -5 /etc/passwd | awk '{print FNR,$0}'
head -5 /etc/passwd | awk 'BEGIN{FS=":"}{print NF}'
head -5 /etc/passwd | awk 'BEGIN{FS=":"}{print $NF}' 输出最后一个字段的内容
```

- 关系操作符
关系操作符
```
< > <= >= == != ~ !~
```

- 布尔操作符
布尔操作符
```
&& || !
```

#### AWK的条件和循环
- 条件语句
条件语句使用if开头，根据表达式的结果来判断执行哪条语句
```
if(表达式)
  awk语句1
[else
  awk语句2
]
```
如果有多个语句需要执行可以使用{}将多个语句括起来

- 循环
while循环：
```
while(表达式)
  awk语句1
```

do循环：
```
do{
  awk语句1
}while(表达式)
```

for循环：
```
for(初始值；循环判断条件；累加)
   awk语句1
```
影响控制的其他语句break，continue

#### AWK的数组
- 数组的定义
数组是一组有某种关联的数据（变量），通过下标依次访问
数组名[下标]=值
下标可以使用数字也可以使用字符串

- 数组的遍历
for(变量in数组名)
使用数组名[变量]的方式依次对每个数组的元素进行操作
```
awk '{ sum=0; for(column=2;column<=NF;column++) sum+=$column;average[$1]=sum/(NF-1)}END{}'
awk '{ sum=0; for(column=2;column<=NF;column++) sum+=$column;
	average[$1]=sum/(NF-1)}END{ for( user in average) ; print user,average[user]}'
awk '{ sum=0; for(column=2;column<=NF;column++) sum+=$column;
	average[$1]=sum/(NF-1)}END{ for( user in average) sum2+=average[user] ;print sum2/NR}'
```

- 删除数组
delete数组[下标]

- 命令行参数数组
ARGC
ARGV
```
BEGIN{
	for(x=0;x<ARGC;x++)
	   print ARGV[x]
	print ARGC
}
```

数组示例
```
{
	sum=0
	for(column = 2; column <=NF;column++)
		  sum += $column

	average[$1] = sum/(NF-1)

	sum_all += average[$1]

}
END{
	average_all = sum_all/NR
	for( user in average )
		  if( average[user] > average_all )
		  		above++
		  else
		  		below++

	print "above",above
	print "below",below
}
```

#### AWK的函数
- 算术函数
sin() cos()
int()
rand() srand()

- 字符串函数
gsub(r,s,t)
index(s,t)
length(s)
match(s,r)
split(s,a,sep)
sub(r,s,t)
substr(s,p,n)

- 自定义函数
```
function 函数名（参数）{
   awk语句
   return awk变量
}
```

## 服务管理

### 防火墙

#### 防火墙分类

1. 软件防火墙和硬件防火墙

2. 包过滤防火墙和应用防火墙
CentOS6默认的防火墙是iptables
CentOS7默认的防火墙是firewallD（底层使用netfilter）

iptables命令总体用法：
1. table
```
-t nat, filter, mangle, raw
```
2. command
```
-A add一条规则到指定链接末尾
-D delete一条规则按号或内容删除
-I insert规则到指定链接中，默认第一行
-R replace指定链接中的某条规则按顺序号来
-L list出指定链接中的所有规则
-F flush清空
-N new一条自定义的规则链接
-X 删除指定表中用户自定义的规则链接
-P policy设置指定链接中的默认策略
```
3. chain
```
INPUT
OUTPUT
FORWARD
PREROUTING
POSTROUTING
```
4. parameter
```
-p protocol指定协议tcp,udp,icmp之一
-s source address指定源地址
-d dest address
-i 指定数据包从哪个设备进入
-o 指定数据包从哪个设备输出
-m 指定数据包规则要使用的模块，一般模块会有一些扩展功能/br> --sport
指定数据包源端口号
--dport 指定数据包目标端口号
```
5. target
-j ACCEPT 允许数据包通过
-j DROP 直接丢弃数据包
-j REJECT 拒绝数据包通过，必要时会返回一个响应

#### iptables的表和链
1. 规则表
filter nat mangle raw

2. 规则链
INPUT OUTPUT FORWARD
PREROUTING POSTROUTING

#### iptables的filter表
iptables -t filter命令 规则链 规则
1. 命令
-L
-A -I 
-D -F -P
-N -X -E

2. 规则
-P
-s -d
-i -o
-j

用完iptables -F后改回来用iptables -P INPUT ACCEPT

#### iptables的nat表
iptables -t nat 命令 规则链 规则
PREROUTING 目的地址转换
POSTROUTING 源地址转换

```
iptables -t nat -A PREROUTING -i ens33 -d 114.115.116.117 -p tcp --dport 80
-j DNAT --to-destination 10.0.0.1
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o ens331 -j SNAT --to-source
111.112.113.114
```

#### iptables的配置文件
/etc/sysconfig/iptables
Centos6
	service iptables save|start|stop|restart
Centos7
	yum install iptables-services

#### firewallD服务
1. firewallD的特点
支持区域“zone”概念
firewall-cmd

```
firewall-cmd --state 查看运行状态
firewall-cmd --list-all 查看运行的详情
firewall-cmd --zone=public --list-interfaces
firewall-cmd --list-ports 查看端口
firewall-cmd --list-services 支持的服务
firewall-cmd --get-zones 可选的区域
firewall-cmd --get-default-zone 默认激活的区域
firewall-cmd --get-active-zone 正在活跃的区域
firewall-cmd --add-service=https 添加服务
firewall-cmd --add-port=81/tcp 添加协议及端口
firewall-cmd --add-port=81/tcp --permanent 开机启动
firewall-cmd --remove-source=10.0.0.1 移除源ip
```

2. systemctl start|stop|enable|disable firewalld.service
停掉iptables功能 service iptables stop

### SSH服务

#### SSH服务介绍
telnet所有传输都是明文传输

安装telnet单机和服务端
yum install telnet telnet-server xinetd

启动
systemctl start xinetd.service
systemctl start telnet.socket
rpm -ql telnet-server

#### SSH服务配置文件
- sshd_config
Port22 默认端口
PermitRootLogin yes是否允许root登录
AuthorizedKeysFile.ssh/authorized_keys

#### SSH命令
1. systemctl status|start|stop|restart|enable|disable sshd.service
 
2. 客户端命令
ssh [-p端口] 用户@远程ip
SecureCRT
Xshell
putty

#### SSH公钥认证
1. 密钥认证原理

2. 常用命令
ssh-keygen -t rsa
ssh-copy-id
```
ssh-copy-id -i /root/.ssh/id_rsa.pub 用户@远程ip
```

#### scp和sftp远程拷贝
常用命令
scp
```
scp template.txt root@10.0.0.1:/tmp/
```
sftp
winscp

### FTP服务

#### FTP服务介绍
- FTP协议
主动模式和被动模式

#### vsftpd服务安装和启动
1. yum install vsftpd ftp

2. systemctl start vsftpd.service

3. 建议将selinux改为permissive
getsebool -a|grep ftpd
setsebool -P <sebool> 1

#### vsftpd服务配置文件
/etc/vsftpd/vsftpd.conf
/etc/vsftpd/ftpusers
/etc/vsftpd/user_list

!ls 查看本地文件

#### FTP命令
1. Linux客户端
FTP命令

2. Windows客户端
资源管理器
FTP工具

#### 使用虚拟用户进行验证
guest_enable=YES
guest_username=vuser
user_config_dir=/etc/vsftpd/vuserconfig
allow_writeable_chroot=YES
pam_service_name=vsftpd.vuser

配置表目录
/etc/vsftpd/vsftpd.conf

配置详情：
```
anonymous_enable=NO             # disable  anonymous login
local_enable=YES		# permit local logins
write_enable=YES		# enable FTP commands which change the filesystem
local_umask=022		        # value of umask for file creation for local users
dirmessage_enable=YES	        # enable showing of messages when users first enter a new directory
xferlog_enable=YES		# a log file will be maintained detailing uploads and downloads
connect_from_port_20=YES        # use port 20 (ftp-data) on the server machine for PORT style connections
xferlog_std_format=YES          # keep standard log file format
listen=NO   			# prevent vsftpd from running in standalone mode
listen_ipv6=YES		        # vsftpd will listen on an IPv6 socket instead of an IPv4 one
pam_service_name=vsftpd         # name of the PAM service vsftpd will use
userlist_enable=YES  	        # enable vsftpd to load a list of usernames
tcp_wrappers=YES  		# turn on tcp wrappers

userlist_enable=YES                   # vsftpd will load a list of usernames, from the filename given by userlist_file
userlist_file=/etc/vsftpd.userlist    # stores usernames.
userlist_deny=NO   

chroot_local_user=YES
allow_writeable_chroot=YES

semanage boolean -m ftpd_full_access --on

systemctl restart vsftpd
```

### Samba和NFS

#### 常见共享服务的区别
协议不同
对操作系统的支持程度不同
交互的便利性不同
Server Messages Block

#### Samba服务的安装
yum install samba

#### Samba服务的配置文件
/etc/samba/smb.conf

[share]
	comment= my share
	browseable = Yes
	writable = Yes
	path=/data/share
	create mask = 0775
	directory mask = 0775
	admin users = username
	valid users = username

```
关闭selinux
chcon -t samba_share "share_folder_path"
```

#### Samba用户的设置

samba是一种linux和windows之间进行文件共享的协议。安装该协议后，可以理解为在linux是插在windows上的一个U盘。

1. smbpasswd命令
-a 添加用户
```
smbpasswd -a username
```
-x 删除用户

2. pdbedit
-L 查看用户

#### Samba服务的启动和停止
1. systemctl start|stop smb.service

2. Linux客户端（samba密码）
mount -t cifs -o username=user1 //127.0.0.1/user1 /mnt

3. Windows客户端
资源管理器访问共享
映射网络驱动器

Windos上传文件到linux

#### NFS服务的配置
web集群中NFS服务器主要用于存储用户上传的信息，方便集群中机器获取用户数据。

1. /etc/exports/data/share *(rw,rsync,all_squash)

将远程访问的所有普通用户及所属组都映射为匿名用户或用户组

```
/data/share 10.0.0.1(ro) 10.0.0.2(rw)
```
2. showmount -e localhost
3. 客户端使用挂载方式访问
mount -t nfs localhost:/data/share /mnt
4. 启动NFS服务
systemctl start|stop nfs.service

### Nginx

#### Nginx

1. Nginx和Web服务介绍
Nginx是一个高性能的Web和反向代理服务器
Nginx支持HTTP、HTTPS和电子邮件代理协议
OpenResty是基于Nginx和Lua实现的Web应用网关，集成了大量的第三方模块

2. OpenResty软件的下载和安装
yum-config-manager --add-repo https://openresty.org/package/centos/openresty.repo
yum install openresty

3. OpenResty的配置文件
/usr/local/openresty/nginx/conf/nginx.conf
service openresty start|stop|restart|reload
 
4. 使用OpenResty配置域名虚拟主机
```
server {
	listen 80;
	server_name www.servera.com;
	location /{
		root html/servera;
		index index.html index.htm;
	}
}
```

生效
cd /usr/local/openresty/nginx/sbin
./nginx -t

重新启动
cd /usr/local/openresty/nginx/sbin
./nginx -s stop
./nginx -s reload 热更新

设置域名
cd /usr/local/openresty/nginx/sbin
vim /etc/hosts

更改首页
cd /usr/local/openresty/nginx/html
mkdir servername
echo servername > servera/index.html

curl http://www.servera.html 检验

### LNMP

1. MySQL安装
可以使用mariadb替代

yum install mariadb mariadb-server

修改默认编码
vim /etc/my.cnf
character_set_server=utf8
init_connect='SET NAMES utf8'

systemctl start mariadb.service
show variables like '%character_set%';

2. PHP安装
- PHP安装
yum install php-fpm php-mysql

- 启动php-fpm
system start php-fpm.service

3. Nginx配置
```
location ~ \.php$ {
	root html;
	fastcgi_pass 127.0.0.1:9000;
	fastcgi_index index.php;
	fastcgi_param SCRIPT_FILENAME $document_root $fastcgi_script_name;
	include fastcgi_params;
}
```

### DNS服务的原理
搭建主域名服务器
1. DNS服务介绍
- DNS（Domain Name System）域名系统
- FQDN（Full Qualified Domain Name）完全限定域名
- 域分类：根域、顶级域（TLD）
- 查询方式：递归、迭代
- 解析方式：正向解析、反向解析
- DNS服务器的类型：缓存域名服务器、主域名服务器、从域名服务器

2. BIND软件的安装
/etc/hosts
yum install bind bind-utils
systemctl start named.service

3. BIND的配置文件
/etc/named.conf
named-checkconf
rndc-reload

```
listen-on port { any; }; 如果127.0.0.1只有自己能查询，改为any
allow-query { any; };
zone "test.com" IN {
	type master;
	file "test.com.zone";
	allow-transfer { 10.211.55.3; };
}
```
/var/named/ 下有named.ca文件
```
$TTL 1D
@	 IN SOA @ns1.test.com. (
				   0    ;serial
				   1D    ;refresh
				   1H    ;retry
				   1W    ;expire
				   3H   ;minimum
)
@	 IN NS ns1
ns1  IN A  10.211.55.3
www  IN A  10.20.0.100
mail IN CNAME  mailexchange
mailexchange IN A 10.20.0.200
```

4. 使用dig和nslookup命令测试DNS
nslookup
dig @DNSSERVER Domain

5. 从域名服务器的配置

```
zone "test.com" IN {
	type slave;
	file "slaves/test.com.zone";
	masters { 10.211.55.3; };
};
```

反向解析配置文件
```
zone "0.20.10.in-addr.arpa" IN {
	type master;
	file "10.20.0.zone";
};
100 IN PTR www.test.com
```

### NAS

NAS(Network Attached Storage)网络附属存储
NAS支持的协议NFS、CIFS、FTP
保证数据安全方式 磁盘阵列