---
title: '[android] 原生安卓系统wifi或移动网络叹号'
date: 2025-02-12 03:02:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

先删除默认的验证地址：
adb shell "settings delete global captive_portal_http_url"
adb shell "settings delete global captive_portal_https_url"

再修改新的验证服务器地址：
adb shell "settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204"
adb shell "settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204"

更新时钟同步服务器地址：
adb shell "settings put global ntp_server ntp1.aliyun.com"


谷歌相机：
```
https://www.celsoazevedo.com/files/android/google-camera

```
