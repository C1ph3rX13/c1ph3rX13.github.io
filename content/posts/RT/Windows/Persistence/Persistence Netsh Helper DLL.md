---
title: Persistence Netsh Helper DLL
date: 2023-09-14T16:39:21+08:00
draft: false
url: /posts/2023-09-14/Netsh-Helper-DLL
tags:
  - Windows
  - Persistence
  - Netsh-Helper-DLL
---

## 0x00 前言

Persistence Netsh Helper DLL

## 0x01 Netsh Helper DLL

Netsh 是一个 Windows 实用程序，管理员可以使用它来执行与系统网络配置相关的任务，并对基于主机的 Windows 防火墙进行修改

Netsh 功能可以通过使用DLL 文件来扩展，需要本地管理员级别的权限，启动时会留下cmd

### 注册表位置

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NetSh
```

## 0x02 利用方法

```cmd
# 添加 DLL
netsh add helper <path to dll>

# 添加自启动注册表
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run" /v netsh /t REG_SZ /d "cmd /c start /b C:\Windows\System32\netsh"
```
