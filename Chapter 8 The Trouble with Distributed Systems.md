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
