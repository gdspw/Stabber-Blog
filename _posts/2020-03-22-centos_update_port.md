layout: post
title:  "更改centos系统ssh连接端口号"
categories: it
tags: [Centos,系统]
excerpt: 一次登录个人服务器，发现“There were 113773 failed login attempts since the last successful login”，这尼玛是服务器被黑的前兆啊，赶紧改下服务器ssh连接端口压压惊...

由于本人平时爱折腾，之前趁着阿里云和腾讯云搞活动期间分别搞了台服务器，刚开始的时候，，服务器的防护措施没有做好，陆陆续续服务器被黑了好几次，最惨的一次是搭建的知识库的服务器被黑，导致所有的数据丢失，黑客在服务器留言需要0.25个比特币赎回数据，当时的内心真实哔了狗了。鉴于之前的惨痛经历，本次登录服务器突然看到“There were 113773 failed login attempts since the last successful login”的提示，内心真实咯噔了一下，以为我的重新搭建的知识库还在这台机器上，当时系统重装后忘了最关键也是最有效的措施：修改ssh连接端口号。

我们都知道，centos默认的ssh连接端口号是22，鉴于很多人刚开始拥有自己服务器的时候，都不会在意这个事情，毕竟平时在公司的时候，这种服务器安全的问题都是运维负责的，当自己独立维护对接公网的服务器的时候很容易被别人黑掉。下面就是讲解如何修改ssh连接的默认端口。

## 1、修改ssh配置文件sshd_config

> [root@VM_0_10_centos ~]# vi /etc/ssh/sshd_config

![jDrdef](http://image.itstabber.com/uPic/jDrdef.png)

截图中红色框中的部分，建议先保留22端口号，然后新增自定义端口号

## 2、防火墙放行

>[root@VM_0_10_centos ~]# firewall-cmd --zone=public --add-port=10010/tcp --permanent
>
>[root@VM_0_10_centos ~]# firewall-cmd --reload



## 3、向SELinux中添加修改的SSH端口

### 3.1 先安装SELinux的管理工具 semanage (如果已经安装了就直接到下一步) 

>[root@VM_0_10_centos ~]# yum provides semanage

### 3.2 安装运行semanage所需依赖工具包 policycoreutils-python：

> [root@VM_0_10_centos ~]# yum -y install policycoreutils-python

### 3.3 查询当前 ssh 服务端口：

>[root@VM_0_10_centos ~]# semanage port -l | grep ssh

![aDG4rJ](http://image.itstabber.com/uPic/aDG4rJ.png)

### 3.4 向 SELinux 中添加 ssh 端口：

>[root@VM_0_10_centos ~]# semanage port -a -t ssh_port_t -p tcp 10010

### 3.5 重启 ssh 服务：

>[root@VM_0_10_centos ~]# systemctl restart sshd.service

测试成功后，把22端口注释掉即可

PS: 有些同学操作完后，新增的端口可能不生效，一般很多人的服务器都是阿里云或者腾讯云，这个时候需要到对应的控制台查看下网络完全组配置，有些人的安全组可能设置了外放访问的端口号，这个时候将需要本次ssh更改的连接端口号加上即可。