title: 用搬瓦工搭建vpn
date: 2016-08-06 22:36:05
tags: [VPN]
---
刚刚用上了自己搭建的VPN，兴奋之余想记录一下过程。

## 步骤一：

首先先去[搬瓦工](https://bandwagonhost.com/index.php)购买 VPS 主机先，有优惠码的，记得搜一下，或者查看主要源码里面有一个优惠码，可以优惠5%左右。

我用的是 centos-6x86_64 系统，

进入到 KiwiVM -> Root shell - advanced ，先运行命令

	yum update
	
更新一下，然后 KiwiVM Extras -> Shadowsocks Server 一键安装 Shadowsocks Servers。

<!-- more -->

## 步骤二：

接下来，终端 ssh 到你的vps主机，

	ssh -p 端口号 root@xx.xx.xx.xx

第一步：先加入 yum源： 

		rpm -Uvh http://poptop.sourceforge.net/yum/stable/rhel6/pptp-release-current.noarch.rpm 
		
然后用yum安装pptpd：
 
		yum install pptpd 
		
第二步：修改配置文件  

1.配置文件/etc/ppp/options.pptpd 末尾添加下面两行 

		ms-dns 8.8.8.8 
		ms-dns 8.8.4.4 

2.配置文件/etc/ppp/chap-secrets 添加用户和密码 
chap-secrets内容如下： 

		#Secrets for authentication using CHAP 
		#client server secret IP addresses 
		
默认有一行类似于“vpn pptpd cdt6dafc *” 

//vpn是你的vpn帐号，cdt6dafc是你的vpn的密码，*表示对任何ip，记得不要丢了这个星号。
 
复制这一行然后把用户名密码修改成你自己的。

如想建多个帐号就多复制粘贴几次并修改用户名。密码可以用一样的。

文档最后有一行空行记得一定保留。 

3.配置文件/etc/pptpd.conf 把最后两行的  # 去掉改成下面的样子并保留最后的空行。 如果没有直接添加下面两行。 

		localip 192.168.0.1 
		remoteip 192.168.0.234-238,192.168.0.245 
 
4.配置文件/etc/sysctl.conf 

将 net.ipv4.ip_forward = 0 改成 net.ipv4.ip_forward = 1 

第三步：启动 pptp vpn服务和 iptables  

		/sbin/service iptables start //启动iptables 
		iptables -t nat -A POSTROUTING    -s 192.168.0.0/24 -j SNAT --to-source xxx.xxx.xxx.xxx  

你需要将xxx.xxx.xxx.xxx替换成你的vps的公网ip地址 。注意to前面是两个短横杠。

		/etc/init.d/iptables save //保存iptables的转发规则 
		/sbin/service iptables restart //重新启动iptables 
		service pptpd restart 
		chkconfig pptpd on //开机启动pptp vpn服务 
		chkconfig iptables on //开机启动iptables 

参考链接：

[搬瓦工搭建VPN教程](https://plus.google.com/+siyuli/posts/eRMcuQPx8GU)（基本上是参照这个的，推荐看这个吧。）

[搬瓦工VPS架设VPN教程](http://blog.csdn.net/hjhjw1991/article/details/45848565/)（这个试过了，搭建成功连上VPN却上不了网）