---
title: Information Gathering Windows
date: 2023-08-23T15:26:24+08:00
draft: false
url: /posts/2023-08-23/Information-Gathering-Windows
tags:
  - Windows
  - Information-Gathering
slug: English-Preview
---
> Information Gathering Windows
> <!--more-->
# 域环境

## 判断是否域环境

~~~powershell
# 查看当前⽹卡和IP信息
ipconfig /all

# 查看系统详细信息
systeminfo

# 查看当前登录域及域用户
net config workstation

# 查看域内时间
net time /domain
~~~

## 域内信息

~~~powershell
# 获取域SID
whoami /all

# 查询域列表
net view /domain

# 查看主域控制器
netdom query pdc

# 查看所有域控制器列表
net group "Domain Controllers" /domain
# ping -n 1 <Domain Controllers> 来确定域控制器的IP

# 查询域信任信息
nltest /domain_trusts

# 查询域密码信息
net accounts /domain

# 查看指定域中的机器列表
net view /domain:<domain name>

# 导出域详细信息
# WinServer 2008 内置工具，安装了AD DS或者Active Directory轻型目录服务服务器角色则此功能可用
csvde -setspn <domain name> -f C:\windows\temp\xxx.csv
~~~

## 域用户信息

```powershell
# 查询域内用户
net user /domain

# 查看域用户的详细信息
net user <username> /domain

# 查询域管理员列表
net group "domain admins" /domain

# 查看登陆本机的域管理员
net localgroup administrators /domain

# 查看域内所有用户组列表
net group /domain

# 查看域用户详细信息
wmic useraccount get /all
```

## 定位域控

```powershell
# 查看域内时间
net time /domain

# 查看主域控制器
netdom query pdc

# DNS记录
# 若当前主机的dns为域内dns，可通过查询dns解析记录定位域控
nslookup -type=all _ldap._tcp.dc._msdcs.rootkit.org

# SPN扫描
setspn -T <domain name> -Q */*

# 查询信任域
nltest /domain_trusts /all_trusts /v /server:<domain ip>

# 查询域详细信息
nltest /dsgetdc:<domain name> /server:<domain ip>

# 查询域机器
net group "domain computers" /domain
```

# 工作组环境

## 系统信息

~~~powershell
# 获取本机⽹络配置信息
ipconfig /all

# 查询操作系统和版本信息
# 英⽂版系统
systeminfo | findstr /B /C:"OS Nmae" /C:"OS Version"
# 中⽂版系统
systeminfo | findstr /B /C:"OS 名称" /C:"OS 版本"

# 查看系统体系结构
echo %PROCESSOR_ARCHITECTURE%

# 查看安装的软件及版本、路径
wmic product get name,version 

# 查询本机服务信息
wmic service list brief

# 查看计划任务
schtasks /query /fo LIST /v

# 查看主机开机时间
net statistics workstation

# 查询补丁信息
systeminfo
# 查询补丁详情
wmic qfe get Caption,Description,HotFixID,InstalledOn
~~~

## 进程信息

查看当前进程列表和软件进程

~~~powershell
tasklist
~~~

查看当前进程列表对应的用户身份

~~~powershell
tasklist /v
~~~

查看当前进程是否有杀毒软件（AV）

```powershell
tasklist /svc
```

## 程序信息

查看启动程序信息

```powershell
wmic startup get command,caption
```

查看目标主机上安装的杀毒软件

```powershell
wmic /namespace:\\root\securitycenter2 path antivirusproduct GET displayName,productState, pathToSignedProductExe
```

## 用户信息

查看有哪些用户

```powershell
net user
```

查看当前在线用户

```powershell
query user || qwinsta
```

获取本地管理员(通常含有域用户)信息：

```powershell
net localgroup administrators
```

## 网络信息

查看本机端口开放情况

```powershell
netstat -ano
```

查询路由表及所有可⽤接⼝的ARP缓冲表

```powershell
route print
```

查看防⽕墙配置

```powershell
# 弃用命令
netsh firewall show config

# Windows 10/11 命令
netsh advfirewall /?
netsh advfirewall show allprofiles
```

查询并开启远程桌⾯连接服务

