---
title: 'tensorflow1.x 在30/40卡配置'
date: 2024-04-29 03:14:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

ref: https://blog.csdn.net/ly869915532/article/details/124542362

tf1.x 官方已经不再支持了， 最新的tf1.15版本也停留在cuda10，而30系以上的新卡都不在支持cuda11之前的版本

解决：nvidia对tf1.15有维护分支，支持了新卡对旧版本tf的支持
https://github.com/NVIDIA/tensorflow
https://docs.nvidia.com/deeplearning/frameworks/tensorflow-wheel-release-notes/
