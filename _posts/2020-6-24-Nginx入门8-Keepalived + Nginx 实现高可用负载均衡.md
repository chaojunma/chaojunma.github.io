---
layout: post
title: "Keepalived + Nginx 实现高可用负载均衡"
date: 2020-6-24
categories: 运维
tags: Nginx Keepalived
--- 


### Keepalived 简要介绍

Keepalived 是一种高性能的服务器高可用或热备解决方案， Keepalived 可以用来防止服务器单点故障的发生，通过配合 Nginx 可以实现 web 前端服务的高可用。
Keepalived 以 VRRP 协议为实现基础，用 VRRP 协议来实现高可用性(HA)。 VRRP(Virtual RouterRedundancy Protocol)协议是用于实现路由器冗余的协议， VRRP 协议将两台或多台路由器设备虚拟成一个设备，对外提供虚拟路由器 IP(一个或多个)，而在路由器组内部，如果实际拥有这个对外 IP 的路由器如果工作正常的话就是 MASTER，或者是通过算法选举产生， MASTER 实现针对虚拟路由器 IP 的各种网络功能，如 ARP 请求， ICMP，以及数据的转发等；其他设备不拥有该虚拟 IP，状态是 BACKUP，除了接收 MASTER 的VRRP 状态通告信息外，不执行对外的网络功能。当主机失效时， BACKUP 将接管原先 MASTER 的网络功能。VRRP 协议使用多播数据来传输 VRRP 数据， VRRP 数据使用特殊的虚拟源 MAC 地址发送数据而不是自身网卡的 MAC 地址， VRRP 运行时只有 MASTER 路由器定时发送 VRRP 通告信息，表示 MASTER 工作正常以及虚拟路由器 IP(组)， BACKUP 只接收 VRRP 数据，不发送数据，如果一定时间内没有接收到 MASTER 的通告信息，各 BACKUP 将宣告自己成为 MASTER，发送通告信息，重新进行 MASTER 选举状态。

### 实现效果

<div style="width:780px;height:460px;margin:50px auto">
    <img alt="keepalived.png" src="/images/keepalived.png" width="780" height="460"/>
</div>

### 环境准备

1、准备两台linux服务器，IP分别为192.168.192.10 和 192.168.192.11，在两台服务器分别安装Nginx

2、在192.168.192.10服务器上部署2个tomcat服务器，端口分别为8080和8081，每个tomcat服务的为webapps目录下分别创建weight目录，并在weight目录下创建index.html页面，内容分别如下：

```
<h1>port:8080</h1>
```

```
<h1>port:8081</h1>
```

3、在192.168.192.10服务器上修改Nginx的配置文件nginx.conf如下:

<div style="width:780px;height:350px;margin:50px auto">
    <img alt="nginx-kl1.png" src="/images/nginx-kl1.png" width="780" height="350"/>
</div>

4、在192.168.192.11服务器上修改Nginx的配置文件nginx.conf如下:

<div style="width:780px;height:350px;margin:50px auto">
    <img alt="nginx-kl2.png" src="/images/nginx-kl2.png" width="780" height="350"/>
</div>

5、在两台linux服务器安装Keepalived，安装命令如下：

```
yum install –y keepalived
```

6、修改 Keepalived 配置文件

(1) MASTER 节点配置文件（192.168.192.10）

```
vim /etc/keepalived/keepalived.conf
```

修改内容如下：

```shell
! Configuration File for keepalived
global_defs {
	router_id  LVS_DEVEL ## 标识本节点的字条串，通常为 hostname
} 

## keepalived 会定时执行脚本并对脚本执行的结果进行分析，动态调整 vrrp_instance 的优先级。如果脚本执行结果为 0，并且 weight 配置的值大于 0，则优先级相应的增加。如果脚本执行结果非 0，并且 weight配置的值小于 0，则优先级相应的减少。其他情况，维持原本配置的优先级，即配置文件中 priority 对应的值。
vrrp_script chk_nginx {
	script "/etc/keepalived/nginx_check.sh" ## 检测 nginx 状态的脚本路径
	interval 2 ## 检测时间间隔
	weight -20 ## 如果条件成立，权重-20
}

## 定义虚拟路由， VI_1 为虚拟路由的标示符，自己定义名称
vrrp_instance VI_1 {
	state MASTER   ## 主节点为 MASTER， 对应的备份节点为 BACKUP
	interface ens33  ## 绑定虚拟 IP 的网络接口，与本机 IP 地址所在的网络接口相同， 我的是 ens33
	virtual_router_id 33 ## 虚拟路由的 ID 号， 两个节点设置必须一样， 可选 IP 最后一段使用, 相同的 VRID 为一个组，他将决定多播的 MAC 地址
	mcast_src_ip 192.168.192.10 ## 本机 IP 地址
	priority 100 ## 节点优先级， 值范围 0-254， MASTER 要比 BACKUP 高
	nopreempt ## 优先级高的设置 nopreempt 解决异常恢复后再次抢占的问题
	advert_int 1 ## 组播信息发送间隔，两个节点设置必须一样， 默认 1s
	## 设置验证信息，两个节点必须一致
	authentication {
		auth_type PASS
		auth_pass 1111 ## 真实生产，按需求对应该过来
	}
	# 虚拟 IP 池, 两个节点设置必须一样
	virtual_ipaddress {
		192.168.192.130 ## 虚拟IP，可以定义多个
	}
}

```

(2)BACKUP 节点配置文件（192.168.192.11）

```
vim /etc/keepalived/keepalived.conf
```

修改内容如下：

```shell
! Configuration File for keepalived
global_defs {
	router_id LVS_DEVEL
}
vrrp_script chk_nginx {
	script "/etc/keepalived/nginx_check.sh"
	interval 2
	weight -20
}
vrrp_instance VI_1 {
	state BACKUP
	interface ens33
	virtual_router_id 33
	mcast_src_ip 192.168.192.11
	priority 90
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		192.168.192.130
	}
}
```

7、编写 Nginx 状态检测脚本

编写 Nginx 状态检测脚本 /etc/keepalived/nginx_check.sh (已在 keepalived.conf 中配置)脚本要求：如果 nginx 停止运行，尝试启动，如果无法启动则杀死本机的 keepalived 进程， keepalied将虚拟 ip 绑定到 BACKUP 机器上。 内容如下：

```shell
#!/bin/bash
A=`ps -C nginx –no-header |wc -l`
if [ $A -eq 0 ];then
 /usr/local/nginx/sbin/nginx
 sleep 2
 if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
 killall keepalived
 fi
fi
```

8、启动 Keepalived

```
service keepalived start
```


### 测试效果

在本地浏览器访问[http://192.168.192.130/weight](http://192.168.192.130/weight)轮询出现以下内容:

<div style="width:780px;height:300px;margin:50px auto">
    <img alt="keepalived-w1.png" src="/images/keepalived-w1.png" width="780" height="300"/>
</div>

<div style="width:780px;height:300px;margin:50px auto">
    <img alt="keepalived-w2.png" src="/images/keepalived-w2.png" width="780" height="300"/>
</div>

说明我们的配置生效了

- 通过虚拟IP访问可以正常访问（keepalived生效了）
- 实现了轮询负载效果