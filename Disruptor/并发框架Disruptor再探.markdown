# 并发框架Disruptor再探

标签（空格分隔）： Disruptor

---
[TOC]

## 剖析Disruptor：为什么会这么快
1. [剖析Disruptor：为什么会这么快？（一）锁的缺点][1]
2. [剖析Disruptor：为什么会这么快？（二）神奇的缓存行填充][2]
3. [剖析Disruptor：为什么会这么快？（三）伪共享][3]
4. [剖析Disruptor：为什么会这么快？（四）揭秘内存屏障][4]

## Disruptor如何工作和使用
1. [如何使用Disruptor（一）RingBuffer的特别之处][5]
2. [如何使用Disruptor（二）如何从RingBuffer读取][6]
3. [如何使用Disruptor（三）写入RingBuffer][7]
4. [解析Disruptor关系组装][8]
5. [Disruptor（无锁并发框架）-发布][9]
6. [LMAX Disruptor--一个高性能、低延迟且简单的框架][10]
7. [Disruptor Wizard已死，Disruptor Wizard永存！][11]
8. [Disruptor 2.0更新摘要][12]
9. [线程间共享数据不需要竞争][13]

## Disruptor的应用
1. [LMAX的架构][14]
2. [通过Axon和Disruptor处理1M tps][15]


  [1]: http://ifeve.com/locks-are-bad/
  [2]: http://ifeve.com/disruptor-cacheline-padding/
  [3]: http://ifeve.com/falsesharing/
  [4]: http://ifeve.com/disruptor-memory-barrier/
  [5]: http://ifeve.com/dissecting-disruptor-whats-so-special/
  [6]: http://ifeve.com/dissecting_the_disruptor_how_doi_read_from_the_ring_buffer/
  [7]: http://ifeve.com/disruptor-writing-ringbuffer/
  [8]: http://ifeve.com/dissecting-disruptor-wiring-up-cn/
  [9]: http://ifeve.com/the-disruptor-lock-free-publishing/
  [10]: http://ifeve.com/disruptor-dsl/
  [11]: http://ifeve.com/disruptor-wizard/
  [12]: http://ifeve.com/disruptor-2-change/
  [13]: http://ifeve.com/sharing-data-among-threads-without-contention/
  [14]: http://ifeve.com/lmax/
  [15]: http://ifeve.com/axon/