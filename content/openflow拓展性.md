date:2014/3/11
tags:SDN,mininet
category:Tech
title:【原】基于mininet的OpenFlow拓展性拓扑搭建

##前言
我们普遍情况下，都是单一控制器去控制一个网络。但是如果多控制器呢？我们如何实现多控制情况下的通信要求呢？

这有点难度！需要一些东西向的协议来实现控制器之间的数据共享与交换。不是我一个人一两天能搞定的事情。且不同控制器对拓扑信息等存储方式不一样，需要一个统一的标准就显得比较困难。我们先不去想太多。首先，我们需要完成的第一项工作就是数据平面上，不同控制器之间的网络可以相互通信。

最简单的通信莫过于ping了！

##实验目的

使用mininet搭建两个独立的网络，分别属于不同的控制器。同时，底层数据平面可以相互ping通。这就是我们这个实验的目的。

##拓扑搭建


底层拓扑搭建我们使用mininet2.0，因为mininet1.0版本好像没有link文件，也没有intf类。也许是我没有找到吧。

以下的代码的作用在于创建一个OVS的网络，并使虚拟机的某个（无ip）网卡连接到这个网中。

直接上代码：

	\#!/usr/bin/python
	"""
		This is a topu of our test. It shows that how to add an interface(for example a real hardware interface) to a network after the network is created.
	    This code writed by li cheng, after learning mininet of sprient's.
	"""
	import re
	from mininet.cli import CLI
	from mininet.log import setLogLevel, info,error
	from mininet.net import Mininet
	from mininet.link import Intf
	from mininet.topolib import TreeTopo
	from mininet.util import quietRun
	from mininet.node import RemoteController, OVSKernelSwitch
	def checkIntf(intf):
		#make sure intface exists and is not configured.
		if(' %s:'% intf) not in quietRun('ip link show'):
			error('Error:', intf, 'does not exist!\n' )
			exit(1)
		ips = re.findall( r'\d+\.\d+\.\d+\.\d+', quietRun( 'ifconfig ' + intf ) )
		if ips:
			error("Error:", intf, 'has an IP address,'
				'and is probably in use!\n')
			exit(1)
	if __name__ == "__main__":
		setLogLevel("info")
		OVSKernelSwitch.setup()
		intfName_1 = "eth1"  #将虚拟机的eth1赋值给intfName_1,
		info("****checking****", intfName_1, '\n')
		checkIntf(intfName_1) #检查eth1是否可用

		info("****creating network****\n")
		net = Mininet(listenPort = 6633) #监听端口是6633

		mycontroller = RemoteController("muziController", ip = "192.168.0.1")#设定远程控制器

		switch_1 = net.addSwitch('s1')#添加交换机s1
		net.controllers = [mycontroller]#添加控制器

		_intf_1 = Intf(intfName_1, node = switch_1, port = 1)#使用接口函数把虚拟机的网卡和mininet中的交换机的端口1连接

		h1 = net.addHost('h1')#创建host1
		h2 = net.addHost('h2')
		net.addLink(switch_1, h1)#将h1连接到s1上
		net.addLink(switch_1, h2)

		info("*****Adding hardware interface ", intfName_1, "to switch:" ,switch_1.name, '\n')

		net.start()#启动mininet类的对象net
		CLI(net)#启动命令行
		net.stop()

以上的代码写完之后，保存为topo.py文件，我们无需再输入什么sudo mn等命令，只需要运行它就可以了。

	sudo python topo.py

另外我们还需要在远端运行一个控制器。这个太简单，不懂的同学可以google一下。

这就完成了一个虚拟机的代码编写，另外一个虚拟机的情况同理去做就可以了。

##关键配置

在这个时候，两个mininet中的网络是无法相互ping通的。

因为还有一些关键配置我们没有做。

###host的ip配置成与虚拟机网卡同一网段的ip

**这一点需要非常注意！**因为mininet模拟出来的host默认情况下，是10网段的，而普通的虚拟机会得到的网段是192的网段。这两个网段并不一样，所以host无法ping通虚拟机网卡也是很正常的，即使物理上做了连接。

所以我们需要对host进行ip配置

在mininet之下：

	h1 ifconfig h1-eth0 192.168.0.6#ip只要是同一网段就可以

最好我们需要把所有的host的ip都修改成192网段的。所以模拟的时候建议host数量不用过多。也许mininet中可以直接指定所有host的ip网段，我还没有研究到，如果你知道，你可以告诉我！相互交流学习！

这个时候你会发现还是没有ping通！我们需要进行下一步。

###  将网卡设为桥接模式，并全部允许混杂，同时不给网卡分配IP。

* 桥接模式能确保虚拟机能和主机通信，并通过主机网卡与两一个虚拟机网卡连接。

* 混杂模式之下会把目的地址不是自己的数据接受并转发出去。

* 不给网卡进行dhclient服务，则网卡没有配置ip，从而作为一个二层原件去转发数据。


当两台虚拟机都完成了以上的工作的时候，相信两个mininet网络之间的host是可以互ping的！好玩吧！

##后续

多控制器的沟通，其实更重要的是控制器之间的信息共享，但是那还比较难，以后做出来再说吧。谢谢浏览！



