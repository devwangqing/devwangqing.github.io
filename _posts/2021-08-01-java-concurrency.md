---
layout: post
title: "java concurrency"
date: 2021-08-01 12:31:00 +0800

---



### CPU缓存

cpu加载数据时，如果每次都从内存中加载，那么会很慢（从内存中读取数据相较于cpu运行速度要慢很多）。所以cpu里增加了缓存，将变量放在缓存里来加快变量下次访问的速度。

【缺图】

cpu加载数据时，会按照 L1 cache、L2 cache、L3 cache 的顺序，逐级往下找。如果缓存中没有找到需要的数据，那么会通过总线（Bus）到内存中加载。从内存加载到数据后，会将数据放到缓存中，便于下次读取。

L1 cache 和 L2 cache 是每个核独有的，L3 cache 是所有核共享的。


> 现在的cpu都是多核多线程的，如2核4线程，也就是说每个核有2个线程。每个核独享一个L1 cache 和 L2 cache。每个核里的2个线程是共享 L1 cache 和 L2 chache的。


### MESI 协议


因为 L1 cache 和 L2 cache 是每个核独有的，那么一旦多个核都缓存了相同变量，且一个核对该变量的值做出了修改，这种情况下，其他核要怎么处理该变量呢？这里就需要用到MESI协议。

MESI协议又叫作缓存一致性协议。MESI这4个字母分别代表着 Modified（修改）、Exclusive（互斥即独享）、Shared（共享）、Invalid（无效的）。

在cpu缓存中，数据是以缓存行（cache line）为单元被缓存的。而MESI描述的就是缓存行的状态。

（总线嗅探机制）

1.core0读取变量x后，缓存行的状态是E。

2.core1读取变量x后，core1缓存行的状态是S。core0缓存行的状态从E变成了S。

3.core1改变变量x值后，core1缓存行的状态变成M，并通知core0，将缓存行的状态改成I。

> M和I的状态是精确的，E的状态不是精确的。

【缺图】


### Store Buffer 和 Invalid Queue

在上面第3步时，如果修改了变量值，需要通知core0缓存行失效（Invalid）。这个过程需要时间，如果cpu一直在阻塞，那就太浪费cpu资源了。所以 cpu 引入了 Store Buffer 和 Invalid Queue。

Store Buffer 用来存储变量需要改变成的值；Invalid Queue 用来存储变量失效的消息。

当core1把变量x的值从1改成10时，core1会将10先存在store buffer里，然后发送变量x失效通知，通知core0。然后就继续执行其他指令了。

当core0接收到变量x失效通知时，不会立即将缓存行的状态改成I。而是先回应通知接收的消息，然后将无效通知放入 invalid queue中，等后续需要的时候，才会读取里面的消息。
