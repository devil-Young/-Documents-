# [『败类教程』美西CN2跨网也能单线程500M且0重传！手把手教你TCP调优](https://www.nodeseek.com/post-197087-1)

[![BlackSheep](https://www.nodeseek.com/avatar/15055.png)](https://www.nodeseek.com/space/15055)

[BlackSheep](https://www.nodeseek.com/space/15055)楼主

34days ago edited 18days ago in [技术](https://www.nodeseek.com/categories/tech)

## 前言：先看两个瓦工DC9单线程测速图（北方联通600M宽带）

> ## 请注意，调参后能跑多少速度是由你的网络环境和线路的优劣决定的，线路垃圾再怎么调也没用

|                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/x86tog.png) | ![image](https://image.mooncloud.top/i/2024/11/13/12dxpvc.png) |

## 一. 参数介绍

> ### net.ipv4.tcp_wmem 和 net.ipv4.tcp_rmem 都是 Linux 内核参数，用于配置 TCP/IP 协议栈中的发送缓冲区和接收缓冲区

- ## net.ipv4.tcp_wmem

  #### 这个参数控制了 TCP 发送缓冲区的大小，发送缓冲区用于存放待发送但尚未被确认的数据；当应用程序发送数据时，数据首先会被放入发送缓冲区，然后通过网络传输，如果发送缓冲区已满，发送操作将会被阻塞，直到有空间可用为止。它实际上是一个数组，包含了三个元素，分别表示最小值、默认值和最大值，格式为 min_size default_size max_size，单位为字节。

- ## net.ipv4.tcp_rmem

  #### 这个参数控制了 TCP 接收缓冲区的大小，接收缓冲区用于存放从网络中接收但尚未被应用程序读取的数据；当数据到达时，它首先会被放入接收缓冲区，然后等待应用程序读取。类似于 net.ipv4.tcp_wmem，它的格式、单位都与之相同。

## 二. 单位科普（与后续操作相关，请不要直接跳过）

- ## B是byte（字节），而b是bit（位、比特），二者是不同单位

  ***\*1B=8b\****
  ***\*1byte=8bit\****

  #### 一般而言，我们所说的50M带宽指的是50Mbps即50Mbit/s，而平时下载文件时通常显示的是MB/s，换算过来就是6.25MB/s

- ## MB和MiB并非同一单位，但长时间以来MB的混乱使用，使得二者极易发生混淆，MB常常被当作MiB来使用

  ***\*1MB=1000KB=1,000,000B\****
  ***\*1MiB=1024KiB=1,048,576B\****

  #### MB的全称是Megabyte，MiB的全称是Mebibyte；Mega是国际单位制定义的十进制前缀，1MB表示10^6个字节；而Mebi则是由国际电工委员会制定的二进制前缀，1MiB表示2^20个字节

- ## Windows系统中磁盘容量所显示的MB、GB实际上是二进制单位MiB、GiB，微软到现在也没有更正这个错误

  #### 硬盘厂商大多是按标准的十进制单位MB、GB来计算容量的，所以不少厂商的1TB硬盘在Windows系统中会显示为931GB，也有的会显示为953GB，这是因为一些厂商给了你1024GB而不是1000GB

  #### 再往深处讲的话，其实固态硬盘颗粒确实是按二进制1024生产的，但厂商仍然按十进制1000宣传容量，这是为了从中留下一部分容量用作OP（Over Provisioning）以改善SSD的性能和耐久性

## 三. iperf3测试

> ### 如果你不是root用户，请使用sudo -i或在命令前添加sudo提权

## 准备工作

#### 查看当前的 net.ipv4.tcp_wmem 参数值（最好记录一下）

```
sysctl net.ipv4.tcp_wmem
```

#### 查询当前使用的 TCP 拥塞控制算法

```
sysctl net.ipv4.tcp_congestion_control
```

#### 查看当前的队列管理算法

```
tc qdisc show
```

#### 如未启用bbr+fq，请使用以下命令开启

```
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
sysctl -p
```

## 服务端Linux安装iperf3

#### 在Debian/Ubuntu系统上安装

```
apt update
apt install iperf3
```

#### 在CentOS/RHEL系统上安装

```
yum install epel-release
yum install iperf3
```

#### 在Fedora系统上安装

```
dnf install iperf3
```

#### 防火墙放行端口 (这里以ufw为例，如果你没有开启防火墙可略过这一步)

```
ufw allow 5201
```

#### 服务端启动iperf3

```
iperf3 -s
```

## 客户端Windows安装iperf3 ([下载链接](https://www.nodeseek.com/jump?to=https%3A%2F%2Fgithub.com%2Far51an%2Fiperf3-win-builds%2Freleases%2Fdownload%2F3.17.1%2Fiperf-3.17.1-win64.zip) | 其他系统大同小异)

#### 在iperf3.exe所在路径输入cmd，回车

![image](https://image.mooncloud.top/i/2024/11/14/xrs3e1.png)

#### 在弹出的窗口里输入如下并敲击回车（此处为测试下行速度30s，想调整时间修改-t后的数值即可，想测上行就去掉-R）

```
iperf3 -c 服务端IP -R -t 30
```

## 四. 初步调整

- #### TCP缓冲区数值务必与自己的实际网络环境相契合，尽量不要使用一键脚本或者直接抄别人给的参数，否则很容易出现高重传和断流，有的脚本甚至会给你调成512MiB，也是相当夸张了

- #### 同样，DMIT默认给的参数也过大，这也导致某些用户测速会出现高重传乃至断流

> 一般Debian/Ubuntu系统的默认参数：
> net.ipv4.tcp_wmem=4096 16384 4194304
> net.ipv4.tcp_rmem=4096 87380 6291456

> DMIT的参数：
> net.ipv4.tcp_wmem = 16384 16777216 536870912
> net.ipv4.tcp_rmem = 16384 16777216 536870912

### LAX.EB用DMIT参数测速

![image](https://image.mooncloud.top/i/2024/11/14/x90vqo.png)

### 参数调整为极不合理值，还可以复现一些用户的断流现象

![image](https://image.mooncloud.top/i/2024/11/14/x9fr9w.png)

## 临时修改参数 (重启会失效)

#### 此处我们仅对服务端TCP缓冲区参数进行调整，客户端一般不需要额外改动（对于Windows用户）

```
sysctl -w net.ipv4.tcp_wmem="4096 16384 你的合理值(此处先用理论值)"
sysctl -w net.ipv4.tcp_rmem="4096 87380 同上"
```

#### 此处我们先用公式计算出一个理论值：BDP（时延带宽积）= 带宽（bps，bit/s）× 往返时延（RTT, 秒）

#### 例：本地600Mbps，小鸡1.5Gbps，RTT170ms，取瓶颈带宽600Mbps，可以求得——600×1000×1000×0.17=102000000bit，再除以8转换为12750000byte

> 请注意，***\*ping值就已经是往返时延了\****，ping的运作原理是向目标主机传出一个ICMP的请求回显数据包，并等待接收回显回应数据包

#### 为什么要计算BDP呢？BDP是在某时链路上可容纳的最大比特数，可以反应网络的理论数据承受能力

#### 但实际网络情况显然要复杂得多，而BDP仅仅是一个理论值，直接使用它时仍有概率出现高重传，抑或是没能完全发挥线路的潜力

## 五. 最终调整 (请在晚高峰进行)

## 方案一: 再次进行iperf3测试，通过测试结果直接调整参数

- #### 如果测试结果0重传或者个位数重传，可适当调大TCP缓冲区的最大值，比如上调2~4MiB；如果测得的重传数较高，则需下调1~2MiB再进行测试

- #### 0重传则上调，高重传则下调，如此反复，直到测速刚刚好能实现0重传或低重传 (100以内)，在此基础上再下调0.5MiB或1MiB追求稳定，防止因网络高峰期而再度出现高重传，此时就得到了适合你的网络环境的参数

- #### 倘若你觉得上述过程过于繁琐，并且用BDP测出来是0重传，也可直接沿用BDP的值，速度虽没达到极限但能节省不少时间

------

## 大致流程：

#### 调整为理论值后用iperf3测速发现0重传，执行下列命令（上调2~5MiB）

```
sysctl -w net.ipv4.tcp_wmem="4096 16384 理论值+3MiB"
sysctl -w net.ipv4.tcp_rmem="4096 87380 理论值+3MiB"
```

#### 再次用iperf3测速发现高重传，执行下列命令（下调1~2MiB）

```
sysctl -w net.ipv4.tcp_wmem="4096 16384 理论值+2MiB"
sysctl -w net.ipv4.tcp_rmem="4096 87380 理论值+2MiB"
```

#### 用iperf3测速发现0重传，这次可上调0.5MiB测试，发现恰好0重传，写入sysctl.conf

#### 编辑系统内核文件

```
nano /etc/sysctl.conf
```

#### 在文件中添加如下

```
net.ipv4.tcp_wmem = 4096 16384 你所测得的值
net.ipv4.tcp_rmem = 4096 87380 同上
```

#### 保存并退出

```
Ctrl+o+Enter（回车）
Ctrl+x
```

#### 使更改生效

```
sysctl -p
```

## 方案二: 将参数修改为较大值并使用Traffic Control限速（灵感来自@nhnhnh000）

> #### 其他限速工具理论上也能达到相同的效果，可自行尝试

#### 编辑系统内核文件

```
nano /etc/sysctl.conf
```

#### 在文件中添加如下（如果你觉得参数不够大可自行修改）

```
net.ipv4.tcp_wmem = 4096 16384 67108864
net.ipv4.tcp_rmem = 4096 87380 67108864
```

#### 保存并退出

```
Ctrl+o+Enter（回车）
Ctrl+x
```

#### 使更改生效

```
sysctl -p
```

#### 查看网络接口配置

```
ip a
```

> #### 与互联网通信的接口一般是eth0，如若不是请将命令中的eth0替换为相应名称

#### 删除现有的队列调度

```
tc qdisc del dev eth0 root
```

#### 添加 htb 队列调度

```
tc qdisc add dev eth0 root handle 1:0 htb default 10
```

#### 对所有源IP进行限速（XXX改为你的瓶颈带宽）

```
tc class add dev eth0 parent 1:0 classid 1:1 htb rate XXXmbit ceil XXXmbit
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip src 0.0.0.0/0 flowid 1:1
```

#### 对所有目的IP进行限速

```
tc class add dev eth0 parent 1:0 classid 1:2 htb rate XXXmbit ceil XXXmbit
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dst 0.0.0.0/0 flowid 1:2
```

#### 查看配置是否生效

```
tc qdisc show dev eth0
tc class show dev eth0
tc -s filter show dev eth0
```

#### 此时再进行iperf3测试，之后和方案一类似

> #### 0重传则上调限速值，高重传则下调，如此反复，直到测速刚刚好能实现0重传或低重传 (100以内)，在此基础上再下调20~50Mbps追求稳定，防止网络高峰期再度出现高重传

#### 将测得的限速值后写入rc.local防止重启失效

```
nano /etc/rc.local
```

#### 在文件中添加如下（YYY改为你测得的限速值）

```
#!/bin/bash
# rc.local
# 本文件将在系统启动时执行

# 在此处添加你希望开机执行的命令：
tc qdisc add dev eth0 root handle 1:0 htb default 10
tc class add dev eth0 parent 1:0 classid 1:1 htb rate YYYmbit ceil YYYmbit
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip src 0.0.0.0/0 flowid 1:1
tc class add dev eth0 parent 1:0 classid 1:2 htb rate YYYmbit ceil YYYmbit
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip dst 0.0.0.0/0 flowid 1:2

exit 0
```

#### 保存并退出

```
Ctrl+o+Enter（回车）
Ctrl+x
```

## 六. 效果展示（默认参数为Debian/Ubuntu系统默认的参数，见上文）

## 方案一：

- ### 瓦工 DC9（美西CN2GIA）

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa31ni.png) | ![内容B](https://image.mooncloud.top/i/2024/11/14/xbub99.png) |

- ### DMIT EB（美西CMIN2）

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa36tc.png) | ![内容B](https://image.mooncloud.top/i/2024/11/14/xbugs9.png) |

- ### Sharon SG（新加坡AS9929）

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa3c3u.png) | ![内容B](https://image.mooncloud.top/i/2024/11/14/xbujwb.png) |

- ### Claw HK（香港CUG）

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa3av8.png) | ![内容B](https://image.mooncloud.top/i/2024/11/14/xbutaa.png) |

- ### 瓦工DC9 ST单线程

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa3mkd.gif) | ![内容B](https://image.mooncloud.top/i/2024/11/14/10t1khi.gif) |

- ### Claw HK ST单线程

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/14/xa3pwn.gif) | ![内容B](https://image.mooncloud.top/i/2024/11/14/xbvagu.gif) |

- ### 瓦工DC9 CloudFlare ST

| ***\*默认参数\****                                           | ***\*调整参数后\****                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/15/x8e13x.png) | ![内容B](https://image.mooncloud.top/i/2024/11/15/xbums2.png) |
| ![image](https://image.mooncloud.top/i/2024/11/15/x8ebjg.png) | ![内容B](https://image.mooncloud.top/i/2024/11/15/xbukjq.png) |

## 方案二：

- ### 大参数+Traffic Control限速

| ***\*瓦工 DC9（限速550Mbps）\****                            | ***\*Sharon SG（限速650Mbps）\****                           |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](https://image.mooncloud.top/i/2024/11/18/ab7314.png) | ![内容B](https://image.mooncloud.top/i/2024/11/18/ab6n5f.png) |

## 七. 其他相关

- ### 如果你有多地区或多运营商的网络环境，请在瓶颈网络下调试（测速最低的那个）

- ### 如美西精品线路调参后无效果，请参见该文章：[部分地区运营商到美西CN2GIA及CMIN2的单线程速度存在问题](https://www.nodeseek.com/post-201967-1)

- ### 实测下来调参对iperf3、speedtest、cloudflare speedtest都有很明显的效果，但油管4K测速不会有明显提升，不过考虑到其专业性不强，就不要执着于此了吧

- ### 由于调整文中的参数就已然卓有成效，本着尽量少动内核的原则，其他参数不再进行调整