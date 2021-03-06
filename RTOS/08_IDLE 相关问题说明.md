# IDLE 相关问题说明

### 在 idle 中释放资源有什么好处和坏处

由于线程自己无法释放自己的资源，所以需要提供一种机制来释放退出线程的资源，而在 idle 线程中释放相关资源就是一种完成该工作的机制。

在 idle 中释放资源首先能想到的问题就是，idle 并不是一个普通的线程，如果该线程在运行时出现阻塞，那么可能会导致系统由于找不到可以运行的线程而跑死的问题。idle 线程是非常特殊的，说起来他是一个优先级最低的线程，但是他在关闭了调度执行 cleanup 的情况下，又有着冲破信号量等 ipc 保护的能力。

先前曾遇到的一个问题是，先关闭调度器然后再进行内存的释放，由于已经关闭了调度器，此时信号量对资源的保护就会失效，使得释放程序越过对内存释放的保护进行释放，这就导致了内存重入的问题。 

举个例子，考虑如下场景：

线程 A 做好工作后，释放信号量 1 通知线程 B 做进一步工作（可能是关闭操作，释放操作），线程 B 被唤醒后，执行相应工作，然后释放信号量 2 通知线程 A 做进一步处理，线程 A 获得信号量 2 后继续后续工作。如果线程 A 是在 IDLE 的 cleanup 中尝试释放资源，但是他在释放信号量 1 之后没有办法切换到线程 B 去继续处理，然后在尝试获取线程 B 释放的信号量 2 时，由于处在临界区中，会越过这层保护，线程 A 认为线程 B 已经处理好了后续任务，此时线程 A 就可能做进一步的数据修改，如果此后退出临界区，再次运行线程 B 时，就会可能遇到数据错误的问题。

### 单核下等待硬件导致 IDLE 重入其他线程的情况说明

如果一个线程获取了信号量 A，而他此时又被一个另外第一个信号量 B 所阻塞，而信号量 B 是在中断中释放的（也就是说该线程需要等待硬件产生中断），此时没有其他线程可以运行，系统就会切换到 idle 线程，此时由于 idle 在关闭了调度后可以冲破信号量的保护，那么在 idle 中就有可能会破坏原信号量 A 保护的资源。换句话说就是 idle 线程重入了信号量 A 保护的资源，导致系统出错。

而在多核情况下，由于在其他核上也有线程运行，而本核上的 idle 仍然有冲破信号量保护的能力，所以这块的处理更加复杂了。