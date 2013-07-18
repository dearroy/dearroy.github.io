---
layout: post
title: "Debian 7 使用Dotdeb源安装LNMP服务器"
date: 2013/06/20
tags: Linux
categories: linux
---
大概在很早之前，我还一直使用着军哥写的LNMP一键包，因为安装方便，配置简单。但经过一段时间后发现，这其实妨碍了我们系统和深入地
学习和理解Linux。

这里是为了方便一个朋友，昨天晚上给他配置了一台服务器，但没有告诉他过程，所以写一篇比较简单的教程给他备忘。


首先导入Dotdeb源，因为系统使用的官方源所安装的软件版本较低，所以这里推荐使用Dotdeb源。

在向源中导入Dotdeb源之前，我们需要先获取并导入GnuPG key：

{% codeblock lang:bash %}
wget http://www.dotdeb.org/dotdeb.gpg
cat dotdeb.gpg | apt-key add -
{% endcodeblock %}

提示OK后则导入成功。然后我们开始导入Dotdeb源，使用vim打开 `/etc/apt/sources.list`：

```
vim /etc/apt/sources.list
```

向其中添加如下两行：
{% codeblock lang:bash %}
deb http://packages.dotdeb.org wheezy all
deb-src http://packages.dotdeb.org wheezy all
{% endcodeblock %}

添加之后，更新一下源：

```
apt-get update
```

<!-- more -->

接下来就开始安装 MySQL + PHP + Nginx 了。

## 1，安装MySQL

执行命令：

```
apt-get install -y mysql-server mysql-client
```

即可安装MySQL，安装过程中会询问 root密码 ，键入你需要的密码之后回车即可。

安装完成后，执行如下命令进行一步安全设置：

```
mysql_secure_installation
```

按照提示进行，过程中会询问是否更改 root密码，是否移除匿名用户，是否禁止root远程登录等。

## 2，安装PHP

执行命令：

```
apt-get install php5-fpm php5-gd php5-mysql php5-memcache php5-curl
```

上面的命令安装了php5-memcache的扩展，于是继续安装 Memcached 。

```
apt-get install memcached
```

安装完毕之后，使用 php5-fpm -v 查看一下PHP的版本：

{% codeblock lang:bash %}
root@ztbox:~# php5-fpm -v
PHP 5.4.16-1~dotdeb.1 (fpm-fcgi) (built: Jun  8 2013 22:20:42)
Copyright (c) 1997-2013 The PHP Group
Zend Engine v2.4.0, Copyright (c) 1998-2013 Zend Technologies
{% endcodeblock %}

## 3，安装Nginx

关于Nginx，拾人牙慧一下：

Nginx ("engine x") 是一个高性能的 HTTP 和反向代理服务器，也是一个 IMAP/POP3/SMTP 代理服务器。 
Nginx 是由 Igor Sysoev 为俄罗斯访问量第二的 Rambler.ru 站点开发的，它已经在该站点运行超过三年了。
Igor 将源代码以类BSD许可证的形式发布。

Nginx 超越 Apache 的高性能和稳定性，使得国内使用 Nginx 作为 Web 服务器的网站也越来越多，
其中包括新浪博客、新浪播客、网易新闻、腾讯网、搜狐博客等门户网站频道，六间房、56.com等视频分享网站，Discuz!官方论坛、水木社区等知名论坛，
盛大在线、金山逍遥网等网络游戏网站，豆瓣、人人网、YUPOO相册、金山爱词霸、迅雷在线等新兴Web 2.0网站。

这里我直接安装了Nginx的全部扩展功能（nginx-full)，以应对以后可能出现的功能性增强。

```
apt-get install -y nginx-full
```

然后启动Nginx:

```
service nginx start
```

<img src="http://pics.dearroy.com/image.php?dm=PBRE" alt="welcome to nginx" />

访问结果如上图，接下来配置Nginx。

```
vim /etc/nginx/sites-available/default
```

{% codeblock lang:bash %}
……
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
    #    # NOTE: You should have "cgi.fix_pathinfo = 0;" in php.ini
    #
    #    # With php5-cgi alone:
    #   fastcgi_pass 127.0.0.1:9000;
    #    # With php5-fpm:
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }
……
{% endcodeblock %}

修改保存之后重启Nginx：

```
service nginx restart
```

接下来我们新建一个phpinfo，查看php的详细信息：

```
vim /usr/share/nginx/html/phpinfo.php
```

{% codeblock lang:php %}
<?php phpinfo(); ?>
{% endcodeblock %}

保存之后访问 http://ip/phpinfo.php , 如果出现 phpinfo 页面，则大功告成。

## 如何新建站点

和军哥的一键包不同，此方法所安装的 LNMP 需要手动添加站点配置文件。

`cd /etc/nginx/conf.d` 进入配置文件目录，新建一个站点配置文件，比如 `vi dearroy.com.conf`，

{% codeblock lang:bash %}
server {
    listen 80;
	
	#ipv6
    #listen [::]:80 default_server;
	
    root /usr/share/nginx/html/dearroy.com;
	
	#默认首页文件名
    index index.php index.html index.htm;
    
	#绑定域名
    server_name localhost;
	
	#伪静态规则
	include wordpress.conf;
	
    location / {
        try_files $uri $uri/ /index.html;        
    }
	#定义错误页面
    #error_page 404 /404.html;  
	
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass 127.0.0.1:9000;
         fastcgi_index index.php;
         include fastcgi_params;
     }
     #PHP
}
{% endcodeblock %}

保存之后重启Nginx，添加及绑定网站即完成。

最后，附两个最常用的程序Nginx伪静态：

WordPress:

{% codeblock lang:bash %}
location / {
if (-f $request_filename/index.html){
                rewrite (.*) $1/index.html break;
        }
if (-f $request_filename/index.php){
                rewrite (.*) $1/index.php;
        }
if (!-f $request_filename){
                rewrite (.*) /index.php;
        }
}
{% endcodeblock %}


Discuz X:

{% codeblock lang:bash %}
rewrite ^([^\.]*)/topic-(.+)\.html$ $1/portal.php?mod=topic&topic=$2 last;
rewrite ^([^\.]*)/article-([0-9]+)-([0-9]+)\.html$ $1/portal.php?mod=view&aid=$2&page=$3 last;
rewrite ^([^\.]*)/forum-(\w+)-([0-9]+)\.html$ $1/forum.php?mod=forumdisplay&fid=$2&page=$3 last;
rewrite ^([^\.]*)/thread-([0-9]+)-([0-9]+)-([0-9]+)\.html$ $1/forum.php?mod=viewthread&tid=$2&extra=page%3D$4&page=$3 last;
rewrite ^([^\.]*)/group-([0-9]+)-([0-9]+)\.html$ $1/forum.php?mod=group&fid=$2&page=$3 last;
rewrite ^([^\.]*)/space-(username|uid)-(.+)\.html$ $1/home.php?mod=space&$2=$3 last;
rewrite ^([^\.]*)/([a-z]+)-(.+)\.html$ $1/$2.php?rewrite=$3 last;
if (!-e $request_filename) {
        return 404;
{% endcodeblock %}



