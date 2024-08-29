---
title: Information Gathering Linux
date: 2024-01-08T16:43:53+08:00
draft: false
url: /posts/2024-01-08/Information-Gathering-Linux
tags:
  - Linux
  - Information-Gathering
slug: English-Preview
---
> Information Gathering Linux
> <!--more-->

# Hidden

+ 取消环境变量 `HISTFILE` 的设置

```bash
unset HISTFILE
```

+ 将环境变量 `HISTFILE` 设置为 `/dev/null`

```bash
export HISTFILE=/dev/null
```

+ 彻底清除历史

```bash
history -c
```

+ 关闭Shell的历史记录功能

```bash
set +o history
```

# Network

- 列出网络接口信息

```bash
/sbin/ifconfig -a
ip addr show

cat /etc/network/interfaces 
cat /etc/sysconfig/network
```

- 查看系统arp表 

```bash
arp -a
```

- 打印路由信息

```bash
route
# 显示核心路由表
ip route show
# 显示邻居表
ip neigh        
```

- 查看DNS配置信息

```bash
cat /etc/resolv.conf
```

- 打印本地端口开放信息

```bash
# -a 表示显示所有活动的连接和监听端口，即显示所有状态的套接字
# -n 表示以数字形式显示IP地址和端口号，而不使用主机名和服务名
# -t 表示只显示TCP连接和监听端口
# -p 表示显示与每个套接字关联的进程ID（PID）和程序名称

netstat -antp
```

- 查看进程端口情况

```bash
netstat -anltp | grep $PID
```

+ 列出iptable的配置规则

```bash
iptables -L
```

- 查看端口服务映射

```bash
cat /etc/services
```

- Hostname

```bash
hostname -f
```

# System

- 版本信息

```bash
uname -a # 所有版本
uname -r # 内核版本信息
uname -n # 系统主机名字
uname -m # Linux内核架构
```

- 内核信息 

```bash
cat /proc/version
```

- CPU信息

```bash
cat /proc/cpuinfo
```

- 发布信息

```bash
cat /etc/*-release
cat /etc/issue
```

- 主机名

```bash
hostname
```

- 文件系统

```bash
df -a
```

- 内核日志 

```bash
dmesg
/var/log/dmesg
```

# Users

- 列出系统所有用户

```bash
cat /etc/passwd
```

- 列出系统所有组

```bash
cat /etc/group
```

- 列出所有用户hash（root）

```bash
cat /etc/shadow
```

- 当前登录的用户

```bash
users
who
/var/log/utmp
```

+ 查询无密码用户

```bash
grep 'x:0:' /etc/passwd
```

- 目前登录的用户

```bash
w
```

- 登入过的用户信息

```bash
last
/var/log/wtmp
```

- 显示系统中所有用户最近一次登录信息

```bash
 lastlog
 /var/log/lastlog
```

- 登录成功日志

```bash
/var/log/secure
```

- 登录失败日志

```bash
/var/log/faillog
```

- 查看特权用户

```bash
grep :0 /etc/passwd
```

- 查看是否存在空口令用户

```bash
awk -F: 'length($2)==0 {print $1}' /etc/shadow
```

- 查看远程登录的账号

```bash
awk '/\$1|\$6/{print $1}' /etc/shadow
```

- 查看具有sudo权限的用户

```bash
cat /etc/sudoers | grep -v "^#\|^$" | grep "ALL=(ALL)"
```

- 可以使用sudo提升到root的用户（root）

```bash
cat /etc/sudoers
```

- 列出目前用户可执行与无法执行的指令

```bash
sudo -l
```

# Env

- 打印系统环境信息

```bash
env
```

- 打印系统环境信息

```bash
set
```

- 环境变量中的路径信息

```bash
echo $PATH
```

- 打印历史命令

```bash
history
~/.bash_history
```

- 打印系统的环境变量和全局的配置信息

```bash
cat /etc/profile
```

- 显示可用的shell

```bash
cat /etc/shells
```

# Process

- 查看进程信息

```bash
ps aux
```

- 资源占有情况

```bash
top -c
```

- 查看进程关联文件

