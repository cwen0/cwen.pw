title: TC - Linux 流量控制工具
author: cwen
date:  2018-05-23
update:  2018-05-23
tags:
    - Linux
    - network
    - 网络
    - 工具

---
最近一个新任务，需要模拟限制主机带宽的场景，之前只是知道使用 `tc` 是可以做到模拟网络延迟、网络丢包等情况，所有就想应该 `tc` 也可以搞定限制带宽的情况，就在网上搜了一波，果不其然 `tc` 还是强大，可是对于用来限制带宽，操作起来还是很蛋疼，踩了不少坑.. <!--more-->

## 简单说一下原理

本来打算直接列一波用法，但是总觉得，不记录一下原理，操作起来也是一脸懵逼。
TC 通过建立处理数据包队列，并定义队列中数据包被发送的方式，从而实现进行流量控制。TC 模拟实现流量控制功能使用的队列分为两类：

* 无类队列（clssless）: 使用 classless 队列，TC 对进入网卡的流量不加任何区分，统一对待, 常用的无类队列有 **pfifo _fast (先进现出) 、TBF ( 令牌桶过滤器) 、SFQ(随机公平队列) 、ID (前 向随机丢包)** 等等。这类队列规定使用的流量整形手段主要 是排序、限速和丢包。
    * fifo, 使用最简单的 qdisc（排队规则），纯粹的先进先出。只有一个参数：limit ，用来设置队列的长度，pfifo 是以数据包的个数为单位；bfifo 是以字节数为单位。
    * pfifo_fast，在编译内核时，如果打开了高级路由器（Advanced Router）编译选项，pfifo_fast 就是系统的标准 qdisc(排队规则)。它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。而三个波段（band）的优先级也不相同，band 0 的优先级最高，band 2 的最低。如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包，band 1 和 band 2 之间也是一样的。数据包是按照服务类型（Type Of Service，TOS ）被分配到三个波段（band）里面的。
    * red，red 是 Random Early Detection（随机早期探测）的简写。如果使用这种 qdsic，当带宽的占用接近与规定的带宽时，系统会随机的丢弃一些数据包。他非常适合高带宽的应用。
    * sfq，sfq 是 Stochastic Fairness Queueing 的简写。它会按照会话（session -- 对应与每个 TCP 连接或者 UDP 流）为流量进行排序，然后循环发送每个会话的数据包。
    * tbf，tbf 是 Token Bucket Filter 的简写，适用于把流速降低到某个值。
* 分类队列（classful）：当使用 classful 队列时，TC 对进行的网络设备的数据包根据不同的需求进行分类处理。TC 使用过滤器进行分类，TC 根据过滤器定义的规则匹配相应的数据包，发送到对应的分类队列，同时我们可以对子类在次使用过滤器进行分类。理论上这可以是一个无限深度的树。
    * CBQ，CBQ 是 Class Based Queueing（基于类别排队）的缩写。它实现了一个丰富的连接共享类别结构，既有限制（shaping）带宽的能力，也具有带宽优先级别管理的能力。带宽限制是通过计算连接的空闲时间完成的。空闲时间的计算标准是数据包离队事件的频率和下层连接（数据链路层）的带宽。
    * HTB，HTB 是 Hierarchy Token Bucket 的缩写。通过在实践基础上的改进，它实现一个丰富的连接共享类别体系。使用 HTB 可以很容易地保证每个类别的带宽，虽然它也允许特定的类可以突破带宽上限，占用别的类的带宽。HTB 可以通过 TBF（Token Bucket Filter）实现带宽限制，也能够划分类别的优先级。
    * PRIO，PRIO qdisc 不能限制带宽，因为属于不同类别的数据包是顺序离队的。使用 PRIO qdisc 可以很容易对流量进行优先级管理，只有属于高优先级类别的数据包全部发送完毕，参会发送属于低优先级类别的数据包。为了方便管理，需要使用 iptables 或者 ipchains 处理数据包的服务类型（Type Of Service，TOS）。

