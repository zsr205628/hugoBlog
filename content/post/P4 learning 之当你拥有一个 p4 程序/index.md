---
title: 'P4 learning 之当你拥有一个 p4 程序'
description: ''
date: '2022-02-28'
slug: 'p4'
image: p1.png
categories: 
    - learning
    - P4
---

## 编译 p4 程序

```
p4c -b bmv2 test.p4 -o test.bmv2
```

 `-b`指的是选择 bmv2 作为程序编译的 target，bmv2 是用来运行 p4 程序的软件交换机。该编译器会生成一个目录 test.bmv2，其中包含一个文件 test.json，该文件用于生成可由软件交换机运行的“可执行代码”。

## 创建三对虚拟以太网接口

veth 接口是一个虚拟的以太网接口，应用程序（在我们的例子中是软件交换机）可以以与真正的以太网接口相同的方式发送和接收以太网数据包。然而，在 veth 接口的情况下，数据包不会从一个真正的以太网端口出去。相反，veth 接口总是成对创建。我们将创建三对：veth0-veth1，veth2-veth3，以及veth4-veth5。当应用程序在一个 veth 接口上发送数据包时，它会到达这对接口中的另一个 veth 接口上。

 创建三对虚拟以太网接口。我们将每个接口的消息传输单元（MTU）设置为9500，以允许我们发送和接收巨量数据包。我们在每个接口上禁用IPv6，以阻止内核发送路由器请求和多播监听器报告（这并不妨碍软件交换机通过接口发送IPv6数据包）。

```
# First pair: veth0-veth1 
sudo ip link add name veth0 type veth peer name veth1 
sudo ip link set dev veth0 up sudo ip link set dev veth1 up 
sudo ip link set veth0 mtu 9500 
sudo ip link set veth1 mtu 9500 
sudo sysctl net.ipv6.conf.veth0.disable_ipv6=1 
sudo sysctl net.ipv6.conf.veth1.disable_ipv6=1 

# Second pair: veth2-veth3 
sudo ip link add name veth2 type veth peer name veth3 
sudo ip link set dev veth2 up 
sudo ip link set dev veth3 up 
sudo ip link set veth2 mtu 9500 
sudo ip link set veth3 mtu 9500 
sudo sysctl net.ipv6.conf.veth2.disable_ipv6=1 
sudo sysctl net.ipv6.conf.veth3.disable_ipv6=1 

\# Third pair: veth4-veth5 
sudo ip link add name veth4 type veth peer name veth5 
sudo ip link set dev veth4 up 
sudo ip link set dev veth5 up 
sudo ip link set veth4 mtu 9500 
sudo ip link set veth5 mtu 9500 
sudo sysctl net.ipv6.conf.veth4.disable_ipv6=1 
sudo sysctl net.ipv6.conf.veth5.disable_ipv6=1
```
## 启动软件开关，执行 p4 程序

 打开多个 SSH 会话，因此需要打开多个终端窗口或选项卡。

 在第一个 SSH 会话中，将软件交换机作为后台进程启动：

```
sudo simple_switch --interface 0@veth0 --interface 1@veth2 --interface 2@veth4 test.bmv2/test.json &
```

 在同一个 SSH 会话中，启动软件交换机的命令行界面 (CLI)：

```
simple_switch_CLI
```

 你应该得到一个 RunTimeCmd 提示：

```
Obtaining JSON from switch...
Done
Control utility for runtime P4 table manipulation
RuntimeCmd:
```

 至此我们就有了一个正在运行的P4软件交换机，有3个接口如下：

