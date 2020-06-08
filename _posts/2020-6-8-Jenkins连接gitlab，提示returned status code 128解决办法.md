---
layout: post
title: "Jenkins连接gitlab，提示returned status code 128解决办法"
date: 2020-6-8
categories: 运维
tags: Jenkins Git
--- 


在项目中配置git仓库地址，报无权限

  Failed to connect to repository : Command "D:\Program Files\Git\mingw64\bin\git.exe ls-remote -h -- http://ip/test/APP-Test.git HEAD" returned status code 128: stdout:


<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-gitlab.png" src="/images/jenkins-gitlab.png" width="780" height="410"/>
</div>
 

 

 我直接从工程里配置credentials，配置Username with password后，凭据选择该配置还是报没有权限，不知道哪里搞错了；后来在网上看到这个git令牌配置，就试了下，

 还是报无权限，可能需要项目里还需要单独配置啥了。后来通过凭据-系统管理-全局凭据，添加了Username with password，设置成git的账户和密码，竟然不报异常了，不知道为啥，总之解决了就好，附解决步骤。

解决步骤如下：

1、登陆gitlab，在用户头像下拉图标，选择“Setting”

    
<div style="width:332px;height:445px;margin:50px auto">
    <img alt="gitlab-u-set.png" src="/images/gitlab-u-set.png" width="332" height="445"/>
</div>
 

2、添加个人访问令牌：

 <div style="width:780px;height:410px;margin:50px auto">
    <img alt="gitlab-token.png" src="/images/gitlab-token.pn" width="780" height="410"/>
</div>
 

3、点击创建后，提示个人令牌，一定要先保存好，一刷新页面就没了

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="gitlab-t.png" src="/images/gitlab-t.png" width="780" height="410"/>
</div>
 

这里可以撤销，删除token，如果该token已经关联到Jenkins，要删除该token，要连带删除Jenkins里的GitLab API token，否则Jenkins里的GitLab API token失效不能用

 <div style="width:780px;height:410px;margin:50px auto">
    <img alt="token.png" src="/images/token.png" width="780" height="410"/>
</div>


4、在Jekins插件管理中安装`GitLab Plugin`插件
   
<div style="width:780px;height:410px;margin:50px auto">
    <img alt="gitlab-plugin.png" src="/images/gitlab-plugin.png" width="780" height="410"/>
</div>

 5、在“Jenkins管理”-“系统管理“”，配置gitlab
 
 <div style="width:780px;height:410px;margin:50px auto">
    <img alt="gitlab-set.png" src="/images/gitlab-set.png" width="780" height="410"/>
</div>

 6、添加Credentials，选择GitLab API token，输入从git服务器获取的token
 
 <div style="width:780px;height:410px;margin:50px auto">
    <img alt="Credentials.png" src="/images/Credentials.png" width="780" height="410"/>
</div>
 
 7、添加完，在Credentials选择GitLabAPItoken，点击test Connection
 
 8、创建任务时配置git地址及账号，如下：

<div style="width:780px;height:410px;margin:50px auto">
    <img alt="jenkins-git.png" src="/images/jenkins-git.png" width="780" height="410"/>
</div>