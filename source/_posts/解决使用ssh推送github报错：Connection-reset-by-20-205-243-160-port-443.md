---
title: 解决使用ssh推送github报错：Connection reset by 20.205.243.160 port 443
date: 2026-10-05 00:35:27
tags: bug复盘
author: Gaesar
---

## 报错含义

TCP 连接被远端服务器强行重置，不是密钥 / 权限错误，纯粹网络层面被中断。

Connection reset 含义：电脑发出连接请求，远端服务器收到后，直接主动断开 TCP 连接。

## 触发原因

流量特征触发临时检测。

短时间多次访问 github 的 ssh/https，会触发**临时的会话拦截**，不是永久拉黑。 短时间频繁重试 ssh、git push，更容易被识别，直接断连。

反复连续执行，越重试越容易被掐断。

## 修复

暂停等待 5~10 分钟后重试。
