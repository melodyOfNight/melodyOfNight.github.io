title: git 管理多个ssh key
date: 2016-07-08 22:08:02
categories: git
tags: git
---
先阐述一下背景和需求：

# 背景：

私人用的 github 有 2 个账号，一个是旧的 github 账号，star 了很多开源库；旧的账号名不好听，新申请了一个 github 账号用于搭建 hexo 博客；再加上公司的 gitlab 账号，总共有 3 个 git 账号。

<!-- more -->

# 需求：

现在想生成并管理 3 个 git 账号的 ssh key 。

明确了需求之后，以下是

# 解决方法：
## 步骤

**1.** 先查看你本地的 **~/.ssh/** 目录下有没有 id_rsa 和 id_rsa.pub 这两个 ssh key 。有的话先删掉，待会儿会重新生成并规范命名。

		ls -al ~/.ssh
		//如果有 id_rsa 和 id_rsa.pub 的话，删除它
		rm -rf ~/.ssh/id_rsa
		rm -rf ~/.ssh/id_rsa.pub

**2. **现在重新生成 github 账号的 ssh key。下面以我的 github 账号 melodyOfNight 为例：

 *(1)* 先 cd 到 **~/.ssh** 目录，然后执行下面命令生成 ssh key。

		ssh-keygen -t rsa -C "aaron.zheng.dev@gmail.com"
		
第一个回车的时候先别急着往下走，它会提示你输入要保存的 ssh key 的名字，输入一个你自定义的名字，比如这里的 id_rsa_github_melodyOfNight ，接下来的输入密码和确认密码提示就直接回车就行了，这样就在 **~/.ssh/** 目录下生成了 id_rsa_github_melodyOfNight 和 id_rsa_github_melodyOfNight.pub 这两个 key 了，一个是私有的，一个是公有的。

 *(2)* 把你的 ssh key 添加到 ssh-agent

		# ensure ssh-agent is enable ! 
		# start the ssh-agent in the background
		$ eval "$(ssh-agent -s)"
		Agent pid 59566  

 .

		$ ssh-add ~/.ssh/id_rsa_github_melodyOfNight
	
**3.** 把你的 ssh key 公钥添加到 github 账号上 。不懂参照[这里](https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/)喽

**4.** 按照 1、2、3 的方法生成另外 2 个账号的 ssh key ,主要命名要规范且容易区分。

**5.** 创建并修改 config 文件

		$ vi config

添加如下内容：

		#kugou gitlab
		Host http://10.16.1.16/  
		HostName http://10.16.1.16/   //你公司内部的 gitlab 地址
		PreferredAuthentications publickey
		IdentityFile ~/.ssh/id_rsa_kugou
		User aaronzheng
		
		#github chaoyua899
		Host github.com
		HostName github.com
		PreferredAuthentications publickey
		IdentityFile ~/.ssh/id_rsa_github_chaoyuan899
		User chaoyuan899
		
		#github melodyOfNight
		Host github.com
		HostName github.com
		PreferredAuthentications publickey
		IdentityFile ~/.ssh/id_rsa_github_melodyOfNight
		User melodyOfNight
**6.** 测试

		$ ssh -T git@github.com              
		Hi melodyOfNight! You've successfully authenticated, but GitHub does not provide shell access.

Note:如果到这里你没有成功的话，别急，教你解决问题的终极办法--debug

比如测试github，

	ssh -vT git@github.com

-v 是输出编译信息，然后根据编译信息自己去解决问题吧。就我自己来说一般是config里的host那块写错了。

**7.** 最后关键的一点，如果之前你有设置过全局 git 用户名和 email 的话，现在要重置一下，要不然进行 git 操作的话会出错。

		git config --global --unset user.name
		git config --global --unset user.email
		
然后在不同的仓库下设置局部的用户名和邮箱

比如在公司的repository下

		git config user.name "yourname"  
		git config user.email "youremail" 
		
在自己的github的仓库在执行刚刚的命令一遍即可。

这样就可以在不同的仓库，以不同的账号登录。 
  


参考链接：

- github 官方给出的 [generating-an-ssh-key](https://help.github.com/articles/generating-an-ssh-key/)
