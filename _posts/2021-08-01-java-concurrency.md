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

**场景：** 假设内存中有个数据x=1，cpu有core0和core1。

**Step 1.** 当 core0 发送读取数据x的消息时，因为其他核里没有缓存数据x，所以 core0 是从内存中读取数据x并存在缓存中，此时缓存行的状态为E。

![cpu-core0-read]({{site.url}}/images/cpu-core0-read.png) <br/>

**Step2.** 当 core1 发送读取数据x的消息时，因为 core0 已经缓存了x，所以 core0 会将自己的缓存行状态设置成S，并将数据x发送给 core1。此时 core1 也会缓存数据x，将缓存行的状态设置成S。

![cpu-core1-read]({{site.url}}/images/cpu-core1-read.png) <br/>

**Step3.** 当 core1 将数据x的值从1改成10时，会将缓存行的状态改成M，并发送通知。通知所有缓存数据x的核，该数据无效。core0收到通知后，会将缓存行的状态设置成I。

![cpu-core1-modify]({{site.url}}/images/cpu-core1-modify.png) <br/>

### Store Buffer 和 Invalid Queue

在上面执行 Step3 时，如果 core1 等待所有的核都应答失效通知后，再将缓存行的数据x的值改成10的话，这个效率是非常的低，core1 会一直处于阻塞状态。为了解决收到所有应答才改变数据这个操作非常耗时的问题，cpu引入了 Store Buffer 和 Invalid Queue。

+ Store Buffer：缓存需要改变的数据，如x=10。
+ Invalid Queue：缓存收到的数据无效的通知。

cpu加入了 Store Buffer 和 Invalid Queue 后，上面的流程就变成如下：

core1 将数据x的值直接写入的 Store Buffer 中，然后发送数据失效通知。发送通知后，core1并不会等待所有缓存数据x的核都应答，而是直接执行其他的cpu指令了。这样就解决了core1一直处于阻塞状态的问题。那么什么时候将 Store Buffer 的数据写入到缓存行呢？

当 core0 收到数据失效通知后，会立即应答这个通知，然后将失效通知放入到 Invalid Queue 中，异步将缓存行的状态设置成失效。

当 core1 收到所有的应答后，就会将 Store Buffer 的数据写入到缓存行。

> cpu 引入 Store Buffer 和 Invalid Queue 就是引入了异步处理机制。

我们还可以试想一个场景：当 core1 将数据x=10的值写入到 Store Buffer，但还没有写入到缓存行时，core1再次读取数据x的值，读到的是缓存行里的1，还是 Store Buffer 里的10呢？我们肯定是希望读到10，不然就出错了。所以cpu又引入了 `Store Forwarding`。如果读取的数据在 Store Buffer 里的话，会读取 Store Buffer，这样就能读取到之前写入的10了。

![cpu-store-forwarding]({{site.url}}/images/cpu-store-forwarding.png)

### 内存屏障

