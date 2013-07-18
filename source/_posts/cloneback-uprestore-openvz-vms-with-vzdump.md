---
layout: post
title: "使用vzdump备份和恢复OpenVZ VPS"
date: 2013/05/09
tags: Linux
categories: linux
---
这本质上来说不算一篇技术文，它仅仅是经验上的总结和部分官方文档的汉化，本只是留给自己备忘，若能给后来者带来一点点帮助，那也便是极好的、

## 准备工作

现在，我们假设有两台OpenVZ母鸡，均为OpenVZ虚拟化环境，现在需要从A母鸡向B母鸡备份转移VPS。

A服务器IP：192.168.0.100

B服务器IP：192.168.0.101

先假设两台服务器均为CentOS 6操作系统，实际上其他的操作系统版本也没有问题。

首先我们需要在两台服务器上都安装 `vzdump` 和 `rsync` 。

```  
yum install vzdump rsync
```


<!-- more -->

##创建镜像备份

此操作仅在A服务器上操作，首先我们使用 `vzlist -a` 查看当前运行的VPS列表。假设我们现在要备份一个VEID为102的VPS，则执行：

```
 vzdump --compress --dumpdir /home/backup --stop 102
```

- `--compress` 意味着备份将以压缩的形式保存，即tgz格式。

- `--dumpdir` 则规定了备份文件保存的位置，如果不进行规定，则默认在 `/vz/dump` 或者 `/var/lib/vz/dump` （取决于操作系统和版本）。

- `--stop` 则在备份VPS之前停止了VPS的工作，并且在完成备份后重新启动VPS。

对应的输出结果应该如下：

{% codeblock lang:bash %}
server1:/vz/dump# vzdump --compress --stop 102
INFO: starting backup for VPS 102 (/var/lib/vz/private/102)
INFO: starting first sync /var/lib/vz/private/102 to /var/lib/vz/dump/tmp9009
INFO: stopping vps
Stopping container ...
Container was stopped
Container is unmounted
INFO: final sync /var/lib/vz/private/102 to /var/lib/vz/dump/tmp9009
INFO: restarting vps
Starting container ...
Container is mounted
Adding IP address(es): 192.168.0.102
Setting CPU units: 1000
Configure meminfo: 65536
Set hostname: test.example.com
File resolv.conf was modified
Container start in progress...
INFO: vps is online again after 15 seconds
INFO: Creating archive '/var/lib/vz/dump/vzdump-102.tgz' (/var/lib/vz/dump/tmp9009/102)
Total bytes written: 340428800 (325MiB, 11MiB/s)
INFO: backup for VPS 102 finished successful (1.37 minutes)
server1:/vz/dump#
{% endcodeblock %}

备份完成后，可以检查一下备份文件的目录：

```
ls -l /vz/dump/
```

你将看到一个 `.tgz` 文件。

{% codeblock lang:bash %}
server1:~# ls -l /vz/dump/
total 147864
-rw-r--r-- 1 root root      1170 2008-11-20 17:40 vzdump-102.log
-rw-r--r-- 1 root root 151249685 2008-11-20 17:40 vzdump-102.tgz
server1:~#
{% endcodeblock %}

现在我们就开始使用 `scp` 命令将备份好的VEID 102 VPS迁移往B服务器（从A服务器的 /vz/dump/vzdump-102.tgz 到B服务器的 /home）

```
scp /vz/dump/vzdump-102.tgz root@192.168.0.101:/home
```

## 恢复VPS备份

此操作只需要在B服务器上完成。执行如下命令：

```
vzdump --restore /home/vzdump-102.tgz 250
```

- `250` 为VPS在B服务器上的新VEID，你可以使用任何值进行替换，当然为了方便也可以直接就使用原始的102作为VEID。

因为VPS进行了迁移，所以分配的IP可能会有所改变，假设`192.168.0.102`是其原IP，我们现在要给它分配一个新IP `192.168.0.250` ， 执行：

```
vzctl set 250 --ipdel 192.168.0.102 --save
```

先删除原来的IP，再添加新的IP：

```
vzctl set 250 --ipadd 192.168.0.250 --save
```

如果需要更改hostname，则执行：

```
vzctl set 250 --hostname test2.example.com --save
```

最后，启动VPS，迁移结束：

```
vzctl start 250
```