```powershell
# 得到连接端口为0xd3d，转换后为3389
REG QUERY "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" /V PortNumber

# 远程桌面连接服务的状态
sc query termservice
sc start termservice
sc config termservice start= auto
```

开启远程桌面的命令

```powershell
# 在 Windows Server 2003 中开启 3389 端口
wmic path win32_terminalservicesetting where (__CLASS !="")  call setallowtsconnections 1

# 在 Windows Server 2008 和 Windows Server 2012 中开启 3389 端口 
wmic /namespace:\\root\cimv2\terminalservices path win32_terminalservicesetting where (__CLASS !="") call setallowtsconnections 1

# 在 Windows 7 中开启 3389 端口
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f

# 防火墙放行3389端口
netsh advfirewall firewall add rule name="Remote Desktop" dir=in protocol=TCP localport=3389 action=allow
# 查看防火墙策略
netsh advfirewall firewall show rule name="Remote Desktop"
```

探测内网中的存活机器

~~~powershell
for /l %i in (1,1,255) do @ping 192.168.0.%i -w 1 -n 1 | find /i "ttl="

# Usages:
for /l %i in (1,1,255) do @ping 192.168.52.%i -w 1 -n 1 | find /i "ttl="
~~~

## 凭证信息

WiFi密码

```powershell
Netsh wlan show profiles
```

# 密码文件搜集

## 低权限下搜集当前机器各类密码文件

在当前目录及其子目录中搜索指定文件

`dir`：表示目录列表命令

+ `/b`：表示以简洁方式（不包括文件详细信息）列出文件和目录

+ `/s`：表示包括指定目录下的所有子目录

~~~powershell
dir /b /s user.*,pass.*,config.*,username.*,password.*

dir /a /s /b "D:\*.txt"
dir /a /s /b "D:\*pass*"
dir /a /s /b "D:\*login* 
dir /a /s /b "D:\*user*  
dir /a /s /b D:\\password.txt  
dir /a /s /b "D:\*.conf" "D:\*.ini" "D:\*.inc" "D:\*.config"
dir /a /s /b "C:\*.txt" "C:\*.xls*" "C:\*.xlsx*" "C:\*.docx" | findstr "拓扑"
/C 参数来指定要查找的字符串
dir /a /s /b "C:\*.conf" "C:\*.ini*" "C:\*.inc*" "C:\*.config" | findstr /C:"运维"
dir /a /s /b "D:\*.txt" "D:\*.xls*" "D:\*.xlsx*" "D:\*.docx" | findstr /C:"密码"
~~~

编写bat

```bat
@echo off 
set "drive=D:"
dir /a /s /b "%drive%\*.txt" >> result.txt
dir /a /s /b "%drive%\*pass*" >> result.txt
dir /a /s /b "%drive%\*login* >> result.txt
dir /a /s /b "%drive%\*user* >> result.txt
dir /a /s /b "%drive%\password.txt" >> result.txt
dir /a /s /b "%drive%\*.conf" "%drive%\*.ini" "%drive%\*.inc" "%drive%\*.config" >> result.txt
dir /a /s /b "%drive%\*.txt" "%drive%\*.xls*" "%drive%\*.xlsx*" "%drive%\*.docx" | findstr "拓扑" >> result.txt
dir /a /s /b "%drive%\*.conf" "%drive%\*.ini*" "%drive%\*.inc*" "%drive%\*.config" | findstr /C:"运维" >> result.txt
dir /a /s /b "%drive%\*.txt" "%drive%\*.xls*" "%drive%\*.xlsx*" "%drive%\*.docx" | findstr /C:"密码" >> result.txt
echo "find success"
```

## 循环搜集当前机器各类敏感密码配置文件

~~~powershell
for /r c:\ %i in (pass.*) do @echo %i
~~~

## findstr 查找某个文件的某个字段

`findstr`：表示字符串查找命令

- `/c:"user"`：表示指定要搜索的第一个字符串 "user"
- `/c:"pass"`：表示指定要搜索的第二个字符串 "pass"
- `/si`：表示在文件中搜索时，忽略大小写
- `*.txt`：表示要搜索的文件名模式，这里使用通配符 `*` 匹配任意字符，而 `.txt` 表示文件扩展名为 .txt 的文本文件。

~~~powershell
findstr /c:"user" /c:"pass" /si *.txt
~~~
