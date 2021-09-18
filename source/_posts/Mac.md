---
title: Mac VNC的坑
categories: 
  - MacOs
tags:
  - Mac
  - VNC
---


# VNC登陆时问题
如果遇到的VNC登陆Mac的时候遇到转圈圈进不去的但ssh能登陆的情况  

## 解决文案  
  
>ssh进系统>切换root账户>`ps -ef|grep login`>观察屏幕输出内容，  
>关掉除`/System/Library/CoreServices/logind`进程外的其他所有  
> `/System/Library/CoreServices/loginwindow.app/Contents/MacOS/loginwindow`  
> 进程

