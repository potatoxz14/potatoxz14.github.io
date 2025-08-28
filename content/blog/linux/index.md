---
title: 🎉 Linux
summary: 关于修改Linux内核
date: 2025-1-27

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
---

## 删除内核

1.查看系统存在的内核

```shell
dpkg --list | grep linux-image
```

2.显示当前的内核

```shell
uname -a
```

3.升级内核

```shell
sudo apt-get dist-upgrade
```

4.删除内核

带有image的文件是需要删除的，一定要写全版本等字符。其他相关文件会自动删除。

```shell
sudo apt-get remove --purge linux-image-2.6.32-24-generic
```

5.删除内核

```shell
sudo apt-get remove --purge linux-headers-2.6.32-24-generic
```

6.更新menu.list

```shell
sudo update-grub
```

7.系统垃圾清理

```shell
sudo apt-get autoclean 
```

清理旧版本的软件缓存

```shell
sudo apt-get clean 
```

清理所有软件缓存

```shell
sudo apt-get autoremove
```

 删除系统不再使用的孤立软件





## 编译内核

Ubuntu下重新编译内核的基本方法和过程
编译内核基本方法
安装必备软件

```shell
sudo apt install make build-essential libncurses5-dev bison flex libssl-dev libelf-dev dwarves aptitude zstd -y
```



从www.kernel.org上下载Linux内核源码，例如 linux-6.2.7.tar.xz，然后解压。

```shell
xz -d linux-
tar -xvf linux-6.2.7.tar.xz
```

进入linux-6.2.7文件夹。

```shell
cd linux-6.2.7
执行如下动作

patch -p1 < path/to/patchfile.patch
```

```shell
//vim Makefile   //该过程可以修改版本号
make mrproper
make clean
make menuconfig
```

在执行完make menuconfig后，会产生图形界面。选择下方的save，将内核配置保存在.config文件中，然后选择exit退出。在编译内核之前，可以通过修改.config的内容来修改内核配置。

配置完成后，编译内核并安装。

```shell
sudo make -j$(nproc)
sudo make modules
sudo make modules_install
sudo make install
```

## 查看所有内核

```bash
dpkg --get-selections | grep linux
```

## 修改启动内核顺序：

```shell
1、 查看当前内核的启动顺序
cat /boot/grub/grub.cfg |grep menuentry
sudo  vim /etc/default/grub    //
#*********
GRUB_DEFAULT=0              #默认启动第0个
GRUB_TIMEOUT_STYLE=menu       #显示选择启动菜单
GRUB_TIMEOUT=15                   #菜单停留多少秒
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=1"
GRUB_CMDLINE_LINUX=""
#############################


sudo update-grub
```

重启系统。

```shell
sudo reboot now
```

重启完成后查看内核版本。以及日志持续输出

```shell
uname -r
dmesg --follow

```

```shell
sudo apt-get remove	linux-headers-4.15.0-39
sudo apt-get remove	linux-headers-4.15.0-39-generic
sudo apt-get remove	linux-image-4.15.0-39-generic
sudo apt-get remove	linux-modules-4.15.0-39-generic
sudo apt-get remove	linux-modules-extra-4.15.0-39-generic

# 可以使用purge连配置文件里一起彻底删除，清理内核列表
# sudo apt-get purge	linux-headers-4.15.0-39
# sudo apt-get purge	linux-headers-4.15.0-39-generic
# sudo apt-get purge	linux-image-4.15.0-39-generic
# sudo apt-get purge	linux-modules-4.15.0-39-generic
# sudo apt-get purge	linux-modules-extra-4.15.0-39-generic
```

清理内核

状态为 deinstall 表示已经卸载，如果不想显示 deinstall 这些项，并删除它们在 /lib/modual/ 下面还有这些内核的配置信息，可以采用下面的命令完全删除，如果还在就手动删：

```shell
sudo dpkg -P linux-image-4.15.0-39-generic 
```

 ## 其他内核名称可以用 tab 键自动补全来查看


证书问题：https://blog.csdn.net/phmatthaus/article/details/124353775

## 修改版本名称：MAKEFILE

```shell
vim Makefile
```

## 修改cgroupv1->v2

```shell
sudo  vim /etc/default/grub
systemd.unified_cgroup_hierarchy=1
GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=1"
sudo update-grub
sudo reboot now
```

## 添加输出

```c
if (strstr(current->comm, "cat"))
printk("%p",f1);
```

输出函数名

```
内核中函数指针用的很多，在debug 的时候能直接打印出一个函数指针对应的函数就会很方便。

打印裸指针(raw pointer)用 %p,%p除了可以用来打印指针外还可以打印其它的信息
%pF可打印函数指针的函数名和偏移地址，%pf只打印函数指针的函数名，不打印偏移地址。
如
printk("%pf %pF\n", ptr, ptr) will print:

module_start module_start+0x0/0x62 [hello]
但是为了支持这个功能你需要开启CONFIG_KALLSYMS 选项
```

## ubuntu同步时间

1.打开终端输入以下命令安装ntpdate工具。

```javascript
sudo apt-get install ntpdate
```

复制

2.再输入命令设置系统时间与网络时间同步。

```javascript
sudo ntpdate cn.pool.ntp.org
```

复制

3.最后输入命令将时间更新到硬件上即可。

```javascript
sudo hwclock --systohc
```

```
#!
```

在unix中，凡是被#!注释的，统统是加载器的路径，常见的有#!/bin/sh，表示将该注释后的代码交给/bin/sh处理，我们常见的处理python脚本一般在bash中输入python run.py，而如果在run.py中将注释换成#!/path/to/python，则在bash中执行run.py即可。

CVE-2019-6736修复方式：

最后，应用以下修复程序来缓解漏洞：

1、创建一个memfd（仅存在于内存中的特殊文件）。

2、将原始runc二进制文件复制到这一fd中。

3、在进入命名空间之前，从这个fd重新执行runc。

```
这一修复程序能够保证，如果攻击者覆盖了/proc/self/exe指向的二进制文件，将不会对宿主机造成任何损害，因为这是宿主机二进制文件的副本，完全存储在内存中（tmpfs）。
这一部分内存referably doing self-clean-up and not being writeable
```

## 屏蔽输出

```
./command > /dev/null 2>&1
```

## 查看内存是否可回收

```shell
查看使用objects或内存最多的slab内存信息。

slabtop -s -a

cat /sys/kernel/slab/<slab NAME>/reclaim_account
#例如，查看名称为kmalloc-192的slab内存是否为不可回收。

cat /sys/kernel/slab/kmalloc-192/reclaim_account
#查询结果为0时，表示slab内存不可回收；查询结果为1时，表示slab内存可回收。
```

### 查看内核配置都有哪些

```
cat /boot/config-$(uname -r)
```

```c

```

## 编译报错

```
make[2]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
```

```
vim .config
```

修改CONFIG_SYSTEM_TRUSTED_KEYS
修改CONFIG_SYSTEM_TRUSTED_KEYS，将其赋空值。
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"

修改CONFIG_SYSTEM_REVOCATION_KEYS（可选）
如果CONFIG_SYSTEM_REVOCATION_KEYS的值不为空的话，也将其赋空值。

make -j8
