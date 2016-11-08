---
title: webview缓存机制
tags:
---

http://blog.csdn.net/t12x3456/article/details/13745553

webiew的cookie存放在数据库中
简单理解为　domain:key:value　形式
api提供removeAll
没法查询
需要更改单条记录

背景：
淘宝登录进入了系统繁忙页面
关闭页面再次进入依然是此页面，无法切换账号
