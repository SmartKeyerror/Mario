---
layout: post
cover: false
navigation: true
title: Linux主机通过Windows虚拟机转发Easyconnect内网请求
date: 2020-02-27 12:06:25
tags: ['Tool']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

世界上有个恶心的公司叫Sangfor，开发出了恶心的工具EasyConnect，本来这东西都是给在校的学生用的，好不好用都无所谓。但是很多公司也开始使用这个来访问内网，并且还不支持Linux(反正到目前Ubuntu下的64bit版本连接就没成功过)，这就很令人讨厌了。回想起Ubuntu下使用Wine安装微信的种种难受，决定还是使用Windows虚拟机开启EasyConnect，并把部分的Linux流量打进虚拟机。

<!---more--->

#### 1. 虚拟机配置
虚拟机需要两块网卡，其中一块必须是Host-only类型的，用来和Linux主机通信，这块儿网卡是内网流量的入口，所以必须配置。另一块可以是NAT，也可以是Bridge，随个人喜好。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/tool/proxy/VMware-Network-Config.png)

#### 2. Windows虚拟机网络配置

进到Windows虚拟机以后，打开网络适配器设置，这时候可以看到两张网卡: Ethernet0和Ethernet1，找到Host-only那张网卡，并修改其名称为`Host`，起一个有意义的名称对后面的操作非常有帮助。如果不确定哪张网卡是Host-only的话，可以对比网卡的MAC地址和虚拟机网络配置中的MAC地址。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/tool/proxy/Ethernet.png)

开启EasyConnect，并建立与内网的连接，此时会多出一张名为"以太网"的网卡，这张网卡的IP地址子网掩码什么的都可以不用管，就是张工具卡。设置该网络的共享属性，和前面建立的Host-only网卡建立共享。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/tool/proxy/share-with-host-only.png)

设置完成以后会发现Host-only那张网卡的IP地址被强制更改成了192.168.137.1，改了就改了吧，也不是不能用。

并关闭Windows的公用网络防火墙，不然数据包会被防火墙拦截。

#### 3. 设置Linux路由规则

```bash
sudo ip a add 192.168.137.2/24 dev vmnet1
sudo ip r add 10.0.0.0/8 via 192.168.137.1
```

给vmnet1，也就是Host-only网卡添加一个和192.168.137.2相同IP地址段的IP地址，使Linux能够建立正常的路由转发规则，否则Linux主机根本就不知道192.168.137.1这个IP地址是谁的。然后将所有的内网(10.0.0.0/8)请求都通过192.168.137.1走Windows的内网网络。


#### 4. Windows安装CCProxy

对于某些需要域名解析的HTTP请求，又不想用dnsmasq去自己解析DNS，就可以直接用该工具进行转发，下载、安装和使用傻瓜式操作，完美，非常适合我这种智商不高的人。

安装完成以后设置下本机局域网IP地址，选择NAT或者是Bridge的那张网卡即可:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/tool/proxy/CCProxy.png)

由于我的Chrome用了SwitchOmega，并不想牺牲某些便利，所以使用Firefox进行代理设置，内网的HTTP请求以后就用Firefox发送，流量使用分明，代理配置也尽量分开。

Firefox中输入`about:preferences`，拉到最底下，选择Network Settings，就可以进行代理设置了。

CCProxy的HTTP协议默认端口为808，IP地址则是NAT或者Bridge网卡的IP地址，进行配置即可:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/tool/proxy/Firefox-Config.png)

需要注意的是FTP端口为2121，SOCKS端口为1080，因为我不需要这两个协议，所以并没有配置。

完)