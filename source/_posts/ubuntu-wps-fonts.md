---
title: 解决ubuntu-20.04下wps字体缺失的问题
date: 2021-09-06 16:32:50
tags:
- ubuntu
- wps
---

> 今天用安装了很久但一直没用的wps打开了个`doc` 文档, 结果发现,字体缺失.
下面就解决ubuntu-20.04下wps字体缺失的问题


1.获取字体

找一个windows机器, 将`c:\windows\fonts` 目录打包,弄到ubuntu机器上。

或者你网上搜索一下，下载过来也行。

2. 导入字体

将获取的字体解压到 `/usr/share/fonts/wps-office`

