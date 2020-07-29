# Redis Data Type

## Redis 主要有以下几种数据类型

* Strings
* Hashes
* Lists
* Sets
* Sorted Sets

> Redis 除了这 5 种数据类型之外，还有 Bitmaps、HyperLogLogs、Streams 等

### Strings

这是最简单的类型，就是普通的 set 和 get，做简单的 KV 缓存

```bash
set name TjSanshao
```

Redis 中的字符串是一种**动态字符串**，这意味着使用者可以修改，它的底层实现有点类似于 Java 中的 ArrayList，有一个字符数组，从源码的 `sds.h/sdshdr` 文件 中可以看到 Redis 底层对于字符串的定义 SDS，即 `Simple Dynamic String` 结构

```cpp
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsignedchar flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

当字符串比较短的时候，`len` 和 `alloc` 可以使用 `byte` 和 `short` 来表示，Redis 为了对内存做极致的优化，**不同长度的字符串使用不同的结构体来表示**

#### SDS 和 C 字符串的区别

* C语言简单的字符串表示方式 不符合 Redis 对字符串在安全性、效率以及功能方面的要求
* 获取字符串长度为 O(N) 级别的操作 → 因为 C 不保存数组的长度，每次都需要遍历一遍整个数组
* 不能很好的杜绝 缓冲区溢出/内存泄漏 的问题 → 跟上述问题原因一样，如果执行拼接 or 缩短字符串的操作，如果操作不当就很容易造成上述问题、
* C 字符串 只能保存文本数据 → 因为 C 语言中的字符串必须符合某种编码（比如 ASCII），例如中间出现的 '\0' 可能会被判定为提前结束的字符串而识别不了

***注：Redis 规定了字符串的长度不得超过 512 MB***

### Hashes

这个是类似 map 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 Redis 里，然后每次读写缓存的时候，可以就操作 hash 里的**某个字段**

Redis 中的字典相当于 Java 中的 HashMap，内部实现也差不多类似，都是通过 "数组 + 链表" 的链地址法来解决部分 哈希冲突，同时这样的结构也吸收了两种不同数据结构的优点

实际上字典结构的内部包含**两个 hashtable**，**通常情况下只有一个 hashtable 是有值的**，但是在字典扩容缩容时，需要分配新的 hashtable，然后进行 **渐进式搬迁**

#### 渐进式 rehash

大字典的扩容是比较耗时间的，需要重新申请新的数组，然后将旧字典所有链表中的元素重新挂接到新的数组下面，这是一个 O(n) 级别的操作，作为单线程的 Redis 很难承受这样耗时的过程，所以 Redis 使用 渐进式 rehash 小步搬迁

![redis渐进式rehash](./images/redis渐进式rehash.png)

渐进式 rehash 会在 rehash 的同时，保留新旧两个 hash 结构，如上图所示，查询时会同时查询两个 hash 结构，然后在后续的定时任务以及 hash 操作指令中，循序渐进的把旧字典的内容迁移到新字典中，当搬迁完成了，就会使用新的 hash 结构取而代之

#### 扩缩容的条件

正常情况下，当 hash 表中 元素的个数等于第一维数组的长度时，就会开始扩容，扩容的新数组是 原数组大小的 2 倍
如果 Redis 正在做 bgsave(持久化命令)，为了减少内存也得过多分离，Redis 尽量不去扩容，但是如果 hash 表非常满了，达到了第一维数组长度的 **5** 倍了，这个时候就会 **强制扩容**

当 hash 表因为元素逐渐被删除变得越来越稀疏时，Redis 会对 hash 表进行缩容来减少 hash 表的第一维数组空间占用，所用的条件是 元素个数低于数组长度的 10%，缩容不会考虑 Redis 是否在做 bgsave

```bash
hset person name TjSanshao age 24

hget person name
```

### Lists

Lists 是有序列表，相当于Java语言中的`LinkedList`，注意它是链表而不是数组

这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)

比如可以通过 list 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西

比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 Redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走

``` bash
# 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有
lrange mylist 0 -1
```

比如可以搞个简单的消息队列，从 list 头怼进去，从 list 尾巴那里弄出来

``` bash
lpush mylist 1
lpush mylist 2
lpush mylist 3 4 5

# 1
rpop mylist
```

### Sets

Sets 是无序集合，自动去重

``` bash
#-------操作一个set-------
# 添加元素
sadd mySet 1

# 查看全部元素
smembers mySet

# 判断是否包含某个值
sismember mySet 3

# 删除某个/些元素
srem mySet 1
srem mySet 2 4

# 查看元素个数
scard mySet

# 随机删除一个元素
spop mySet

#-------操作多个set-------
# 将一个set的元素移动到另外一个set
smove yourSet mySet 2

# 求两set的交集
sinter yourSet mySet

# 求两set的并集
sunion yourSet mySet

# 求在yourSet中而不在mySet中的元素
sdiff yourSet mySet
```

### Sorted Sets

Sorted Sets 是排序的 set，去重但可以排序，写进去的时候给一个分数，自动根据分数排序

Redis 最具特色的一个数据结构，内部实现用的是一种叫做 **「跳跃表」** 的数据结构

``` bash
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu

# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
zrevrange board 0 3

# 获取某用户的排名
zrank board zhaoliu
```
