---
layout: post
title: "在CentOS 6上部署Xen虚拟化环境"
date: 2013/06/18
tags: Linux
categories: linux
---

众所周知，在centos 6中，官方源已经去除了xen的rpm包，只能使用第三方源或自行编译，因为编译安装对于我这种菜鸟来说实在太过于繁琐，
所以我选择来自 [Steve](http://xen.crc.id.au/) 的这个第三方源来部署。

如果这篇日志算是一篇教程的话，这是一篇偏实际操作的教程，至于理论教程，大家可以自行搜索学习，融会贯通。

## 1. 准备工作

首先检查SELinux是否已经禁用。

```
vi /etc/sysconfig/selinux
```

{% codeblock lang:bash %}
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#       enforcing - SELinux security policy is enforced.
#       permissive - SELinux prints warnings instead of enforcing.
#       disabled - SELinux is fully disabled.
SELINUX=disabled
# SELINUXTYPE= type of policy in use. Possible values are:
#       targeted - Only targeted network daemons are protected.
#       strict - Full SELinux protection.
SELINUXTYPE=targeted
{% endcodeblock %}

修改完毕保存之后，使用 `reboot` 命令重启服务器。

<!-- more -->

## 2. 创建一个网桥

给服务器创建一个网桥，如此一来，此服务器上的所有虚拟服务器就可以像物理系统进入。要创建网桥，我们需要安装 `bridge-utils`。

```
yum install bridge-utils
```

安装完成之后，我们开始配置网桥，创建一个 ` /etc/sysconfig/network-scripts/ifcfg-br0`，并从 `/etc/sysconfig/network-scripts/ifcfg-eth0`中将
`IPADDR, PREFIX, GATEWAY, DNS1和DNS2`这些参数复制过来，并确保你使用的是 `TYPE=Bridge` 而不是TYPE=Ethernet:

首先创建ifcfg-br0:

```
vi /etc/sysconfig/network-scripts/ifcfg-br0
```

{% codeblock lang:bash %}
DEVICE="br0"
NM_CONTROLLED="yes"
ONBOOT=yes
TYPE=Bridge
BOOTPROTO=none
IPADDR=192.168.0.88
PREFIX=16
GATEWAY=192.168.0.1
DNS1=8.8.8.8
DNS2=8.8.4.4
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME="System br0"
{% endcodeblock %}

然后修改ifcfg-eth0，并注释掉 `BOOTPROTO, IPADDR, PREFIX, GATEWAY, DNS1, and DNS2`，添加 `BRIDGE=br0`。

```
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

{% codeblock lang:bash %}
DEVICE="eth0"
NM_CONTROLLED="yes"
ONBOOT=yes
HWADDR=00:1E:90:F3:F1:07
TYPE=Ethernet
#BOOTPROTO=none
#IPADDR=192.168.0.88
#PREFIX=16
#GATEWAY=192.168.0.1
#DNS1=8.8.8.8
#DNS2=8.8.4.4
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME="System eth0"
UUID=5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03
BRIDGE=br0
{% endcodeblock %}

保存之后，重启网络。

```
/etc/init.d/network restart
```

执行 `ifconfig`，查看当前网络情况：

{% codeblock lang:bash %}
[root@dearroy ~]# ifconfig
br0       Link encap:Ethernet  HWaddr 00:1E:90:F3:F1:07
          inet addr:192.168.0.88  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::21e:90ff:fef3:f002/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:29 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1196 (1.1 KiB)  TX bytes:2794 (2.7 KiB)

eth0      Link encap:Ethernet  HWaddr 00:1E:90:F3:F1:07
          inet6 addr: fe80::21e:90ff:fef3:f002/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:4554 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3020 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:6249612 (5.9 MiB)  TX bytes:254928 (248.9 KiB)
          Interrupt:25 Base address:0x6000

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1304 (1.2 KiB)  TX bytes:1304 (1.2 KiB)

[root@dearroy ~]#
{% endcodeblock %}

## 3. 安装Xen

首先要检测你的CPU是否支持HVM虚拟化（硬件虚拟化），使用如下命令：

```
egrep '(vmx|svm)' --color=always /proc/cpuinfo
```

输出结果中如果有如下显示，则说明你的CPU支持HVM虚拟化，如果什么都没有显示，则说明你的CPU支持paravirtualization虚拟化，即PV虚拟化，而不是HVM虚拟化。

{% codeblock lang:bash %}
flags		: fpu vme de pse tsc msr pae mce cx8 apic mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx lm constant_tsc arch_perfmon pebs bts rep_good aperfmperf pni dtes64 monitor ds_cpl vmx tm2 ssse3 cx16 xtpr pdcm dca lahf_lm dts tpr_shadow vnmi flexpriority
flags		: fpu vme de pse tsc msr pae mce cx8 apic mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx lm constant_tsc arch_perfmon pebs bts rep_good aperfmperf pni dtes64 monitor ds_cpl vmx tm2 ssse3 cx16 xtpr pdcm dca lahf_lm dts tpr_shadow vnmi flexpriority
{% endcodeblock %}

开头已经说过，因为CentOS 6 是基于 RedHat 6，官方源已经去除了xen的rpm包，所以这里我们使用一个第三方的源。

```
yum install http://au1.mirror.crc.id.au/repo/el6/x86_64/kernel-xen-release-6-5.noarch.rpm
```

然后我们安装Xen, 

```
yum install kernel-xen xen
```

这样之后，Xen以及Xen内核就已经安装到我们的服务器上了。在我们重启系统进入Xen内核之前，我们需要检查一下`GRUB bootloader`的启动引导配置，

```
vi /boot/grub/menu.lst
```

可以看到第一个内核应该就是刚刚安装的Xen内核：

{% codeblock lang:bash %}
[...]
title CentOS (2.6.32.54-1.el6xen.x86_64)
        root (hd0,0)
        kernel /vmlinuz-2.6.32.54-1.el6xen.x86_64 ro root=/dev/mapper/VolGroup00-LogVol00 rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD quiet SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=de rd_LVM_LV=VolGroup00/LogVol01 rd_LVM_LV=VolGroup00/LogVol00 rd_NO_DM
        initrd /initramfs-2.6.32.54-1.el6xen.x86_64.img
[...]
{% endcodeblock %}

我们需要注意的是，Xen管理必须最先被加载。在 `kernel /vmlinuz...` 这一行，将第一个单词由`kernel`替换为`module`，下一行也一样操作
——将 `initrd /initramfs...` 修改为 `module /initramfs...`。

然后在以上提到的两行之前添加一行： 

```
kernel /xen.gz dom0_mem=1024M cpufreq=xen dom0_max_vcpus=1 dom0_vcpus_pin
```

添加之后，你的这一部分应该是如下这样：

{% codeblock lang:bash %}
[...]
title CentOS (2.6.32.54-1.el6xen.x86_64)
        root (hd0,0)
        kernel /xen.gz dom0_mem=1024M cpufreq=xen dom0_max_vcpus=1 dom0_vcpus_pin
        module /vmlinuz-2.6.32.54-1.el6xen.x86_64 ro root=/dev/mapper/VolGroup00-LogVol00 rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD quiet SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=de rd_LVM_LV=VolGroup00/LogVol01 rd_LVM_LV=VolGroup00/LogVol00 rd_NO_DM
        module /initramfs-2.6.32.54-1.el6xen.x86_64.img
[...]
{% endcodeblock %}

最后，检查一下 default 的值，如果不是0，修改为0，以确保第一个内核（也就是Xen内核）可以默认载入。

一份完整的 `/boot/grub/menu.lst` 如下：

{% codeblock lang:bash %}
# grub.conf generated by anaconda
#
# Note that you do not have to rerun grub after making changes to this file
# NOTICE:  You have a /boot partition.  This means that
#          all kernel and initrd paths are relative to /boot/, eg.
#          root (hd0,0)
#          kernel /vmlinuz-version ro root=/dev/mapper/VolGroup00-LogVol00
#          initrd /initrd-[generic-]version.img
#boot=/dev/sde
default=0
timeout=5
splashimage=(hd0,0)/grub/splash.xpm.gz
hiddenmenu
title CentOS (2.6.32.54-1.el6xen.x86_64)
        root (hd0,0)
        kernel /xen.gz dom0_mem=1024M cpufreq=xen dom0_max_vcpus=1 dom0_vcpus_pin
        module /vmlinuz-2.6.32.54-1.el6xen.x86_64 ro root=/dev/mapper/VolGroup00-LogVol00 rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD quiet SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=de rd_LVM_LV=VolGroup00/LogVol01 rd_LVM_LV=VolGroup00/LogVol00 rd_NO_DM
        module /initramfs-2.6.32.54-1.el6xen.x86_64.img
title CentOS (2.6.32-220.el6.x86_64)
        root (hd0,0)
        kernel /vmlinuz-2.6.32-220.el6.x86_64 ro root=/dev/mapper/VolGroup00-LogVol00 rd_NO_LUKS LANG=en_US.UTF-8 rd_NO_MD quiet SYSFONT=latarcyrheb-sun16 rhgb crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=de rd_LVM_LV=VolGroup00/LogVol01 rd_LVM_LV=VolGroup00/LogVol00 rd_NO_DM
        initrd /initramfs-2.6.32-220.el6.x86_64.img
{% endcodeblock %}

最后，重启系统，准备进入xen内核。

## 4. 进入Xen内核

系统重启之后，不出意外服务器就已经是Xen内核了。

{% codeblock lang:bash %}
[root@gentlexen1 ~]# uname -a
Linux gentlexen.paulvps.com 3.8.6-1.el6xen.x86_64 #1 SMP Sat Apr 6 12:09:44 EST 2013 x86_64 x86_64 x86_64 GNU/Linux
{% endcodeblock %}

查看xm info:

{% codeblock lang:bash %}
[root@gentlexen1 ~]# xm info
host                   : gentlexen1.paulvps.com
release                : 3.8.6-1.el6xen.x86_64
version                : #1 SMP Sat Apr 6 12:09:44 EST 2013
machine                : x86_64
nr_cpus                : 8
nr_nodes               : 1
cores_per_socket       : 4
threads_per_core       : 2
cpu_mhz                : 3392
hw_caps                : bfebfbff:28100800:00000000:00007f40:77bae3ff:00000000:00000001:00000281
virt_caps              : hvm hvm_directio
total_memory           : 32662
free_memory            : 9697
free_cpus              : 0
xen_major              : 4
xen_minor              : 2
xen_extra              : .1
xen_caps               : xen-3.0-x86_64 xen-3.0-x86_32p hvm-3.0-x86_32 hvm-3.0-x86_32p hvm-3.0-x86_64 
xen_scheduler          : credit
xen_pagesize           : 4096
platform_params        : virt_start=0xffff800000000000
xen_changeset          : unavailable
xen_commandline        : dom0_mem=1024M cpufreq=xen dom0_max_vcpus=1 dom0_vcpus_pin
cc_compiler            : gcc (GCC) 4.4.7 20120313 (Red Hat 4.4.7-3)
cc_compile_by          : mockbuild
cc_compile_domain      : crc.id.au
cc_compile_date        : Sat Apr  6 12:57:25 EST 2013
xend_config_format     : 4
{% endcodeblock %}

到此，在CentOS 6上安装Xen的工作就已经完成了。接下来我还会写一篇如何安装api管理工具、编译安装libvirt以及安装xen guest的日志。

如果大家有什么疑问，可以在此页留言和我一起探讨。