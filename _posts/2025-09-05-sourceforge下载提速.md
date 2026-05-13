---
title: 'sourceforge下载提速'
date: 2025-09-05 03:24:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

从sourceforge正常下载，拷贝下载链接，将其中前面部分替换成：
https://nchc.dl.sourceforge.net/project/
开始下载，即可加速

思路都是更换镜像，在下载的时候不等倒计时结束，点击 "problem downloading?"，切换不同的镜像服务器：
Download -> cancel download -> Click on "problem downloading?" -> Use a different server


可以查看 mirror列表，https://sourceforge.net/p/forge/documentation/Mirrors/
从中选择镜像地址进行替换
ref:https://hugo.utermux.dev/default/sourceforge-endpoint/

列出几个常用镜像：
台湾地区：nchc.dl.sourceforge.net
香港：zenlayer.dl.sourceforge.net
肯尼亚：liquidtelecom.dl.sourceforge.net
拉斯维加斯：versaweb.dl.sourceforge.net

有些下载器（FDM、IDM等下载器）支持多个镜像加速，可以多填写几个镜像加速