![1](https://hikingandcoding.files.wordpress.com/2019/09/getting-started-with-p4-e1568717104549.png)

 在软件交换机 CLI 中输入 help 以查看可用的命令：

```
RuntimeCmd: help

Documented commands (type help <topic>):
========================================
act_prof_add_member_to_group       set_crc16_parameters                   
act_prof_create_group              set_crc32_parameters                   
act_prof_create_member             set_queue_depth                        
act_prof_delete_group              set_queue_rate                         
act_prof_delete_member             shell                                  
act_prof_dump                      show_actions                           
act_prof_dump_group                show_ports                             
act_prof_dump_member               show_tables      
[...]
```

 show_tables 命令可以显示 p4 程序中对应的 table：

```
RuntimeCmd: show_tables
my_ingress.ipv4_match          [implementation=None, mk=ipv4.dst_addr(lpm, 32)]
tbl_test87                     [implementation=None, mk=]
```

 table_info 命令报告有关该表的更多详细信息：

```
RuntimeCmd: table_info ipv4_match
my_ingress.ipv4_match          [implementation=None, mk=ipv4.dst_addr(lpm, 32)]
********************************************************************************
my_ingress.drop_action         []
my_ingress.to_port_action      [port(9)]
```

 在软件交换机 CLI 中，可以使用 table_add 命令路由添加到路由表：

 例如：

```
add traffic to prefix 10.10.0.0/16 is sent to port 0
```

目的 ip 前缀匹配满足 10.10.0.0/16 的数据报转发至接口 0 

```
table_add ipv4_match to_port_action 10.10.0.0/16 => 0
```

表名 ：ipv4_match ；对应的动作 ：to_port_action ; 动作需要的参数 ：10.10.0.0/16 ；最终结果 ：egress_port = 0

```
RuntimeCmd: table_add ipv4_match to_port_action 10.10.0.0/16 => 0
Adding entry to lpm match table ipv4_match
match key:           LPM-0a:0a:00:00/16
action:              to_port_action
runtime data:        00:00
Entry has been added with handle 0
```

 table_dump 命令显示了我们添加到表中的条目:

```
RuntimeCmd: table_dump ipv4_match
==========
TABLE ENTRIES
**********
Dumping entry 0x0
Match key:
* ipv4.dst_addr       : LPM       0a0a0000/16
Action entry: my_ingress.to_port_action - 00
**********
Dumping entry 0x1
Match key:
* ipv4.dst_addr       : LPM       0b0b0000/16
Action entry: my_ingress.to_port_action - 01
**********
Dumping entry 0x2
Match key:
* ipv4.dst_addr       : LPM       0c0c0000/16
Action entry: my_ingress.to_port_action - 02
**********
Dumping entry 0x3
Match key:
* ipv4.dst_addr       : LPM       14141400/24
Action entry: my_ingress.drop_action - 
==========
Dumping default entry
Action entry: my_ingress.drop_action - 
==========
```

## 观察将 p4 程序执行注入软件交换机后，交换机的行为

 在一个单独的终端窗口和SSH会话中，发出以下命令来存储软件交换机在 port1 上发出的数据包（即到达接口 veth3 的数据包）：

```
sudo tcpdump -n -i veth3
```

 初始输出为：

```
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode listening on veth3, link-type EN10MB (Ethernet), capture size 262144 bytes`
```



 在 veth1 上转发一些数据包，此时需要打开另一个 ssh 会话窗口，启动 scapy：

```
sudo scapy
```

 然后，使用 scapy，向接口 1 注入一个目标 IP 地址为 11.11.1.1 的数据包：

```
INFO: Can't import matplotlib. Won't be able to plot.
INFO: Can't import PyX. Won't be able to use psdump() or pdfdump().
WARNING: No route found for IPv6 destination :: (no default route?)
INFO: Can't import python ecdsa lib. Disabled certificate manipulation tools
Welcome to Scapy (2.3.3)
>>> p = Ether()/IP(dst="11.11.1.1")/UDP()
>>> sendp(p, iface="veth1")
.
Sent 1 packets.
```

 此时通过前缀匹配，数据包会匹配至接口1及转发至 veth3 ，可以从之前使用 tcpdump 转储 的数据窗口看到：

```
10:48:52.746999 IP 172.31.41.253.53 > 11.11.1.1.53: [|domain]
```



## over！

