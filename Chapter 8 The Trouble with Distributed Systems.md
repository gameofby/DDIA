>This chapter is a thoroughly pessimistic and depressing overview of things that may go wrong in a distributed system.

# Faults and Partial Failures
单台电脑的设计，屏蔽了底层硬件错误，为用户呈现出一个idealized system model。但是分布式系统没有这样的抽象，用户需要直面各种各样的异常。

# Cloud Computing and Supercomputing
通过对比云计算和超级计算机，来说明云计算依赖的分布式系统存在极大的不稳定性。超级计算机更像是一个大号的个人计算机。而分布式系统就是另一个维度的东西。

所以，分布式系统就是要在不稳定的地基之上，通过容错设计，强行创造稳定性。
>If we want to make distributed systems work, we must accept the possibility of partial failure and build fault-tolerance mechanisms into the software. In other words, we need to build a reliable system from unreliable components.

在分布式系统领域，怀疑、悲观、疑神疑鬼者成功。
>In distributed systems, suspicion, pessimism, and paranoia pay off.

# Unreliable Networks
分布式系统是一种shared noting system, 机器之间不share memory，也不share disk。仅通过network来通信。但是，这个network是相当的不靠谱。


## Network Faults in Practice
一些实际的案例，说明网络问题是普遍的、无法完全规避的

## Detecting Faults
在一些特定情况下，可以主动检测特定network faults的。但是没有完美的解法
>the uncertainty about the network makes it difficult to tell whether a node is working or not. In some

## Timeouts and Unabandoned Delays
networks通常有着unbounded delays，所以设置timeout没有一个统一标准。timeout需要设置足够合适，过大或者过小都会对系统造成很大影响


### Network congestion and queueing
一些会造成network delay的情况：
1. network switch: 同时收到多个指向同意destination的packets, 需要one by one执行。没轮到的packet需要排队。如果queue capacity满了，再进来的packet会被丢掉，源头再重试
2. OS: CPU都busy的情况下，即使已经收到packet，OS也会queue them
3. virtual machine: CPU要在多个虚拟机之间分时运行，某个当前时刻处在等待CPU状态的虚拟机无法处理incoming packets, which are queued(buffered)
4. TCP flow control: TCP会做流控，控制其向network发送packet的rate。This means additional queueing at the sender before the data even enters the network.
5. timeout：超时的packet需要重发。application层会受这种delay影响

## Synchronous and Asynchronous Networks
为啥网络不能够做的足够可靠？这样分布式系统的实现就简单多了，可以依赖于稳定的bounded delay的网络

比如，传统的电话需要实时传输，但是确很少出现延迟或者断联，why？ 因为通信两端分配了专线(circuit)，包括中间跳转的routers，都预留了专门的bandwidth。所以可以实现非常稳定的synchronous通信

### Can we not simply make network delays predictable?
为啥TCP网络不能和电话网一样实现？
1. 电话/视频传输过程中，数据量是想对稳定的，所以可以独占一个circuit，充分利用bandwidth，也不浪费
2. 网络应用（比如email, web...）,这些的传输的数据量、持续时间等是不可预知的，为了应对这种`bursty traffic`的场景，需要更加动态灵活的设计。所以TCP是一种packet-switched protocol, 而不是circuit-switched protocol

# Unreliable clocks
## Monotonic Versus Time-of-Day Clocks
### Time-of-day clocks
以unix timestamp为例，记录了从unix epoch(midnight UTC on January 1, 1970)开始到当前的秒/毫秒。多台机器通过NTP服务器（专门用来同步时间的服务器，与原子钟/GPS等保持一致）来协调一致

### Monotonic clocks
主要用来计算duration。其时刻本身可能来自于机器启动时间，或者其他任意的时间，没有太大意义。  只用来计算time elapse

## Clock Synchronization and Accuracy



