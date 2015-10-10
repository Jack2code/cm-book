# Disruptor机制原理

标签（空格分隔）： Disruptor

---
[TOC]

## RingBuffer到底是什么？
顾名思义，`RingBuffer`是一个环（首尾相接的环），而且你可以把它用在不同上下文（线程）间传递数据的buffer。

![][1]

`RingBuffer`拥有一个游标，这个游标指向环中下一个可用的元素。如下图所示，游标指向环的索引为4的位置：

![][2]

随着你不停地填充这个buffer（可能也会有相应的读取），这个游标会一直增长，直到绕过这个环（绕圈）。

![][3]

要找到环中当前游标指向的元素，可以通过mod（取模）操作：

```
sequence mod array length = array index           
```

以上面的`RingBuffer`为例（java的mod语法）：`12 % 10 = 2`。很简单吧。

事实上，上图中的`RingBuffer`只有10个槽只是个假设。如果槽的个数是2的N次方更有利于基于二进制的计算机进行计算。
（*`sequence & （array length－1） = array index`，比如一共有8槽，`3&（8－1）=3`，HashMap就是用这个方式来定位数组元素的，这种方式比取模的速度更快。*）

## 那又怎么样？
如果你看了维基百科里面的关于环形buffer的词条，你就会发现，我们的实现方式，与其最大的区别在于：没有尾指针。我们只维护了一个指向下一个可用位置的游标。我们选择用环形buffer的最初原因就是想要提供可靠的消息传递。我们需要将已经被服务发送过的消息保存起来，这样当另外一个服务通过`NAK`[^nak]告诉我们没有成功收到消息时，我们能够重新发送给他们。

听起来，环形buffer非常适合这个场景。它维护了一个指向尾部的游标，当收到`NAK`请求，可以重发从那一点到当前游标之间的所有消息：

![][4]

我们实现的`RingBuffer`和常用的队列之间的区别是：不删除buffer中的数据，也就是说这些数据一直存放在buffer中，直到新的数据覆盖他们。这就是和维基百科版本相比，我们不需要尾指针的原因。`RingBuffer`本身并不控制是否需要重叠（*决定是否重叠是生产者-消费者行为模式的一部分*）。

## 它为什么如此优秀？
RingBuffer之所以采用这种数据结构，是因为它在可靠消息传递方面有很好的性能。这就够了，不过它还有一些其他的优点。

**首先**，因为它是数组，所以要比链表快，而且有一个容易预测的访问模式（*数组内元素的内存地址是连续存储的*）。这是对CPU缓存友好的—也就是说，在硬件级别，数组中的元素是会被预加载的，因此在`RingBuffer`当中，cpu无需时不时去主存加载数组中的下一个元素（*因为只要一个元素被加载到缓存行，其他相邻的几个元素也会被加载进同一个缓存行*）。

**其次**，可以为数组预先分配内存，使得数组对象一直存在（除非程序终止）。这就意味着不需要花大量的时间用于垃圾回收。此外，不需要像链表那样，添加对象时创建对象，删除节点时，需要进行内存清理。

### ProducerBarriers
Disruptor代码给**消费者**提供了一些接口和辅助类，但是没有给**生产者**提供写入`RingBuffer`的接口。这是因为除了你需要知道生产者之外，没有别人需要访问它。尽管如此，`RingBuffer`还是像消费端一样提供了一个 `ProducerBarrier`对象，让生产者通过它来写入`RingBuffer`。

写入`RingBuffer`的过程涉及到两阶段提交 (`two-phase commit`)。首先，你的生产者需要申请buffer里的下一个节点。然后，当生产者向节点写完数据，它将会调用`ProducerBarrier`的`commit`方法。

那么让我们先来看看第一步。 “给我`RingBuffer`里的下一个节点”，这句话听起来很简单。的确，从生产者角度来看它很简单：简单地调用`ProducerBarrier`的`nextEntry()`方法，这样会返回给你一个`Entry`对象，这个对象就是`RingBuffer`的下一个节点。

### ProducerBarrier如何防止RingBuffer重叠
在后台，由`ProducerBarrier`负责所有的交互细节来从`RingBuffer`中找到下一个节点，然后才允许生产者向它写入数据。

