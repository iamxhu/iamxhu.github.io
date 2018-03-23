---
layout: post
title:  文件权限错误导致SSH无法登录
date: 2018-03-23 11:10:23.000000000 +09:00
tags: Linux
---

###缘起
2015年帮一个朋友在Linode买了个空间，主要作用是帮他部署了一个企业官方网站。前两天碰到一个问题，他们的技术人员说重启系统后，服务器无法登录了，网站也出问题了。最终花了2天时间，咨询了一下Linode的技术支持弄清楚怎么进入secure mode，最终发现是因为/etc目录下配置文件的权限问题导致了SSH无法登录。 

### 发现问题
接到问题后，首先怀疑是不是防火墙的问题，因为之前出现过一次这种问题。于是登录Linode，试图通过管理界面提供的Lish登录到系统，检查防火墙设置。结果发现情况比想像地糟糕，Lish也无法通过root登录系统。

几番尝试后，在Linode的Support系统里面提了个Ticket，Supporter建议通过进行secure mode，检查一下是否是/etc/securetty文件限制了Lish终端的登录，并给了进入secure mode的说明。于是按他提供的方式进入了secure mode，检查/etc/securetty文件，发现里面已经包含了Lish的ttySO。在安全模式下尝试ssh登录，发现可以登录到系统，于是reboot，结果仍然无法登录。

在Google搜索了一上 “root无法ssh登录系统”，Stackoverflow上很多人建议检查/etc/ssh/sshd_config文件，看看下面的配置项是否为yes。
>PermitRootLogin yes

查看后，这一项配置是正常的，一时限入了困境。 

### 找到原因
这时想起来应该看看/var/log/下面的日志，于是打开/var/log/auth.log，发现里面很多报错:

>Mar 21 08:11:58 xx sshd[3819]: error: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Mar 21 08:11:58 xx sshd[3819]: error: @         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
Mar 21 08:11:58 xx sshd[3819]: error: @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Mar 21 08:11:58 xx sshd[3819]: error: Permissions 0777 for '/etc/ssh/ssh_host_ed25519_key' are too open.
Mar 21 08:11:58 xx sshd[3819]: error: It is required that your private key files are NOT accessible by others.
Mar 21 08:11:58 xx sshd[3819]: error: This private key will be ignored.
Mar 21 08:11:58 xx sshd[3819]: error: bad permissions: ignore key: /etc/ssh/ssh_host_ed25519_key
Mar 21 08:11:58 xx sshd[3819]: error: Could not load host key: /etc/ssh/ssh_host_ed25519_key

于是在google了一下原因，发现是权限设置太宽松，导致出错。 于是修改了这些key文件的权限，重启系统后，发现仍然无法登录。

继续看重启后/var/log/auth.log文件，发现日志中多了下面的行
>Mar 21 08:13:27 xx login[3648]: pam_securetty(login:auth): /etc/securetty is either world writable or not a normal file

很明显，也是因为文件权限引起的问题，于是将该文件权限改成644，再次重启，终于可以ssh登录了。 

继续解决网站的问题，发现网站无法访问是因为MySQL没有启动起来导致的，于是看/var/log/syslog日志，发现如下行日志:
>Mar 22 02:38:39 xx /etc/mysql/debian-start[4168]: Warning: World-writable config file '/etc/mysql/my.cnf' is ignored

又一个因为权限导致无法启动的问题，修改文件权限后，重启MySQL，网站可以正常访问了。

经过前后2天，断断续续的排查，问题最终解决。在这个过程中，通过不断尝试找到问题的原因，学习到了Linux控制root登录的机制。下面对此做个梳理：

1. 首先/etc/securetty 配置确定了哪些类型的终端允许以root的方式登录系统，如果没有这个文件，则所有终端都能登录。
2. sshd服务本身也提供了控制root登录的配置项，即/etc/ssh/sshd_config 文件中的 PermitRootLogin 参数。

最后，/etc下面的配置文件的权限非常重要！杜绝使用777权限。