为了解决性能问题，cpu引入了 Store Buffer 和 Invalid Queue。但是 Store Buffer 和 Invalid Queue 的数据处理是异步的，所以某些场景下会出现数据不一致的情况。[内存屏障的来历](https://zhuanlan.zhihu.com/p/125549632)这篇文章就详细说明了数据不一致的原因。下面简单对文章里的问题场景进行下描述：

+ 针对于Store Buffer
  + core0 改变数据a的值，因为数据a所在缓存行的状态是S。所以数据a先写入Store Buffer，等待其他核的应答；
  + 在等待应答期间，core0又改变了数据b的值，因为数据b所在缓存行的状态是E。所以数据b的值会直接写到缓存行里；
  + 当core1读取数据b的值时，因为本地没有缓存，会读取到core0缓存行里最新数据b的值；
  + 当core0读取数据a的值时，因为本地缓存了数据a，所以读取的是本地缓存a的值；这样就出现了数据a和数据b不一致的情况；

为了解决Store Buffer数据不一致的问题，cpu加入了写屏障。

加入了写屏障，写屏障后执行的指令会写到Store Buffer中（Store Buffer不为空的话）。core0将数据a写到Store Buffer，等待应答时，core0改变数据b的值，此时不是直接将数据b写入到缓存行里，而是会写到Store Buffer 里。这样当core1读取数据b时，读到的是core0缓存行里数据b的值，这样就能保证数据a和数据b的一致性。

+ 针对于Invalid Queue
  + core0 改变数据a的值，因为数据a所在缓存行的状态是S。所以数据a先写入Store Buffer，等待其他核的应答；
  + 在等待应答期间，core0又改变了数据b的值，因为有写屏障，数据b的值写到了Store Buffer 里；
  + core1 收到失效通知后，应答该通知，并将失效通知写到Invalid Queue，等待异步处理；
  + core0 收到失效通知应答后，将Store Buffer 里数据a和数据b写入缓存行里；
  + 当core1 读取数据b时，因为本地没有缓存，会读取到core0缓存行里数据b的值；
  + 当core1读取数据a时，因为本地缓存了数据a，所以读取的是本地缓存a的值。因为Invalid Queue的失效通知还没有执行，所以缓存里a的值是一个无效的值；这样就出现了数据a和数据b不一致的情况；

为了解决Invalid Queue数据不一致的问题，cpu加入了读屏障。

加入读屏障后，core1读取数据a的值时，会先处理Invalid Queue的失效通知，然后再读取数据的a。这样就不会出现数据不一致的情况。

cpu 对指令执行了重排序，导致了数据一致性问题。使用内存屏障来解决cpu重排序导致的问题。

### JMM里的内存屏障

jvm 里加载数据指令是load，存储数据指令是store。这两个指令排列组合，就形成了4种内存屏障。

| 屏障类型            | 指令示例                   | 说明                                                         |
| :------------------ | :------------------------- | :----------------------------------------------------------- |
| LoadLoad Barriers   | Load1; LoadLoad; Load2     | 确保 Load1 数据的装载先于 Load2 及所有后续装载指令的装载     |
| StoreStore Barriers | Store1; StoreStore; Store2 | 确保 Store1 数据对其他处理器可见（刷新到内存）先于 Store2 及所有后续存储指令的存储 |
| LoadStore Barriers  | Load1; LoadStore; Store2   | 确保 Load1 数据装载先于 Store2 及所有后续的存储指令刷新到内存 |
| StoreLoad Barriers  | Store1; StoreLoad; Load2   | 确保 Store1 数据对其他处理器可见（刷新到内存）先于 Load2 及所有后续装载指令的装载。StoreLoad Barriers 会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令 |

### volatile

java的volatile的作用有两个，禁止编译器重排序和变量可见性。

#### 禁止编译器重排序

这里的重排序指的是编译器重排序，而不是cpu重排序。

#### 变量可见性

为了解决变量可见性，java的volatile干了两件事：1.使用c/c++的volatile关键字；2.内存屏障。

使用c/c++的volatile关键字，转变成汇编语言后，volatile变量会增加Lock前缀，lock前缀的作用是锁定cpu缓存，其他cpu读取该缓存的请求都会被阻塞等待，直到锁被释放。并且锁被释放的同时，会将缓存的脏数据刷新到主内存中。还利用缓存一致性协议，让其他cpu缓存的该数据失效。

如果只是单纯的将脏数据刷到主内存中，并没有解决多个数据一致性问题。操作volatile的变量时，java还增加了内存屏障。

volatile 只解决了变量可见性的问题，并没有解决原子性问题。如果要保证原子性的话，代码需要加锁。使用 synchronized 或 Lock 来解决。

<link rel="stylesheet" href="https://unpkg.com/gitalk/dist/gitalk.css">
<script src="https://unpkg.com/gitalk/dist/gitalk.min.js"></script>

<script type="javascript">
  var gitalk = new Gitalk({
  clientID: '23d11844462ff8356df3',
  clientSecret: '415f292888e045fac19698cbc5c4b8273a2cb137',
  repo: 'devwangqing.github.io',
  owner: 'devwangqing',
  admin: ['devwangqing']
})

gitalk.render('gitalk-container')
</script>
