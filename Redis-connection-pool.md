# Redis Connection Pool

## 为什么使用连接池

Redis也是一种数据库，基于C/S模式，因此如果需要使用必须建立连接

通常情况下, 当需要做Redis操作时, 会创建一个连接, 并基于这个连接进行Redis操作, 操作完成后, 释放连接，一般情况下, 这是没问题的, 但当并发量比较高的时候, 频繁的连接创建和释放对性能会有较高的影响

连接池通过**预先创建多个连接**，当进行Redis操作时， 直接获取已经创建的连接进行操作， 而且操作完成后，**不会释放**， 用于后续的其他Redis操作，通过避免频繁的Redis连接创建和释放，从而提高性能

## 连接池配置（以JedisPool为例）

|属性|说明|默认值|
|---|---|---|
|maxActive|连接池中最大连接数|8|
|maxIdle|连接池中最大空闲的连接数|8|
|minIdle|连接池中最少空闲的连接数|0|
|maxWaitMillis|当连接池资源用尽后，调用者的最大等待时间（单位为毫秒），一般不建议使用默认值|-1：表示永远不超时，一直等|
|jmxEnabled|是否开启jmx监控，如果应用开启了jmx端口并且jmxEnabled设置为true，就可以通过jconsole或者jvisualvm看到关于连接池的相关统计，有助于了解连接池的使用情况，并且可以针对其做监控统计|true|
|minEvictableIdleTimeMillis|连接的最小空闲时间，达到此值后空闲连接将被移除|1000\*60\*30ms=30m|
|testOnBorrow|向连接池借用连接时是否做连接有效性检测（ping），无效连接会被移除，每次借用多执行一次ping命令|false|
|testOnReturn|向连接池归还连接时是否做连接有效性检测（ping），无效连接会被移除，每次归还多执行一次ping命令|false|
|testWhileIdle|向连接池借用连接时是否做空闲检测，空闲超时的连接将会被移除|false|
|timeBetweenEvictionRunsMills|空闲连接的检测周期（单位为毫秒）|-1：表示不做检测|
|blockWhenExhausted|当连接池用尽后，调用者是否要等待，这个参数是和maxWaitMillis对应的，只有当此参数为true时，maxWaitMillis才会生效|true|
