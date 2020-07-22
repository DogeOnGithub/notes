# Redis CAS

## 悲观锁

某个时刻，多个系统实例都去更新某个 key，可以基于 zookeeper 实现分布式锁，每个系统通过 zookeeper 获取分布式锁，确保同一时间，只能有一个系统实例在操作某个 key，别人都不允许读和写

![zookeeper-distributed-lock](./images/zookeeper-distributed-lock.png)

## 乐观锁

要写入缓存的数据，都是从 mysql 里查出来的，都得写入 mysql 中，写入 mysql 中的时候必须保存一个时间戳，从 mysql 查出来的时候，时间戳也查出来

每次要**写之前，先判断**一下当前这个 value 的时间戳是否比缓存里的 value 的时间戳要新，如果是的话，那么可以写，否则，就不能用旧的数据覆盖新的数据