![][5]

在上图中，我们假设只有一个生产者写入 `RingBuffer`。过一会儿我们再处理多个生产者的复杂问题。

`ConsumerTrackingProducerBarrier`对象拥有所有正在访问`RingBuffer`的消费者列表。这看起来有点儿奇怪－我从没有期望`ProducerBarrier`了解任何有关消费端那边的事情。但是等等，这是有原因的。因为我们不想与队列“混为一谈”（队列需要追踪队列的头和尾，它们有时候会指向相同的位 置），`Disruptor`由消费者负责通知它们处理到了哪个序列号，而不是`RingBuffer`。所以，如果我们想确定我们没有让`RingBuffer`重叠，就需要检查所有的消费者们都读到了哪里。

在上图中，有一个消费者顺利的读到了最大序号12（用红色/粉色高亮）。第二个消费者有点儿落后——可能它在做I/O操作之类的——它停在序号3。因此消费者2在赶上消费者1之前要跑完整个`RingBuffer`一圈的距离。

现在生产者想要写入`RingBuffer`中序号3占据的节点，因为它是`RingBuffer`当前游标的下一个节点。但是`ProducerBarrier`明白现在不能写入，因为有一个消费者正在占用它。所以，`ProducerBarrier`停下来自旋 (spins)，等待，直到那个消费者离开。

##申请下一个节点
现在可以想像消费者2已经处理完了一批节点，并且向前移动了它的游标。可能它挪到了序号9（因为消费端的批处理方式，现实中我会预计它到达12，但那样的话这个例子就不够有趣了）。

![][6]

上图显示了当消费者2挪动到序号9时发生的情况。在这张图中我已经忽略了`ConsumerBarrier`，因为它没有参与这个场景。
`ProducerBarier`会看到下一个节点——序号3那个已经可以用了。它会抢占这个节点上的`Entry`（我还没有特别介绍`Entry`对象，基本上它是一个放写入到某个序号的`RingBuffer`数据的桶），把下一个序号（13）更新成`Entry`的序号，然后把`Entry`返回给生产者。生产者可以接着往`Entry`里写入数据。

## 提交新的数据
两阶段提交的第二步是——对，提交。

![][7]

绿色表示最近写入的`Entry`，序号是13。
当生产者结束向`Entry`写入数据后，它会要求`ProducerBarrier`提交。

`ProducerBarrier`先等待`RingBuffer`的游标追上当前的位置（对于单生产者这毫无意义－比如，我们已经知道游标到了12，而且没有其他人正在写入`RingBuffer`）。然后`ProducerBarrier`更新`RingBuffer`的游标到刚才写入的`Entry`序号－在我们这儿是13。接下来，`ProducerBarrier`会让消费者知道buffer中有新东西了。它戳一下`ConsumerBarrier`上的`WaitStrategy`对象说－“喂，醒醒！有事情发生了！”（注意－不同的`WaitStrategy`实现以不同的方式来实现提醒，取决于它是否采用阻塞模式。）

现在消费者1可以读`Entry13`的数据，消费者2可以读`Entry13`以及前面的所有数据，然后它们都过得很happy。

## ProducerBarrier上的批处理
有趣的是`Disruptor`可以同时在生产者和消费者两端实现批处理。还记得伴随着程序运行，消费者2最后达到了序号9吗？`ProducerBarrier`可以在这里做一件很狡猾的事－它知道`RingBuffer`的大小，也知道最慢的消费者位置。因此它能够发现当前有哪些节点是可用的。

![][8]

如果`ProducerBarrier`知道`RingBuffer`的游标指向12，而最慢的消费者在9的位置，它就可以让生产者写入节点3，4，5，6，7和8，中间不需要再次检查消费者的位置。

## 多个生产者的场景
到这里你也许会以为我讲完了，但其实还有一些细节。

在上面的图中我稍微撒了个谎。我暗示了`ProducerBarrier`拿到的序号直接来自`RingBuffer`的游标。然而，如果你看过代码的话，你会发现它是通过 `ClaimStrategy`获取的。我省略这个对象是为了简化示意图，在单个生产者的情况下它不是很重要。

