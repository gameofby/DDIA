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


