# Ceilometer源码分析 - Collector

标签（空格分隔）： Ceilometer 

---
[TOC]

## 概述
Collector负责搜集来自OpenStack其他组件（Nova、Glance、Cinder等）的Notification消息，以及compute-agent和central-agent发送到消息总结上并被collector采集到的数据，然后将这些数据存储到数据库中。

## PubSubHubbub
![PubSubHubbub](http://rnd-github.huawei.com/l00210728/nim/blob/master/performance&alarm/pic/pubsubhubbub.png?raw=true)

一个PubSubHubbub的大致流程如下：
1.	Sub找Pub订阅内容，Pub将Hub的地址发给Sub，告诉Sub：你以后找它要内容去
2.	Sub将自己要订阅的地址发给Hub，并在Hub那里注册了一个Callback函数，以后有新内容麻烦给Callback就好啦
3.	Hub可以主动，也可以被动的从Pub那里获得内容，然后再分发给在自己这里注册的Sub
图中可以看到，有这么几个关键部分，在Ceilometer中，它们对应如下：
•	Publisher - 内容提供方，OpenStack的各组件和Agent模块的角色
•	Subscriber - 内容订阅方，Collector的角色
•	Hub - 中转，Collector也充当了这个角色







