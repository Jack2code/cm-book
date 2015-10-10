# 分布式日志收集系统Scribe

标签（空格分隔）： 未分类

---

## 简介

Scribe是facebook开源的日志收集系统，在facebook内部已经得到大量的应用。 Scribe是基于一个使用非阻断C++服务器的thrift服务的实现。它能够从各种日志源上收集日志，存储到一个中央存储系统 （可以是NFS，分布式文件系统等）上，以便于进行集中统计分析处理。它为日志的“分布式收集，统一处理”提供了一个可扩展的，高容错的方案。

## Scribe系统架构

![Scribe系统架构][1]

如上图所示：Scribe从各种数据源上收集数据，放到一个共享队列上，然后push到后端的中央存储系统上。当中央存储系统出现故障时，scribe可以暂时把日志写到本地文件中，待中央存储系统恢复性能后，scribe把本地日志续传到中央存储系统上。

## Scribe技术架构

![Scribe技术架构][2]

如上图所示：Scribe服务器底层数据通信框架是Thrift，Thrift也是Facebook开源的，并得到了广泛的使用。也用到了C++的准标准库boost，主要使用共享指针和文件相关的功能。Thrift也用到了libevent开发库和socket编程技术。

## Scribe部署架构

![Scribe部署架构][3]




  [1]: http://pic002.cnblogs.com/images/2011/162856/2011121301081683.jpg
  [2]: http://pic002.cnblogs.com/images/2011/162856/2011121301084063.jpg
  [3]: http://pic002.cnblogs.com/images/2011/162856/2011121301085749.jpg