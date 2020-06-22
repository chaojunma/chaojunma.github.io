---
layout: post
title: "CentOS7安装Nginx"
date: 2020-6-22
categories: 运维
tags: CentOS Nginx
--- 


安装所需插件
1、安装gcc
gcc是linux下的编译器在此不多做解释，感兴趣的小伙伴可以去查一下相关资料，它可以编译 C,C++,Ada,Object C和Java等语言

命令：查看gcc版本 

```
gcc -v
```

<div style="width:780px;height:250px;margin:50px auto">
    <img alt="gcc.png" src="/images/gcc.png" width="780" height="250"/>
</div>


一般阿里云的centOS7里面是都有的，没有安装的话会提示命令找不到，

安装命令:

```
yum -y install gcc
```



2、pcre、pcre-devel安装
pcre是一个perl库，包括perl兼容的正则表达式库，nginx的http模块使用pcre来解析正则表达式，所以需要安装pcre库。

安装命令：

```
yum install -y pcre pcre-devel
```


3、zlib安装
zlib库提供了很多种压缩和解压缩方式nginx使用zlib对http包的内容进行gzip，所以需要安装

安装命令：

```
yum install -y zlib zlib-devel
```

4、安装openssl
openssl是web安全通信的基石，没有openssl，可以说我们的信息都是在裸奔。。。。。。

安装命令：

yum install -y openssl openssl-devel
 

安装nginx
1、下载nginx安装包

```
wget http://nginx.org/download/nginx-1.12.2.tar.gz  
```

2、把压缩包解压到/usr/local/src

```
tar -zxvf  nginx-1.12.2.tar.gz
```

3、切换到cd /usr/local/src/nginx-1.12.2/下面
执行三个命令：

```
./configure
```
 
```
make
```

```
make install
```

5、切换到/usr/local/nginx安装目录

<div style="width:773px;height:327px;margin:50px auto">
    <img alt="nginx-l.png" src="/images/nginx-l.png" width="773" height="327"/>
</div>

6、配置nginx的配置文件nginx.conf文件，主要也就是端口

<div style="width:773px;height:450px;margin:50px auto">
    <img alt="nginx-p.png" src="/images/nginx-p.png" width="773" height="450"/>
</div>


可以按照自己服务器的端口使用情况来进行配置

ESC键，wq！强制保存并退出

7、启动nginx服务
切换目录到/usr/local/nginx/sbin下面



启动nginx命令：

```
./nginx
```

8、查看nginx服务是否启动成功

```
ps -ef | grep nginx
```

9、开放端口

在 windows 系统中访问 linux 中 nginx，默认不能访问的，因为防火墙问题

查看开放的端口号

```
firewall-cmd --list-all
```

设置开放的端口号

```
firewall-cmd --add-service=http --permanent  #永久开放http
firewall-cmd --add-port=80/tcp --permanent   #永久开放80端口
```

重启防火墙

```
firewall-cmd --reload
```

10、访问你的服务器IP
显示

<div style="width:780px;height:450px;margin:50px auto">
    <img alt="nginx-w.png" src="/images/nginx-w.png" width="780" height="450"/>
</div>

说明安装和配置都没问题OK了
