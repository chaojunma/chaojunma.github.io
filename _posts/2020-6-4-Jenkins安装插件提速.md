---
layout: post
title: "Jenkins安装插件提速"
date: 2020-6-4
categories: 运维
tags: Jenkins
--- 


看到好多加速Jenkins安装插件速度的文章，大多数教程中都是在插件配置里使用

https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

替换原来的官方的json，我们来看看清华源拉下来的是什么 这里使用官方的下载插件的url全局搜索



这里我们发现，每个插件下载路径依旧没有改变，变的只是这个json是从清华源下来的，其内写死的插件下载地址是没有变的，还是从官网下载！

所以无论是更换还是没更换镜像json，下载插件的速度其实是没有变的！这真是令人心痛！💔​

所以本文的目的在于：真正加速Jenkins安装插件的速度，减少失败率！

操作步骤
以上的配置Json其实在Jenkins的工作目录中

```
$ cd {你的Jenkins工作目录}/updates  #进入更新配置位置
```
第一种方式：使用vim
```
$ vim default.json   #这个Json文件与上边的配置文件是相同的
```
这里wiki和github的文档不用改，我们就可以成功修改这个配置

使用vim的命令，如下，替换所有插件下载的url

```
:1,$s/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g
```

替换连接测试url

```
:1,$s/http:\/\/www.google.com/https:\/\/www.baidu.com/g
```


`进入vim先输入：然后再粘贴上边的：后边的命令，注意不要写两个冒号！`

修改完成保存退出:wq

第二种方式：使用sed
```
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

这是直接修改的配置文件，如果前边Jenkins用sudo启动的话，那么这里的两个sed前均需要加上sudo

重启Jenkins，安装插件试试，简直超速！！