```bash
lsof -c $PID
```

- 完整命令行信息

```bash
/proc/$PID/cmdline
```

- 进程的命令名

```bash
/proc/$PID/comm
```

- 进程当前工作目录的符号链接

```bash
/proc/$PID/cwd
```

- 运行程序的符号链接

```bash
/proc/$PID/exe
```

- 进程的环境变量

```bash
/proc/$PID/environ
```

- 进程打开文件的情况

```bash
/proc/$PID/fd
```

# Service

- 由inetd管理的服务列表

```bash
cat /etc/inetd.conf
```

- 由xinetd管理的服务列表

```bash
cat /etc/xinetd.conf
```

- nfs服务器的配置

```bash
cat /etc/exports
```

- 邮件信息

```bash
/var/log/mailog
```

- ssh配置

```bash
sshd_config
```

# Crontab

- 显示指定用户的计划作业（root）

```bash
crontab -l -u %user%
```

- 计划任务

```bash
# 查看开机启动服务命令
chkconfig                   
# 查看开机启动配置文件命令
ls /etc/init.d              
# 查看 rc 启动文件
cat /etc/rc.local           

/var/spool/cron/*
/var/spool/anacron/*
/etc/crontab
/etc/anacrontab
/etc/cron.*
/etc/anacrontab

ls -alh /var/spool/cron 
ls -al /etc/ | grep cron 
ls -al /etc/cron* 
cat /etc/cron* 
cat /etc/at.allow 
cat /etc/at.deny 
cat /etc/cron.allow 
cat /etc/cron.deny 
cat /etc/crontab
cat /var/spool/cron/crontabs/root
```

- 开机启动项

```bash
/etc/rc.d/init.d/
```

# Installed Programs

- Redhat

```bash
rpm -qa --last
```

- CentOS

```bash
yum list | grep installed
ls -l /etc/yum.repos.d/
```

- Debian

```bash
dpkg -l
```

- Debian APT

```bash
cat /etc/apt/sources.list
```

- xBSD

```bash
pkg_info
```

- Solaris

```bash
pkginfo
```

- Arch Linux

```bash
pacman -Q
```

- Gentoo

```bash
emerge
```

# Files

- 最近五天的文件

```bash
find / -ctime +1 -ctime -5
```

- 文件系统细节

```bash
debugfs
```

# SSH Key

```bash
~/.ssh
/etc/ssh
```

# logs

```bash
/var/log/boot.log
/var/log/cron
/var/log/faillog
/var/log/lastlog
/var/log/messages
/var/log/secure
/var/log/syslog
/var/log/syslog
/var/log/wtmp
/var/log/wtmp
/var/run/utmp
```

# Virtual Env

```bash
lsmod | grep -i "vboxsf\|vboxguest"
lsmod | grep -i "vmw_baloon\|vmxnet"
lsmod | grep -i "xen-vbd\|xen-vnif"
lsmod | grep -i "virtio_pci\|virtio_net"
lsmod | grep -i "hv_vmbus\|hv_blkvsc\|hv_netvsc\|hv_utils\|hv_storvsc"
```

# Container

```bash
capsh --print
cat /proc/1/cgroup
env | grep KUBE
ls -l .dockerenv
ls -l /run/secrets/Kubernetes.io/
mount
ps aux
```

# Keyword

```bash
grep -i user [filename] 

grep -i pass [filename] 

grep -C 5 "password" [filename] 

find . -name "*.php" -print0 | xargs -0 grep -i -n "var $password" # Joomla
```

# History

```bash
cat ~/.bash_history 

cat ~/.nano_history 

cat ~/.atftp_history 

cat ~/.mysql_history 

cat ~/.php_history
```

# Configs

```bash
cat /etc/syslog.conf

cat /etc/chttp.conf

cat /etc/lighttpd.conf

cat /etc/cups/cupsd.conf

cat /etc/inetd.conf

cat /etc/apache2/apache2.conf

cat /etc/my.conf

cat /etc/httpd/conf/httpd.conf

cat /opt/lampp/etc/httpd.conf

ls -aRl /etc/ | awk '$1 ~ /^.\*r.\*/
```

