---
layout: post
title:  Discord 如何存储数十亿消息数据
excerpt: cassandra 在 Discord 中的应用
comments: true
share: true
categories: articles
---

原文链接: https://blog.discordapp.com/how-discord-stores-billions-of-messages-7fa6ec7ee4c7#.p2vea9li9


Discord 的发展速度已经超出了我们的预期，更多的用户产生更多的聊天信息。7 月份，一天的消息量
已经达到了 4000 万，12 月份的时候就到了 1 亿，在写这篇博客的时候每天的消息量已经超过了 1.2
亿。我们很早就决定永久存储聊天记录，用户可以在任何时间，任何设备上查看他们的数据。有如此多的
数据，而且还在不断的增长，并且要保证可用性，我们要如何实现呢？我们选择了 Cassandra！

## 我们在做什么

Discord 的早期版本是在 2015 年初花了两个月左右的时间完成的。在这种情况下，满足快速迭代的
最佳数据库就是 MongoDB。 Discord 的所有数据都是存储在一个单独的 MongoDB 副本集合里，这是
我们有意为之的，但是我们也有计划迁移到新的数据库（我知道由于 MongoDB 分片 使用起来很复杂并且
有未知的不稳定因素，我们将不再使用它）。实际上这也是我们公司文化的一部分：快速构建以证明产品
特性，不过通常会使用比较粗暴的解决方案。

消息被存储在一个以 channel_id 和 created_at 字段为组合索引的 MongoDB 集合中。在 2015 年的
11 月份左右，消息数达到了 1 亿的存储量，此时我们预计到的问题发生了：RAM 没有足够的空间存储
数据和索引，延迟也变得不可预期，所以是时候迁移到一个新的数据库了。


## 选择正确的数据库

在选择一个新的数据库之前，我们必须理解当前系统的读写模式以及当前的解决方案为什么会产生这样的
问题。

* 很显然的是，我们的读操作非常的随机并且读写比例是 50/50；

* 语音聊天服务器几乎不发送文本消息，每隔几天才会产生一条文本消息，就算是一年也不太可能会产生
1000 条文本消息。问题在于即便是如此少的文本消息数据也很难提供给用户，就算只返回 50 条信息也可
能导致许多磁盘随机搜索从而导致磁盘缓存驱逐；

* 私有的文本聊天服务器会发送很多消息，一年时间内很容易达到 10 - 100 万条记录，用户请求的
数据通常是最近的，问题在于这些服务器的用户通常在 100 人以下，这些数据被请求的频率很低，因此
不太可能在磁盘缓存里获取到。

* 庞大的公共聊天服务器会发送大量的消息，数以千计的用户每天都会发送很多消息，所以一年很容易
就会产生几百万条消息。这类数据会在最近的一个小时里被频繁访问，因此数据通常会在磁盘缓存里。

* 我们知道在未来的一年里，我们会提供更多的方式去帮助用户实现随机读：有能力浏览关于你过去 30
 天内的数据，然后跳转到历史记录的某个点上，以及全文搜索。所有的这些都预示着更多的随机读！！


接下来定义我们的需求：

* Linear scalability（线性扩展）—— 我们希望推迟该方案，也不想手动重新分配（re-shard）数据。

* Automatic failover（自动故障转移）—— 我们喜欢晚上睡觉，让 Discord 可以尽可能的自我治愈。

* Low maintenance （维护成本低）—— 搭建好后就可以工作，随着数据的增长，我们只需要添加节点。

* Proven to work （被证明可用）—— 我们喜欢尝试新技术，但不要太新！

* Predictable performance （可预期的性能）—— 当 95% 的接口响应时间超过 80 ms 时，我们
应该收到警报，我们也不想用 Redis 或者 Memcached 缓存消息。

* Not a blob store （不是二进制对象存储）—— 如果我们不得不经常反序列化和追加 blob，每秒
成千上万的写消息将会很糟糕。

* Open source （开源）—— 我们想自己控制命运而不像依赖于一个第三方公司。






