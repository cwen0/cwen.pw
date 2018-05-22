title: 漏桶算法 && 令牌桶算法
author: cwen
date:  2018-05-22
update:  2018-05-22
tags:
    - Linux
    - 算法
    - 网络
    - algorithm

---

在看 `TC` 文档的时候，文档中提到了 token bucket，TC 中可以使用这个算法来进行流量控制，模糊印象中听过这个算法，但是并不知道这个算法具体是什么，就好奇的去一探究竟... <!--more-->

## 先说漏桶（Leaky bucket）

从 wiki 上介绍，Leaky bucket 算法有两个版本：

* as a meter（作为计量工具）
* as a queue（作为调度队列）

其中，第一种含义和 Token Bucket 是等价的（所有我就没详细的去看版本一，而是后面直接去看 Token Bucket 算法），只是表述的角度不同。更有趣的是，第二种含义其实是第一种的特例。这些对比和区别在后面再谈，先整体看一下 Leaky Bucket。

先看一张图：

![](http://7xnp02.com1.z0.glb.clouddn.com/Leaky_bucket_analogy.JPG)

其实这图已经很好的展示了 leaky bucket 的抽象模型，就是一个会漏水的桶😂

如图描述，桶有一个输入一个输出，输出就是桶的下方有一个恒定速度的往下漏水，输入就是以一个变化的速度往水桶里进水，一旦水满了，上方的水就无法加入。对于如何处理桶满后的上方欲流下的水，有有一下两种常见的方式（其实这已经不是 leaky bucket 算法考虑的事情了）。

* [Traffic Shaping](http://en.wikipedia.org/wiki/Traffic_shaping): 暂时拦截住上方水的向下流动，等待桶中的一部分水漏走后，再放行上方水。
* [Traffic Policing](http://en.wikipedia.org/wiki/Traffic_policing): 溢出的上方水直接抛弃。

其实是将水看作网络通信中数据包的抽象，Traffic Shaping 的核心理念是 “等待”，Traffic Policing 的核心理念是 “丢弃”。它们是两种常见的流速控制方法。

`as a meter` 版本的 leaky bucket 和 token bucket 只是换个描述角度，原理上是一样的，所以就没有做过多的研究。可以看一下 wiki 上两种算法的对比，你就会发现其实是一样的。

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-05-22%20at%201.56.57%20PM.png)

## 令牌桶 （Token Bucket)

还是先上图

![](http://7xnp02.com1.z0.glb.clouddn.com/token_bucket.JPG)

概述：令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。
算法基本过程：

* 假如用户配置的平均发送速率为 r，则每隔 1/r 秒一个令牌被加入到桶中；
* 假设桶最多可以存发 b 个令牌。如果令牌到达时令牌桶已经满了，那么这个令牌会被丢弃；
* 当一个 n 个字节的数据包到达时，就从令牌桶中删除 n 个令牌，并且数据包被发送到网络；
* 如果令牌桶中少于 n 个令牌，那么不会删除令牌，并且认为这个数据包在流量限制之外；

算法允许最长 b 个字节的突发，但从长期运行结果看，数据包的速率被限制成常量 r。对于在流量限制外的数据包可以以不同的方式处理：
* 它们可以被丢弃；
* 它们可以排放在队列中以便当令牌桶中累积了足够多的令牌时再传输；
它们可以继续发送，但需要做特殊标记，网络过载的时候将这些特殊标记的包丢弃

github 实现源码:

1. https://github.com/rbarrois/throttle
2. https://github.com/titan-web/rate-limit
3. https://github.com/yangwenmai/ratelimit

## 总结

其实 `as a queue leaky bucket` 算法就是 token bucket 算法的一个特例，平时我们大多数说的漏桶算法就是指 `as a queue leaky bucket`。漏桶算法主要目的是控制数据注入到网络的速率，平滑网络上的突发流量。漏桶算法提供了一种机制，通过它，突发流量可以被整形以便为网络提供一个稳定的流量。而令牌桶算法能够在限制数据的平均传输速率的同时还允许某种程度的突发传输。

## 参考

1. [wiki Token Bucket](http://en.wikipedia.org/wiki/Token_bucket)
2. [wiki Leaky Bucket](http://en.wikipedia.org/wiki/Leaky_bucket)
3. [High-performance rate limiting](https://medium.com/smyte/rate-limiter-df3408325846)
4. [流量调整和限流技术](http://www.cnblogs.com/LBSer/p/4083131.html)
5. [漏桶算法 Leaky Bucket （令牌桶算法 Token Bucket）学习笔记](http://blog.51cto.com/leyew/860302)
6. [python 网速控制](https://caden16.github.io/python/python%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6/)

