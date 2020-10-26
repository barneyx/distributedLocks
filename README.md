# distributedLocks



## 0x01 什么是分布式锁

当**多个进程**在同一个系统中，用分布式锁控制多个进程对资源的访问



## 0x02 **分布式锁应用场景**

（1）传统的单体应用单机部署情况下，可以使用java并发处理相关的API进行互斥控制。

（2）分布式系统后由于多线程，多进程分布在不同机器上，使单机部署情况下的并发控制锁策略失效，为了解决跨JVM互斥机制来控制共享资源的访问，这就是分布式锁的来源；分布式锁应用场景大都是高并发、大流量场景。



## 0x03 基于etcd分布式锁的实现

### 3.1 etcd分布式锁实现的基础机制

### Lease机制

租约机制（**TTL**，Time To Live），etcd 可以为存储的 **key-value** 对设置租约，**当租约到期，key-value 将失效删除**；

同时也支持**续约**，通过客户端可以在租约到期之前续约，
以避免 **key-value** 对过期失效。

Lease 机制可以**保证分布式锁的安全性**，为锁对应的 key 配置租约，
**即使锁的持有者因故障而不能主动释放锁，锁也会因租约到期而自动释放**。

### Revision机制

每个 key 带有一个 Revision 号，每进行一次事务便+1，它是全局唯一的，
通过 Revision 的大小就可以知道进行写操作的顺序。

在实现分布式锁时，多个客户端同时抢锁，
根据 Revision 号大小依次获得锁，可以避免 “羊群效应” ，实现**公平锁**。

> 羊群效应：羊群是一种很散乱的组织，平时在一起也是盲目地左冲右撞，但一旦有一只头羊动起来，其他的羊也会不假思索地一哄而上，全然不顾旁边可能有的狼和不远处更好的草。
>
> etcd的Revision机制，可以根据Revision号的大小顺序进行写操作，因而可以避免“羊群效应”。
>
> 这根zookeeper的临时顺序节点+监听机制可以避免羊群效应的原理是一致的。

### Prefix机制

即前缀机制。

例如，一个名为 /etcd/lock 的锁，两个争抢它的客户端进行写操作，
实际写入的 key 分别为：key1="/etcd/lock/UUID1"，key2="/etcd/lock/UUID2"。

其中，UUID 表示全局唯一的 ID，确保两个 key 的唯一性。

写操作都会成功，但返回的 Revision 不一样，
那么，如何判断谁获得了锁呢？通过前缀 /etcd/lock 查询，返回包含两个 key-value 对的的 KeyValue 列表，
同时也包含它们的 Revision，通过 Revision 大小，客户端可以判断自己是否获得锁。

### Watch机制

即监听机制。

Watch 机制支持 Watch 某个固定的 key，也支持 Watch 一个范围（前缀机制）。

当被 Watch 的 key 或范围发生变化，客户端将收到通知；在实现分布式锁时，如果抢锁失败，可通过 Prefix 机制返回的 Key-Value 列表获得 Revision 比自己小且相差最小的 key（称为 pre-key），对 pre-key 进行监听，因为只有它释放锁，自己才能获得锁，如果 Watch 到 pre-key 的 DELETE 事件，则说明 pre-key 已经释放，自己将持有锁。

### 3.2 etcd分布式锁原理图

![etcd分布式锁实现原理](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/838d3a78acb54b87a35eb6cd7bb3681a~tplv-k3u1fbpfcp-zoom-1.image)

### 3.3 etcd分布式锁的实现流程
1. 建立连接

客户端连接 etcd，以 /etcd/lock 为前缀创建全局唯一的 key，假设第一个客户端对应的 key="/etcd/lock/UUID1"，第二个为key="/etcd/lock/UUID2"；客户端分别为自己的 key 创建租约 - Lease，租约的长度根据业务耗时确定；

2. 创建定时任务作为租约的“心跳”

当一个客户端持有锁期间，其它客户端只能等待，为了避免等待期间租约失效，客户端需创建一个定时任务作为“心跳”进行续约。

此外，如果持有锁期间客户端崩溃，心跳停止，key 将因租约到期而被删除，从而锁释放，避免死锁；

3. 客户端将自己全局唯一的 key 写入 etcd

执行 put 操作，将步骤 1 中创建的 key 绑定租约写入 Etcd，根据 Etcd 的 Revision 机制，假设两个客户端 put 操作返回的 Revision 分别为 1、2，客户端需记录 Revision 用以接下来判断自己是否获得锁；

4. 客户端判断是否获得锁

客户端以前缀 /etcd/lock/ 读取 key-Value 列表，判断自己 key 的 Revision 是否为当前列表中最小的，如果是则认为获得锁；否则监听列表中前一个 Revision 比自己小的 key 的删除事件，一旦监听到删除事件或者因租约失效而删除的事件，则自己获得锁；

5. 执行业务

获得锁后，操作共享资源，执行业务代码

6. 释放锁

完成业务流程后，删除对应的key释放锁

## 0x04 基于ZooKeeper分布式锁的实现

## 0x05 基于Redis分布式锁的实现

## 0x06 基于MySQL分布式锁的实现

