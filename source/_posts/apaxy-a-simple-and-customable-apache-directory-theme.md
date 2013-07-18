---
layout: post
title: "Apaxy - 轻便精致的Apache目录主题"
date: 2013/05/06
tags: Apache
categories: web
---
<img src="http://adamwhitcroft.com/apaxy/img/example.png">
Apaxy是我一直在使用的Apache目录主题，它改变了Apache默认的目录列表样式，它的可扩展性和可定制型
极大地改善了用户的浏览体验。Apaxy主要使用了mod_autoindex这个模块以及一些CSS文件来对默认样式进行
重写。

在开始介绍之前，你可以先看看罗伊的 [演示](http://dearroy.info) 以及它的 [GitHub repo](https://github.com/AdamWhitcroft/Apaxy)。

## 特色
Apaxy可能看起来十分简单，但你的创造性和动手能力足以让它变得更加符合你的心意：

- 使用CSS定制默认目录样式
- 使它和 `Javascript` 或者 `JQuery` 结合
- 添加欢迎信息、下载提示或者版权提示
- 自定义mime类型图标 （需要.htaccess协同工作）

## 部署
首先，Apaxy需要在 `Apache(2.0.23+)` 环境下才能正常工作。

现在，假设你有一个目录名字叫share想要作为你的网页目录（即http://yourdomain.tld/share):

- 从Github下载Apaxy并解压
- 上传 `/apaxy` 文件夹中的内容到你的服务器或主机的 `/share` 目录
- 编辑 `htaccess.txt` （现在在 `/share` 目录下），对相应的路径进行更新以匹配你当前的主机或服务器
- 完成编辑后，将 `/share` 目录和 `/share/theme` 下的 `htaccess.txt` 重命名为 `.htaccess` 

大功告成，访问你的指定地址看看，是不是变样了？