在多个生产者的场景下，你还需要其他东西来追踪序号。这个序号是指当前可写入的序号。注意这和“向Ring Buffer的游标加1”不一样－如果你有一个以上的生产者同时在向`RingBuffer`写入，就有可能出现某些`Entry`正在被生产者写入但还没有提交的情况。

![][9]

让我们复习一下如何申请写入节点。每个生产者都向`ClaimStrategy`申请下一个可用的节点。生产者1拿到序号13，这和上面单个生产者的情况一样。生产者2拿到序号14，尽管`RingBuffer`的当前游标仅仅指向12。这是因为`ClaimSequence`不但负责分发序号，而且负责跟踪哪些序号已经被分配。

现在每个生产者都拥有自己的写入节点和一个崭新的序号。

我把生产者1和它的写入节点涂上绿色，把生产者2和它的写入节点涂上可疑的粉色－看起来像紫色。

![][10]

现在假设生产者1还生活在童话里，因为某些原因没有来得及提交数据。生产者2已经准备好提交了，并且向`ProducerBarrier`发出了请求。

就像我们先前在`commit`示意图中看到的一样，`ProducerBarrier`只有在 `RingBuffer`游标到达准备提交的节点的前一个节点时它才会提交。在当前情况下，游标必须先到达序号13我们才能提交节点14的数据。但是我们不能这样做，因为生产者1正盯着一些闪闪发光的东西，还没来得及提交。因此 `ClaimStrategy`就停在那儿自旋 (spins)， 直到`RingBuffer`游标到达它应该在的位置。

![][11]

现在生产者1从迷糊中清醒过来并且申请提交节点13的数据（生产者1发出的绿色箭头代表这个请求）。`ProducerBarrier`让`ClaimStrategy`先等待`RingBuffer`的游标到达序号12，当然现在已经到了。因此`RingBuffer`移动游标到13，让`ProducerBarrier`戳一下`WaitStrategy`告诉所有人都知道`RingBuffer`有更新了。现在`ProducerBarrier`可以完成生产者2的请求，让`RingBuffer`移动游标到14，并且通知所有人都知道。

你会看到，尽管生产者在不同的时间完成数据写入，但是`RingBuffer`的内容顺序总是会遵循`nextEntry()`的初始调用顺序。也就是说，如果一个生产者在写入`RingBuffer`的时候暂停了，只有当它解除暂停后，其他等待中的提交才会立即执行。


  [1]: http://s4.51cto.com/wyfs01/M01/0F/5D/wKioOVHBJZTAf1LjAAAYhJtfSLs240.jpg
  [2]: http://s4.51cto.com/wyfs01/M01/0F/5C/wKioJlHBJZTBMmQpAAAmGPxPLmQ319.jpg
  [3]: http://s1.51cto.com/wyfs01/M00/0F/5C/wKioJlHBJZTC4GyHAAAkysAIzvw325.jpg
  [4]: http://s5.51cto.com/wyfs01/M00/0F/5D/wKioOVHBJZSweQJtAAAunA8JQ4Q654.jpg
  [5]: http://s8.51cto.com/wyfs01/M00/0F/50/wKioOVG_0kKihz4fAAB2U89N4PQ157.jpg
  [6]: http://s4.51cto.com/wyfs01/M01/0F/50/wKioOVG_0kPjdUCuAACAEGBsVTY487.jpg
  [7]: http://s8.51cto.com/wyfs01/M00/0F/4E/wKioJlG_0kOjuZJkAABzQr0GGVQ716.jpg
  [8]: http://s9.51cto.com/wyfs01/M01/0F/4E/wKioJlG_0kPwZHUyAAB5Dct1Vbs169.jpg
  [9]: http://s3.51cto.com/wyfs01/M01/0F/50/wKioOVG_0kPANMTmAACOvUAHxbw728.jpg
  [10]: http://s8.51cto.com/wyfs01/M00/0F/50/wKioOVG_0kORGA88AABlFmWGd6g622.jpg
  [11]: http://s6.51cto.com/wyfs01/M01/0F/4E/wKioJlG_0kPj4wPLAACm4N2N-H4811.jpg
  [^nak]: 拒绝应答信号