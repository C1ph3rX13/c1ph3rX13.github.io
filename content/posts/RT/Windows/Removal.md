---
title: Removal
date: 2023-08-23T18:02:06+08:00
draft: false
url: /posts/2023-08-23/Removal
tags:
  - Windows
  - Removal
---

## 0x00 前言

Removal

微软文档：https://learn.microsoft.com/zh-cn/windows-server/administration/windows-commands/wevtutil

## 0x01 清除系统日志

### 通过 PowerShell 删除

#### 方法一

```powershell
# Usage
PowerShell -Command "& {Clear-Eventlog -Log 你要清理的日志(如Security)}"

PowerShell -Command "& {Clear-Eventlog -Log Application}"
PowerShell -Command "& {Clear-Eventlog -Log Security}"
PowerShell -Command "& {Clear-Eventlog -Log Setup}"
PowerShell -Command "& {Clear-Eventlog -Log System}"
PowerShell -Command "& {Clear-Eventlog -Log ForwardedEvents}"

# 一次性清空多种日志
PowerShell -Command "& {Clear-Eventlog -Log Application,System,Security}"
```

#### 方法二

```powershell
# Usage
PowerShell -Command "& Get-WinEvent -ListLog 你要清理的日志(如Security) -Force | % {Wevtutil.exe cl $_.Logname}"

PowerShell -Command "& Get-WinEvent -ListLog Security -Force | % {Wevtutil.exe cl $_.Logname}"
PowerShell -Command "& Get-WinEvent -ListLog Security -Force | % {Wevtutil.exe cl $_.Logname}"
PowerShell -Command "& Get-WinEvent -ListLog Security -Force | % {Wevtutil.exe cl $_.Logname}"
PowerShell -Command "& Get-WinEvent -ListLog Security -Force | % {Wevtutil.exe cl $_.Logname}"

# 一次性清空多种日志
PowerShell -Command "& Get-WinEvent -ListLog Application,Setup,Security -Force | % {Wevtutil.exe cl $_.Logname}"
```

### 使用 wevtutil 删除

#### 统计日志列表数目信息

```powershell
wevtutil.exe gli Application
wevtutil.exe gli Security
```

#### 查询指定类别的日志

```powershell
# Usage
wevtutil qe <log name> /c:<line> /f:text

wevtutil qe Security /f:text
```

#### 全量删除

```powershell
# 变量名需要大写
# Usage
wevtutil cl <log name>

wevtutil cl System
# 会留下一个事件id为1102的日志
```

#### 定向清理

```powershell
# Usage
wevtutil qe <log name> /f:text /rd:true /c:<line>

wevtutil qe Security /f:text /rd:true /c:10
```

#### 删除某指定单条记录

1. 删除 EventRecordID 指定的日志

```powershell
# 将 Windows 安全事件日志中除了 EventRecordID 为 1102 的条目之外的所有条目导出到指定的文件中
# "/q:*[System [(EventRecordID!=1102)]]" : 查询参数，用于筛选要导出的日志条目。
# 只导出 EventRecordID 不等于 1102 的日志条目
wevtutil epl Security tmp.evtx /q:"*[System[(EventID!=1102)]]"

# 只导出 EventRecordID 等于 1102 的日志条目
wevtutil epl Security tmp.evtx /q:"Event/System/EventID=1102"
```

2. 结束 evenvmr 进程

```powershell
net stop eventlog /y
```

3. 替换原日志文件

```
copy tmp.evtx %SystemRoot%\System32\winevt\Logs\Security.evtx
```

4. 重启日志服务

```powershell
net start eventlog
```

## 0x02 暴力删除日志文件

#### 停止 Windows Event Log 服务

```powershell
# 强制停止 Windows Event Log
net stop eventlog /y
```

#### 删除对应的文件

```powershell
# 日志文件路径
# 绝对路径
C:\Windows\System32\winevt\Logs

# 相对默认位置
%SystemRoot%\System32\winevt\Logs\System.evtx
%SystemRoot%\System32\winevt\Logs\Security.evtx
%SystemRoot%\System32\winevt\Logs\Application.evtx

# IIS 日志
%SystemDrive%\inetpub\logs\LogFiles\W3SVC1\
```

## 0x03 Windows 远程连接日志

### 删除 Default.rdp 文件

使用`attrib`去掉`Default.rdp`文件：系统文件属性(S)，隐藏文件属性(H)之后删除

```powershell
attrib "%userprofile%\documents\Default.rdp" -s -h && del "%userprofile%\documents\Default.rdp"
```

### 注册表清理

查询远程连接在注册表中的键值

```powershell
reg query "HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client\Default"
```

删除对应的键值

```powershell
reg delete "HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client\Default" /f /v MRU0
```

**如使用当前机器作为跳板的话需要使用同步骤清理Servers下的键值**

```powershell
reg query "HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client\Servers"
```

**删除上面`Servers`查询出来的键值**

```powershell
reg delete "HKEY_CURRENT_USER\Software\Microsoft\Terminal Server Client\Servers\192.168.52.138" /f 
```

## 0x04 删除文件

### Recent

```powershell
del /f /s /q "%userprofile%\Recent*.*"

C:/Users/Administrator/AppData/Local/Microsoft/Windows/History
```

### Amcache / RecentFileCache.bcf

Windows中的使用这两个文件来跟踪具有不同可执行文件的应用程序兼容性问题，它可用于确定可执行文件首次运行的时间和最后修改时间

在Windows 7、Windows Server 2008 R2等系统中，文件保存在 `C:\Windows\AppCompat\Programs\RecentFileCache.bcf` ，包含程序的创建时间、上次修改时间、上次访问时间和文件名

```powershell
del /f /s /q "C:\Windows\AppCompat\Programs\RecentFileCache.bcf"
```

在Windows 8、Windows 10、Windows Server 2012等系统中，文件保存在 `C:\Windows\AppCompat\Programs\Amcache.hve` ，包含文件大小、版本、sha1、二进制文件类型等信息

```powershell
del /f /s /q "C:\Windows\AppCompat\Programs\Amcache.hve"
```

### Prefetch

预读取文件夹，用来存放系统已访问过的文件的预读信息，扩展名为PF。位置在 `C:\Windows\Prefetch` 

```powershell
del /f /s /q "C:\Windows\Prefetch\*"
```

### JumpLists

记录用户最近使用的文档和应用程序，方便用户快速跳转到指定文件，位置在 `%APPDATA%\Microsoft\Windows\Recent` 

```powershell
del /f /s /q "%APPDATA%\Microsoft\Windows\Recent\*"
```

## 0x05 格式化磁盘

### 彻底格式化

多次覆写文件 

```powershell
cipher /w:<path>
```

格式化某磁盘

```powershell
format D: /P:<count>
```

