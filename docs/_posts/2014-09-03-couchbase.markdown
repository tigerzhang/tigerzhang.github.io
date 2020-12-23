---
layout: post
title: Couchbase介绍，更好的Cache系统
date: '2014-09-03 12:05:24'
---

在移动互联网时代，我们面对的是更多的客户端，更低的请求延迟，这当然需要对数据做大量的 Cache 以提高读写速度。
#术语

* 节点：指集群里的一台服务器。
#现有 Cache 系统的特点

目前业界使用得最多的 Cache 系统主要是 memcached 和 redis。 这两个 Cache 系统都有都有很大的用户群，可以说是比较成熟的解决方案，也是很多系统当然的选择。 不过，在使用 memcached 和 redis 过程中，还是碰到了不少的问题和局限：

* Cluster 支持不够。在扩容、负载均衡、高可用等方面存在明显不足。
* 持久化支持不好，出现问题后恢复的代价大。memcached 完全不支持持久化，redis 的持久化会造成系统间歇性的负载很高。

#我期待的理想 Cache 系统

##良好的 cluster 支持

* Key 可以动态分散（Auto Sharding）在不同的服务器上，可以通过动态添加服务器节点增加系统容量。
* 没有单点失效，任何一个单点都不会造成数据不可访问。
* 读写负载可以均匀分布在系统的不同节点上。

##支持异步持久化支持

* 方便快速恢复，甚至可以直接用作 key/value 数据库。
经常在跟业界朋友交流时，会提到用 key 分段的方法来做容量扩展以及负载均衡。但是用静态的 key 分段会有不少问题：
* Cache 系统本身及使用 cache 的客户端都需要预设一个分段逻辑，这个逻辑后期如果需要调整将会非常困难。不能解决单点失效的问题，还需要额外的手段。运维需要更多的人为参与，避免 key 超出现有分区，一旦出现 key 找不到对应服务器，访问直接失败。

#最接近需求的系统：Couchbase

基于这些想法，我花了几天时间在 google, stack overflow, quora 上看了很多大家关于 cache cluster 的讨论，找到一个比较新系统 Couchbase。

![mem vs cb](/content/images/2014/Sep/74930860449119095.png)
memcached VS couchbase

##Couchbase 的集群设计对等网

Couchbase 群集所有点都是对等的，只是在创建群或者加入集群时需要指定一个主节点，一旦结点成功加入集群，所有的结点对等。

![high_level_architecture](/content/images/2014/Sep/high_level_architecture.png)

图片来源：couchbase.com

对等网的优点是，集群中的任何节点失效，集群对外提供服务完全不会中断，只是集群的容量受影响。
Smart Client

由于 couchbase 是对等网集群，所有的节点都可以同时对客户端提供服务，这就需要有方法把集群的节点信息暴露给客户端，couchbase 提供了一套机制，客户端可以获取所有节点的状态以及节点的变动，由客户端根据集群的当前状态计算 key 所在的位置。
vBucket

vBucket 概念的引入，是 couchbase 实现 auto sharding，在线动态增减节点的重要基础。

简单的解释 vBucket 可以从静态分片开始说起，静态分片的做法一般是用 key 算出一个 hash，得到对应的服务器，这个算法很简单，也容易理解。如以下代码所示：


    servers = ['server1:11211', 'server2:11211', 'server3:11211']
    server_for_key(key) = servers[hash(key) % servers.length]

但也有几个问题：

* 如果一台服务器失效，会造成该分片的所有 key 失效。
* 如果服务器容量不同，管理非常麻烦。
* 前面提到过，运维、配置非常不方便。

为了把 key 跟服务器解耦合，couchbase 引入了 vBucket。可以说 vBucket 代表一个 cache 子集，主要特点：

* key hash 对应一个 vBucket，不再直接对应服务器。
* 集群维护一个全局的 vBucket 与服务器对应表。
* 前面提到的 smart client 重要的功能就是同步 vBucket 表。

如以下代码所示：

    servers = ['server1:11211', 'server2:11211', 'server3:11211']
    vbuckets = [0, 0, 1, 1, 2, 2]
    server_for_key(key) = servers[vbuckets[hash(key) % vbuckets.length]]

![vBucket](/content/images/2014/Sep/74932204971389427.png)

图片来源：http://dustin.sallings.org/2010/06/29/memcached-vbuckets.html

由于 vBucket 把 key 跟服务器的静态对应关系解耦合，基于 vBucket 可以实现一些非常强大有趣的功能，例如：

* Replica，以 vBucket 为单位的主从备份。如果某个节点失效，只需要更新 vBucket 映射表，马上启用备份数据。
* 动态扩容。新增加一个节点后，可以把部分 vBucket 转移到新节点上，并更新 vBucket 映射表。

vBucket 非常重要，以后可以单独写一篇文章分享。

#总结

* Couchbase 的对等网设计，smart client 直接获取整的集群的信息，在客户端实现负载均衡，整个集群没有单点失效，并且完全支持平行扩展。
* vBucket 的引入，完全实现了 auto sharding，可以方便灵活的把数据的子集在不同节点上移动，以实现集群动态管理。
* Couchbase 有一个非常专业的 web 管理界面，并且支持通过 RESTful API 管理，这也是 memcached, redis 不能企及的。
* 如果只是做 key/value 的 cache，Couchbase 可以完全取代 memcached。
* Couchbase 已经被我们在生产环境中大量采用。
