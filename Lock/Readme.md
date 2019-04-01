### 线程的中断
1. Object类的wait(),wait(long),wait(long,int),Thread join(),sleep()
2. 实现了 InterruptibleChannel 接口的类中的一些 I/O 阻塞操作，如 DatagramChannel 中的 connect 方法和 receive 方法等
3.  Selector 中的 select 方法:一旦中断，方法立即返回
#### 对于以上 3 种情况是最特殊的，因为他们能自动感知到中断（这里说自动，当然也是基于底层实现），并且在做出相应的操作后都会重置中断状态为 false。
**唤醒后不会重置中断状态，所以唤醒后去检测中断状态将是 true**
**Lock方法不响应中断，但是会记录中断信息**
