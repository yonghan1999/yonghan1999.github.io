---
layout: post
title: Git 本地仓库同时推送多个远程仓库
date: 2020-06-26
categories: 技术
tags: Git
---

### 1. 生成新的 SSH key

~~~shell
ssh-keygen -t rsa -C  'xxxxxxxx@xxx.com' -f id_rsa_tencen
~~~

### 2. 上传公钥到git仓库提供商

#### Gitee 设置账户 SSH 公钥

用户可以通过主页右上角 **「个人设置」->「安全设置」->「SSH公钥」->「[添加公钥](https://gitee.com/profile/sshkeys)」** ，添加生成的 public key 添加到当前账户中。

> 需要注意： **添加公钥需要验证用户密码**

![gitee_ssh.png](https://i.loli.net/2020/06/26/6N8PziKCtpTGWhZ.png)

#### Github 设置账户 SSH 公钥

用户可以通过主页右上角 **「头像按钮」->「Settings」->「SSH and GPG keys」->「New SSH key (or Add SSH key)」** ，添加生成的 public key 添加到当前账户中。

![github_ssh.png](https://i.loli.net/2020/06/26/REPdXyWTgs3Dp42.png)

### 配置 SSH 的 config 文件

找到 .ssh 目录下的 config 文件并编辑（如果没有 config 文件则自己创建一个）

~~~bash
cd ~/.ssh/
vim config
~~~

~~~ruby
#这里我又两个Gitee的仓库， 所以Host要自定义为不同的两个名字
#HostName 仓库的域名，如仓库在 Github 就写 github.com
#User说明该配置的用户
#IdentityFile 指定了该使用哪个ssh key文件，这里的key文件一定指的是私钥文件
Host name1	
	HostName gitee.com
	User userName
	PreferredAuthentications publickey
	PasswordAuthentication yes
	IdentityFile ~/.ssh/name1_id_rsa

Host name2
	HostName gitee.com
	User userName
	PreferredAuthentications publickey
	PasswordAuthentication yes
	IdentityFile ~/.ssh/name2_id_rsa


Host github.com
        HostName github.com
        User userName
        PreferredAuthentications publickey
        PasswordAuthentication yes
        IdentityFile ~/.ssh/id_rsa
	
~~~

完成后可以使用（三条命令分别对应 config 配置文件中的配置）

~~~bash
$ ssh -T git@name1
$ ssh -T git@name2
$ ssh -T git@github.com
~~~



### 绑定远程仓库

在创建好本地仓库和远程仓库后，我们要绑定本地仓库到多个远程仓库

~~~bash
$ git remote add origin git@name1:xxx.git
$ git remote set-url origin --add git@name2:xxx.git
$ git remote set-url origin --add git@github.com:xxx.git
~~~

然后我们推送本地代码到远程仓库

~~~bash
$ git push origin master
~~~

这时发现代码已经被推送到了撒个仓库中啦，是不是很方便呢。