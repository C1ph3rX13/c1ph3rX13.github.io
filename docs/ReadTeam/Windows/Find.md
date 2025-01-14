## 探测存活主机

### Ping

#### 探测存活主机

```cmd
for /L %I in (1,1,254) DO @ping -w 1 -n 1 192.168.0.%I |findstr "TTL="
```

#### 查询域名对应内网IP

```cmd
for /f "delims=" %i in (D:/domains.txt) do @ping -w 1 -n 1 %i | findstr /c:"[192." >> c:/windows/temp/ds.txt
```

### NC

#### 探测端口

批量探测端口

```cmd
nc.exe -znv 192.168.0.98 1-65535
```

- `-z` 表示执行扫描操作，但不发送任何实际数据
- `-n` 表示禁止解析主机名，而直接使用 IP 地址
- `-v` 表示启用详细输出模式，显示更多信息

在 192.168.0.110 主机上扫描端口范围 1-1000，尝试连接这些端口，但不发送任何实际数据。通过此命令，可以快速确定目标主机上哪些端口处于开放状态或关闭状态

```cmd
nc.exe -v -w 1 192.168.0.110 -z 1-1000
```

- `-v`：启用详细输出模式，显示更多信息
- `-w 1`：设置超时时间为 1 秒，表示在 1 秒后如果没有连接成功就放弃
- `192.168.0.110`：目标 IP 地址，即要连接的主机地址
- `-z`：指定 `nc` 命令执行零字节传输，只用于扫描连接是否成功，不发送实际数据
- `1-1000`：要扫描的端口范围，这里是从端口 1 扫描到端口 1000

在多个目标主机上扫描特定的端口范围（21-25）。通过使用循环结构，可以依次扫描不同的主机并显示扫描结果

```cmd
for i in {101..102}; do nc.exe -vv -n -w 1 192.168.0.$i 21-25 -z; done
```

- `for i in {101..102};`：这是一个 for 循环，迭代变量 `i` 的范围是从 101 到 102
- `nc -vv -n -w 1 192.168.0.$i 21-25 -z;`：循环体内的命令，与第一个命令类似，但是用 `$i` 替代了 IP 地址的最后一位。这样，循环将分别尝试连接 192.168.0.101 和 192.168.0.102 主机上的端口范围 21-25

## 探测端口&服务

### 常见端口

| 服务       | 端口                            |
| :--------- | :------------------------------ |
| Mssql      | 1433                            |
| SMB        | 445                             |
| WMI        | 135                             |
| winrm      | 5985                            |
| rdp        | 3389                            |
| ssh        | 22                              |
| oracle     | 1521                            |
| mysql      | 3306                            |
| redis      | 6379                            |
| postgresql | 5432                            |
| ldap       | 389                             |
| smtp       | 25                              |
| pop3       | 110                             |
| imap       | 143                             |
| exchange   | 443                             |
| vnc        | 5900                            |
| ftp        | 21                              |
| rsync      | 873                             |
| mongodb    | 27017                           |
| telnet     | 23                              |
| svn        | 3690                            |
| java rmi   | 1099                            |
| couchdb    | 5984                            |
| pcanywhere | 5632                            |
| web        | 80-90,8000-10000,7001,9200,9300 |