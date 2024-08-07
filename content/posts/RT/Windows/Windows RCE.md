---
title: Windows RCE
date: 2024-01-18T10:38:28+08:00
draft: false
url: /posts/2024-01-18/Windows-RCE
tags:
  - Windows
  - RCE
---

## 0x00 Description

Windows 远程执行命令

## 0x01 WinRM

Windows 远程管理 (WinRM) 是 [WS-Management 协议](https://learn.microsoft.com/zh-cn/windows/win32/winrm/ws-management-protocol)的 Microsoft 实现，该协议是标准简单对象访问协议 (基于 SOAP) 的防火墙友好协议，允许不同供应商的硬件和操作系统之间进行互操作。

WS-Management协议规范为系统提供了一种跨 IT 基础结构访问和交换管理信息的常用方法。 WinRM 和 [智能平台管理接口 (IPMI) ](https://learn.microsoft.com/zh-cn/windows/win32/winrm/windows-remote-management-glossary#i)标准，以及 [事件收集器服务](https://learn.microsoft.com/zh-cn/previous-versions/windows/it-pro/windows-server-2003/cc785056(v=ws.10)#event-collector) 是称为 [硬件管理](https://learn.microsoft.com/zh-cn/previous-versions/windows/it-pro/windows-server-2003/cc785056(v=ws.10))的功能集的组件。

### 利用条件

WinRM默认使用TCP协议，并依赖于以下两个端口：

1. HTTP（默认端口5985）：WinRM可以通过HTTP协议进行通信。当使用HTTP时，通信数据不加密，可能存在安全风险。
2. HTTPS（默认端口5986）：WinRM还可以通过HTTPS协议进行加密通信。使用HTTPS可以提供更高的安全性，确保通信数据的机密性和完整性。

### Quickconfig

```cmd
winrm quickconfig -q
winrm set winrm/config/Client @{TrustedHosts="*"}
netstat -ano | find "5985"
```

### 远程命令执行

```cmd
winrs -r:http://ip:5985 -u:username -p:password "whoami /all"
winrs -r:http://ip:5985 -u:username -p:password cmd
```

### Bypass UAC

```cmd
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f

winrs -r:http://ip:5985 -u:username -p:password "whoami /groups"
```

## 0x02 WMI

WMI 命令行 (WMIC) 实用工具为 Windows Management Instrumentation (WMI) 提供命令行接口。 WMIC 与现有的 shell 和实用工具命令兼容。 以下信息是 WMIC 的一般参考指南。 有关如何使用 WMIC 的详细信息和指南，包括有关别名、谓词、开关和命令的其他信息，请参阅 [使用 Windows Management Instrumentation 命令行](https://learn.microsoft.com/zh-cn/previous-versions/windows/it-pro/windows-server-2003/cc779482(v=ws.10)) 和 [WMIC - 对 WMI 进行命令行控制](https://learn.microsoft.com/zh-cn/previous-versions/windows/it-pro/windows-2000-server/bb742610(v=technet.10))。

### 利用条件

1. 远程服务器启动Windows Management Instrumentation服务（默认开启）

2. 135 端口未被过滤 ，默认配置下目标主机防火墙开启将无法连接

### 执行命令

```cmd
# 执行命令无回显
wmic /node:ip /user:username /password:password process call create "cmd.exe /c ipconfig > c:/1.txt"
```

