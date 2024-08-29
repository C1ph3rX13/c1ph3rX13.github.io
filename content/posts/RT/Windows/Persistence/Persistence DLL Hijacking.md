---
title: Persistence DLL Hijacking
date: 2023-09-13T17:39:41+08:00
draft: false
url: /posts/2023-09-14/DLL-Hijacking
tags:
  - Windows
  - Persistence
  - DLL-Hijacking
slug: English-Preview
---
> Persistence DLL Hijacking
<!--more-->

# DLL Hijacking

当程序启动时，许多 DLL 被加载到其进程的内存空间中，Windows 通过按特定顺序查找系统文件夹来搜索进程所需的 DLL

## MSDTC

分布式事务协调器是一个 Windows 服务，负责协调数据库 (SQL Server) 和 Web 服务器之间的事务

当此服务启动时会尝试从 System32 加载三个 DLL 文件：oci.dll、SQLLib80.dll、xa80.dll

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\MSDTC\MTxOCI
```

### oci.dll

在默认的 Windows 安装中，System32 文件夹中缺少 ***“oci.dll ”***。这使得有机会将任意 DLL 植入到该文件夹中，该文件夹具有相同的名称（需要管理员权限），以便执行恶意代码

```cmd
sc qc msdtc
sc config msdtc start= auto

net stop msdtc && net start msdtc
msdtc -install
```

## Narrator 旁白

Microsoft Narrator 是一款适用于 Windows 环境的屏幕阅读应用程序

本地化设置相关的 DLL 丢失 (MSTTSLocEnUS.DLL)，并且可能被滥用来执行任意代码

当“ *Narrator.exe* ”进程启动时，DLL 将被加载到该进程中

```cmd
# 系统语言为中文时，可能不起作用

C:\Windows\System32\Speech\Engines\TTS\MSTTSLocEnUS.DLL
```