classful 队列规定（qdisc）, 类（class）和过滤器（filter）这 3 个组件组成，绘图中一般用圆形表示队列规定，用矩形表示类，图 copy 自 [Linux 下 TC 以及 netem 队列的使用](http://blog.51cto.com/professor/1570778)

![](http://7xnp02.com1.z0.glb.clouddn.com/wKiom1RU0lKAEIb9AAFPHLlaSng303.jpg)


看一张图说明一下基本原理（直接 copy 自 [流量控制工具 TC 详细说明](http://www.ituring.com.cn/article/274014)：
![](http://7xnp02.com1.z0.glb.clouddn.com/image_1b6at6vf47qubm51vq69qvaod1g.png)

* 接收包从输入接口（Input Interface）进来后，经过流量限制（Ingress Policing）丢弃不符合规定的数据包，由输入多路分配器（Input De-Multiplexing）进行判断选择......
* 如果接收包的目的是本主机，那么将该包送给上层处理；否则需要进行转发，将接收包交到转发块（Forwarding Block）处理。转发块同时也接收本主机上层（TCP、UDP 等）产生的包。转发块通过查看路由表，决定所处理包的下一跳。然后，对包进行排列以便将它们传送到输出接口（Output Interface）。
* 一般我们只能限制网卡发送的数据包，不能限制网卡接收的数据包，所以我们可以通过改变发送次序来控制传输速率。（控制网卡入口流量，需要把数据包重定向到虚拟设备 ifb， 在 ifb 的输出方向可以配置，后面会详细说这个）
* Linux 流量控制主要是在输出接口排列时进行处理和实现的。

## TC 基础

### 结构

都是以一个根 qdisc 开始的，若根 qdisc 是不分类的队列规定，那它就没有子类，因此不可能包含其他的子对象，也不会有过滤器与之关联，发送数据时，数据包进入这个队列里面排队，然后根据该队列规定的处理方式将数据包发送出去。

分类的 qdisc 内部包含一个或多个类，而每个类可以包含一个队列规定或者包含若干个子类，这些子类友可以包含分类或者不分类的队列规定，如此递归，形成了一个树。

句柄号：qdisc 和类都使用一个句柄进行标识，且在一棵树中必须是唯一的，每个句柄由主号码和次号码组成 qdisc 的次号码必须为 0（0 通常可以省略不写）

根 qdisc 的句柄为 1：，也就是 1：0。类的句柄的主号码与它的父辈相同（父类或者父 qdisc），如类 1：1 的主号码与包含他的队列规定 1：的主号码相同，1：10 和 1：11 与他们的父类 1：1 的主号码相同，也为 1。

新建一个类时，默认带有一个 pfifo_fast 类型的不分类队列规定，当添加一个子类时，这个类型的 qdisc 就会被删除，所以，非叶子类是没有队列规定的，数据包最后只能到叶子类的队列规定里面排队。

若一个类有子类，那么允许这些子类竞争父类的带宽，但是，以队列规定为父辈的类之间是不允许相互竞争带宽的。

### 基本命令

* add 命令：在一个节点里加入一个 qdisc，类或者过滤器。添加时，需要传递一个祖先作为参数，传递参数时既可以使用 ID 也可以直接传递设备的根，若建一个 qdisc 或者 filter，可以使用句柄来命名，若建一个类，使用类识别符来命名。
* del：删除由某个句柄指定的 qdisc，根 qdisc 也可以被删除，被删除的 qdisc 上的所有子类以及附属于各个类的过滤器都会被自动删除。
* change：以替代方式修改某些项目，句柄和祖先不能修改，change 和 add 语法相同。
* replace：对一个现有节点进行近于原子操作的删除 / 添加，如果节点不存在，这个命令就会建立节点。
* link：只适用于 qdisc，替代一个现有的节点

## 控制出口流量 (egress)

默认 TC 的 qdisc 控制就是出口流量，要使用 TC 控制入口，需要把流量重定向到 ifb 网卡，其实就是加了一层，原理上还是控制出口😂 。

### 先说 classless 队列

为何要先说 classless 队列，毕竟这个简单嘛，要快速使用，那么这个就是首选了。基于 classless 队列，我们可以进行故障模拟，也可以用来限制带宽。

TC 使用 linux network [netem](https://wiki.linuxfoundation.org/networking/netem) 模块进行网络故障模拟

#### 模拟网络延迟

* 添加一个固定延迟到本地网卡 eth0

```
// delay: 100ms
tc qdisc add dev eth0 root netem delay 100ms
```

* 给延迟加上上下 10ms 的波动

```
tc qdisc change dev eth0 root netem delay 100ms 10ms
```

* 加一个 25% 的相关概率

> 相关性，是这当前的延迟会和上一次数据包的延迟有关，短时间里相邻报文的延迟应该是近似的而不是完全随机的。这个值是个百分比，如果为 100%，就退化到固定延迟的情况；如果是 0% 则退化到随机延迟的情况，
> Pn = 25% Pn-1 + 75% Random


```
tc qdisc change dev eth0 root netem delay 100ms 10ms 25%
```

* 让波动变成正态分布的

```
tc qdisc change dev eth0 root netem delay 100ms 20ms distribution normal
```

#### 模拟网络丢包

* 设置丢包率为 1%

```
tc qdisc change dev eth0 root netem loss 0.1%
```

* 添加一个相关性参数

> 这个参数表示当前丢包的概率与上一条数据包丢包概率有 25% 的相关性
> Pn = 25% Pn-1 + 75% Random

```
tc qdisc change dev eth0 root netem loss 0.1% 25%
```

#### 模拟数据包重复

* 1% 的数据包重复

```
tc qdisc change dev eth0 root netem duplicate 1%
```

#### 模拟包损坏

* 2% 的包损坏

```
tc qdisc add dev eth0 root netem corrupt 2%
```

#### 模拟包乱序

网络传输并不能保证顺序，传输层 TCP 会对报文进行重组保证顺序，所以报文乱序对应用的影响比上面的几种问题要小。

报文乱序可前面的参数不太一样，因为上面的报文问题都是独立的，针对单个报文做操作就行，而乱序则牵涉到多个报文的重组。模拟报乱序一定会用到延迟（因为模拟乱序的本质就是把一些包延迟发送），netem 有两种方法可以做。

1. 第一种是固定的每隔一定数量的报文就乱序一次：

* 每 5th (10th, 15th...) 的包延迟 10ms

```
tc qdisc change dev eth0 root netem gap 5 delay 10ms
```

2. 第二种方法使用概率来选择乱序，相对来说更偏向实际情况一些

* 25 % 的立刻发送（50% 的相关性），其余的延迟 10ms

```
tc qdisc change dev eth0 root netem delay 10ms reorder 25% 50%
```

3. 只使用 delay 也可能造成数据包的乱序

```
 tc qdisc change dev eth0 root netem delay 100ms 75ms
```
> 比如 first one random 100ms, second random 25ms，这样就是会造成第一个包先于第二个包发送

#### 限制出口带宽

以 tbf (Token Bucket Filter) 为例，

参数说明：

```
tc qdisc add tbf limit BYTES burst BYTES rate KBPS [mtu BYTES] [peakrate KBPS] [latency TIME] [overhead BYTES] [linklayer TYPE]

rate 是第一个令牌桶的填充速率
peakrate 是第二个令牌桶的填充速率
peakrate>rate
burst 是第一个令牌桶的大小
mtu 是第二个令牌桶的大小
burst>mtu (此处是个坑，一定要保证 burst > mtu, 要不然一个包也发不出了)
若令牌桶中令牌不够，数据包就需要等待一定时间，这个时间由 latency 参数控制，如果等待时间超过 latency，那么这个包就会被丢弃
limit 参数是设置最多允许多少数据可以在队列中等待
latency=max((limit-burst)/rate,(limit-mtu)/peakrate);
burst 应该大于 mtu 和 rate
overhead 表示 ADSL 网络对数据包的封装开销
linklayer 指定了链路的类型，可以是以太网或者 ATM 或 ADSL
ATM 和 ADSL 报头开销均为 5 个字节。
```

* example：

限制 100mbit

```
tc qdisc add dev ens33 root tbf rate 100mbit latency 50ms burst 1600
```

限制延迟 100ms, 流量 100mbit

```
tc qdisc add dev eth0 root handle 1:0 netem delay 100ms
tc qdisc add dev eth0 parent 1:1 handle 10: tbf rate 256kbit buffer 1600 limit 3000
```

### classful 队列

这个就复杂一些，同样也特别灵活，可以限制特定的 ip 或者服务类型以及端口

以使用 htb 为例

```
// 给 root 添加 htb 队列 id: 1:0
tc qdi sc add dev eth0 root handle 1: htb default 20
// 给 htb 队列添加一个 class， clas id: 1:1
tc class add dev eth0 parent 1:0 classid 1:1 htb rate 100mbit burst 10k
// 给这个 class 1:1 加上一个 filter, 此 filter 只是对 ip 做了过滤
tc filter add dev eth0 parent 1: protocol ip prio 16 u32  match ip dst 0.0.0.0/0 flowid 1:1
```

## 控制入口流量

使用 TC 进行入口限流，需要把流量重定向到 ifb 虚拟网卡，然后在控制 ifb 的输出流量

原理图：

![](http://7xnp02.com1.z0.glb.clouddn.com/1354528849_1019.png)

```
// 开启 ifb 虚拟网卡
modprobe ifb numifbs=1
ip link set dev $IFB up

// 将 eth0 流量重定向到 ifb0
tc qdisc add dev eth0 handle ffff: ingress
tc filter add dev eth0 parent ffff: protocol ip u32 match u32 0 0 flowid 1:1 action mirred egress redirect dev ifb0

// 然后就是限制 ifb0 的输出就可以了
......
```

## 在说几句

TC 还有很多细节没有去研究到，目前这些基本上可以满足目前的需求，最后还想吐槽一句，当 TC 命令出错的时候，TC 的报错信息太少了，调试起来太坑...

> [wondershaper](https://github.com/magnific0/wondershaper) 这个脚本用起来还是很方便

## 参考

1. [Traffic Control HOWTO](http://www.tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/)
2. [netem](https://wiki.linuxfoundation.org/networking/netem)
3. [Linux 流量控制工具 TC](http://www.ituring.com.cn/article/274015)
4. [流量控制工具 TC 详细说明](http://www.ituring.com.cn/article/274014)
5. [Linux 下 TC 以及 netem 队列的使用](http://blog.51cto.com/professor/1570778)
6. [输入方向的流量控制](https://blog.csdn.net/zhangskd/article/details/8240290)
7. [wondershaper](https://github.com/magnific0/wondershaper)
