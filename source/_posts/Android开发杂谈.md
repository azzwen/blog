---
title: Android开发杂谈
date: 2016-03-31 16:12:28
tags:
    - Android
---
记录 Android 开发过程中遇到的奇奇怪怪的事情。
<!-- more -->

<!-- toc -->

# Android Studio
## Android Device Monitor
### 窗口消失之谜
不知道点击的哪里，所有的窗口都不见了，如图所示。
![](/assets/images/Android 开发杂谈/monitor_empty.png)
尝试了两种方式：
- Window -> Show View -> Any View 
- Window -> Reset Perspective

但是不起任何作用，差点重装 Android Studio，还好有 Google 。搜索关键字：**Android Device Monitor empty**，找到了解决方案 - [链接地址](http://stackoverflow.com/questions/24723585/android-studio-android-device-monitor-empty-view)。
解决方法也比较简单。首先尝试 **Reset Perspective**，如果不起作用，就删除 `%USERPROFILE%\.android\monitor-workspace`。Windows 下目录为：`C:\Users\your_username\.android`。
> **monitor-workspace** 为 DDMS 的配置信息目录，相关问题都可以通过删除目录解决。