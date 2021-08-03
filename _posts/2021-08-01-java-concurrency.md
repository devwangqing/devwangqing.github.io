---
layout: post
title: "java concurrency"
date: 2021-08-01 12:31:00 +0800

---



### CPU缓存

cpu加载数据时，如果每次都从内存中加载，那么会很慢（从内存中读取数据相较于cpu运行速度要慢很多）。所以cpu里增加了缓存，将变量放在缓存里来加快变量下次访问的速度。

![cpu-architecture]({{site.url}}/images/cpu-architecture.png) <br/>

cpu加载数据时，会按照 L1 cache、L2 cache、L3 cache 的顺序，逐级往下找。如果缓存中没有找到需要的数据，就会通过Bus（总线）到内存中读取。读取到内存数据后，会将数据放到缓存中，便于下次读取。

L1 cache 和 L2 cache 是每个核独有的，L3 cache 是所有核共享的。

> 现代cpu都是多核多线程的，如2核4线程，也就是说cpu有2个核，每个核有2个线程。那么L1 cache 和 L2 cache是每个核独享的，但每个核里的2个线程是共享 L1 cache 和 L2 chache的。

### MESI 协议

因为 L1 cache 和 L2 cache 是每个核独享的，那么一旦多个核都缓存了相同变量，且一个核对该变量的值做出了修改，这种情况下，其他核要怎么处理该变量呢？这里就需要用到MESI协议。

MESI协议又叫作缓存一致性协议。MESI 4个字母分别代表cpu缓存行（cache line，cpu缓存的逻辑存储单元，一般大小为64个字节）的4种状态： Modified（修改）、Exclusive（互斥即独享）、Shared（共享）、Invalid（无效的）。

+ Modified：对缓存中的数据进行了修改。
+ Exclusive：只有一个核缓存了该数据，其他核没有缓存。
+ Shared：多个核都缓存了该数据。
+ Invalid：其他核对它们缓存中的数据进行了修改，而本核收到了该数据失效的通知后，将缓存行的状态改为失效。

假设内存中有个数据x=1，cpu有core0和core1。

**Step 1.** 当 core0 发送读取数据x的消息时，因为其他核里没有缓存数据x，所以 core0 是从内存中读取数据x并存在缓存中，此时缓存行的状态为E。

![cpu-core0-read]({{site.url}}/images/cpu-core0-read.png) <br/>

**Step2.** 当 core1 发送读取数据x的消息时，因为 core0 已经缓存了x，所以 core0 会将自己的缓存行状态改成S，并将数据x发送给 core1。此时 core1 也会缓存数据x，将缓存行的状态设置成S。

![cpu-core1-read]({{site.url}}/images/cpu-core1-read.png) <br/>

**Step3.** 当 core1 将数据x的值从1改成10时，会将缓存行的状态改成M，并发送通知。通知所有缓存数据x的核，该数据无效。core0收到通知后，会将缓存行的状态设置成I。

![cpu-core1-modify]({{site.url}}/images/cpu-core1-modify.png) <br/>

### Store Buffer 和 Invalid Queue

在上面执行 Step3 时，如果 core1 等待所有的核都应答失效通知后，再将缓存行的数据x的值改成10的话，这个效率是非常的低，core1 会一直处于阻塞状态。为了解决收到所有应答才改变数据这个操作非常耗时的问题，cpu引入了 Store Buffer 和 Invalid Queue。

+ Store Buffer：缓存需要改变的数据，如x=10。
+ Invalid Queue：缓存收到的数据无效的通知。

cpu加入了 Store Buffer 和 Invalid Queue 后，上面的流程就变成如下：

core1 将数据x的值直接写入的 Store Buffer 中，然后发送数据失效通知。发送通知后，core1并不需要等待所有缓存数据x的核都应答，而是直接执行其他的cpu指令了。这样就解决了core1一直处于阻塞状态的问题。那么什么时候将 Store Buffer 的数据写入到缓存行呢？

当 core0 收到数据失效通知后，会立即应答这个通知。然后将失效通知放入到 Invalid Queue 中，等到需要执行的时候才会去执行。

当 core1 收到所有的应答后，就会将 Store Buffer 的数据写入到缓存行。

> cpu 引入 Store Buffer 和 Invalid Queue 就是引入了异步处理。
