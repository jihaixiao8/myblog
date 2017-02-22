---
layout:     post
title:      "CentOS7管理防火墙和打开端口"
subtitle:   " \"打开关闭防火墙 \""
date:       2017-02-22 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Linux
---

## 正文

### 1：安装iptables-service，打开防火墙

刚刚初始化的CentOS系统里面，在/etc/sysconfig目录下面是没有iptables文件的，所以想往里面写入需要先把它初始化出来。

如果你在未初始化之前直接使用这个命令，打开防火墙：

```xml
service iptables start
```

就会报错：Failed to restart iptables.service: Unit iptables.service failed to load: No such file or directory.

所以在想使用这个命令之前，先安装下iptables-services

1. ```
   yum install iptables-services
   ```

2. 设置开机启动：

   ```
   systemctl enable iptables
   ```

3. 这时候service iptables命令就可以正常使用了

   ```
   service iptables [stop|start|restart]
   ```

4. 使用save命令，即可在/etc/sysconfig命令下初始化iptables文件了

   ```
   service iptables save
   ```

### 2：打开某个固定端口

防火墙开启后，如果想开启某个端口，例如80端口（nginx默认端口）

使用如下命令可以打开，打开后可以使用telnet测试端口连通性，或者浏览器访问

打开某个固定端口命令如下：

```
sbin/iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

### 3：CentOS7下打开关闭防火墙

CentOS7下还可以使用如下命令打开，关闭，禁用防火墙

```
systemctl [start][stop][mask] firewalld
```

