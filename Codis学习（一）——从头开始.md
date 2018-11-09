Codis学习（一）——从头开始
===========================



Codis是一个用Go编写的分布式、高性能Redis集群解决方案。并且已投入生产，广泛应用于许多公司。

对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有显著区别 (除了部分codis不支持的命令外), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。
****
	
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com



****
## 今日内容预览
* [scan功能简介](#scan功能简介)
* [python多线程简介](#python多线程简)
* [scan+多线程集群遍历实现](#scan+多线程集群遍历实现)




### 与Twemproxy和Redis-Cluster对比
-----------
|-|Codis|Twemproxy|Redis-Cluster|
|-|-|-|-|-|
|不重启集群的情况下resharding|支持|不支持|支持|
|pipeline|支持|支持|不支持|
|hash的多域操作|支持|支持|支持|
|resharding时的多键操作|支持|-|不支持|
|Redis客户端支持|任意客户端|任意客户端|客户端必须支持cluster协议|

“resharding”意味着将数据从一个redis服务器迁移到另一个redis服务器，通常在增加/减少redis服务器数量时发生。

相对于twemproxy的优劣？
>codis和twemproxy最大的区别有两个：一个是codis支持动态水平扩展，对client完全透明不影响服务的情况下可以完成增减redis实例的操作；一个是codis是用go语言写的并支持多线程而twemproxy用C并只用单线程。 后者又意味着：codis在多核机器上的性能会好于twemproxy；codis的最坏响应时间可能会因为GC的STW而变大，不过go1.5发布后会显著降低STW的时间；如果只用一个CPU的话go语言的性能不如C，因此在一些短连接而非长连接的场景中，整个系统的瓶颈可能变成accept新tcp连接的速度，这时codis的性能可能会差于twemproxy。


相对于redis cluster的优劣？
>redis cluster基于smart client和无中心的设计，client必须按key的哈希将请求直接发送到对应的节点。这意味着：使用官方cluster必须要等对应语言的redis driver对cluster支持的开发和不断成熟；client不能直接像单机一样使用pipeline来提高效率，想同时执行多个请求来提速必须在client端自行实现异步逻辑。 而codis因其有中心节点、基于proxy的设计，对client来说可以像对单机redis一样去操作proxy（除了一些命令不支持），还可以继续使用pipeline并且如果后台redis有多个的话速度会显著快于单redis的pipeline。同时codis使用zookeeper来作为辅助，这意味着单纯对于redis集群来说需要额外的机器搭zk，不过对于很多已经在其他服务上用了zk的公司来说这不是问题：）

### Codis特性
-----------
> 可视化管理界面以及admin命令行管理工具
> 支持大多数Redis命令，与Twemproxy完全兼容
> 动态扩缩容


### Codis组成
-----------
Codis 3.x 由以下组件组成：

>Codis Server：基于 redis-3.2.8 分支开发。增加了额外的数据结构，以支持 slot 有关的操作以及数据迁移指令。具体的修改可以参考文档 redis 的修改。

>Codis Proxy：客户端连接的 Redis 代理服务, 实现了 Redis 协议。 除部分命令不支持以外(不支持的命令列表)，表现的和原生的 Redis 没有区别（就像 Twemproxy）。

对于同一个业务集群而言，可以同时部署多个 codis-proxy 实例；
不同 codis-proxy 之间由 codis-dashboard 保证状态同步。
>Codis Dashboard：集群管理工具，支持 codis-proxy、codis-server 的添加、删除，以及据迁移等操作。在集群状态发生改变时，codis-dashboard 维护集群下所有 codis-proxy 的状态的一致性。

对于同一个业务集群而言，同一个时刻 codis-dashboard 只能有 0个或者1个；
所有对集群的修改都必须通过 codis-dashboard 完成。
>Codis Admin：集群管理的命令行工具。

可用于控制 codis-proxy、codis-dashboard 状态以及访问外部存储。
>Codis FE：集群管理界面。

多个集群实例共享可以共享同一个前端展示页面；
通过配置文件管理后端 codis-dashboard 列表，配置文件可自动更新。
>Storage：为集群状态提供外部存储。

提供 Namespace 概念，不同集群的会按照不同 product name 进行组织；
目前仅提供了 Zookeeper、Etcd、Fs 三种实现，但是提供了抽象的 interface 可自行扩展。


### codis的分片
-----------
Codis 采用 Pre-sharding 的技术来实现数据的分片, 默认分成 1024 个 slots (0-1023), 对于每个key来说, 通过以下公式确定所属的 Slot Id : SlotId = crc32(key) % 1024。

每一个 slot 都会有一个且必须有一个特定的 server group id 来表示这个 slot 的数据由哪个 server group 来提供。数据的迁移也是以slot为单位的。

### 关于迁入codis
-----------
分两种情况:

1、原来使用 twemproxy 的用户: 可以, 使用codis项目内的redis-port工具, 可以实时的同步 twemproxy 底下的 redis 数据到你的 codis 集群上. 搞定了以后, 只需要你修改一下你的配置, 将 twemproxy 的地址改成 codis 的地址就好了. 除此之外, 你什么事情都不用做.

2、原来使用 Redis 的用户: 如果你使用了下面提到的codis不支持的命令，是无法直接迁移到 Codis 上的. 你需要修改你的代码, 用其他的方式实现.

### codis不支持的命令列表
-----------
以下命令codis不允许使用，如果使用，proxy会关闭连接进行警告。
|类型|命令|
|-|-|
|keys|keys,migrate,move,object,randomkey,rename,renamenx,scan|
|string|bitop,msetnx|
|list|blpop,brpop,brpoplpush|
|pub/sub|psubscribe,publish,punsbscribe,subscribe,unsubscribe|
|transaction|discard,exec,multi,unwatch,watch|
|scripting|script|
|server|bgrewriteaof,bgsave,client,config,dbsize,debug,flushall,flushdb,lastsave,latency,monitor,psync,replconf,restore,save,shutdown,slaveof,slowlog,sync,time|
|codis slot|SLOTSCHECK,SLOTSDEL,SLOTSINFO,SLOTSMGRTONE,SLOTSMGRTSLOT,SLOTSMGRTTAGONE,SLOTSMGRTTAGSLOT|

以下命令是“半支持”的。Codis不支持跨节点操作，因此您必须使用哈希标签将一个请求中显示的所有密钥放入同一个插槽中，然后您就可以使用这些命令。 Codis不检查密钥是否具有相同的标签，因此如果您不使用标签，您的程序将得到错误的响应。
|类型|命令|
|-|-|
|list|rpoplpush|
|set|sdiff,sinter,sinterstore,smove,sunion,sunionstore|
|sorted set|zinterstore,zunionstore|
|hyperloglog|pfmerge|
|scripting|eval,evalsha|