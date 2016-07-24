---
title: 聊聊Android的启动模式
tags:
---
1. task 返回栈
2. 案例：ProxyActivity会被游戏Activity的singleTask清掉
3. 解决方案：CP不愿意改ａ.用一个单独的启动页面　　b.不使用new task，使其在一个task里面
