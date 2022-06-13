# Redis的基本数据类型

## 1.前言

下面介绍的Redis命令有很多，如果你想通过死记硬背来记住这些命令几乎不可能，但是如果理解了Redis的一些机制，这些命令其实是有很强的通用性的，通过理解来记忆是最好的。 另外，每种数据类型都有其适合的使用场景，文中会给与说明，如果滥用，反而会适得其反。

![1120165-20200517211130457-705858897](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20200517211130457-705858897.png)

## 2.String数据类型

string 是Redis的最基本的数据类型，可以理解为与 Memcached 一模一样的类型，一个key 对应一个 value。string 类型是二进制安全的，意思是 Redis 的 string 可以包含任何数据，比如图片或者序列化的对象，一个 redis 中字符串 value 最多可以是 512M。

**①、相关命令介绍**

string 数据类型在 Redis 中的相关命令：![1120165-20180525081907781-1123991256](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525081907781-1123991256.png)

测试:

![1120165-20180525082445697-472653917](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525082445697-472653917.png)

PS：

　　①、上面的 ttl 命令是返回 key 的剩余过期时间，单位为秒。

　　②、mset和mget这种批量处理命令，能够极大的提高操作效率。因为一次命令执行所需要的时间=1次网络传输时间+1次命令执行时间，n个命令耗时=n次网络传输时间+n次命令执行时间，而批量处理命令会将n次网络时间缩减为1次网络时间，也就是1次网络传输时间+n次命令处理时间。

　　但是需要注意的是，Redis是单线程的，如果一次批量处理命令过多，会造成Redis阻塞或网络拥塞（传输数据量大）。

　　③、setnx可以用于实现分布式锁，具体实现方式后面会介绍。

　　上面是 string 类型的基本命令，下面介绍几个自增自减操作，这在实际工作中还是特别有用的（分布式环境中统计系统的在线人数，利用Redis的高性能读写，在Redis中完成秒杀，而不是直接操作数据库。）。![1120165-20180525081933519-1451227901](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525081933519-1451227901.png)

测试:

![1120165-20180525082803456-307428951](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525082803456-307428951.png)

**②、典型使用场景**

　	一、计数

　　由于Redis单线程的特点，我们不用考虑并发造成计数不准的问题，通过 incrby 命令，我们可以正确的得到我们想要的结果。

　　二、限制次数

　　比如登录次数校验，错误超过三次5分钟内就不让登录了，每次登录设置key自增一次，并设置该key的过期时间为5分钟后，每次登录检查一下该key的值来进行限制登录。

## 3.hash数据类型

hash 是一个键值对集合，是一个 string 类型的 key和 value 的映射表，key 还是key，但是value是一个键值对（key-value）。类比于 Java里面的 Map<String,Map<String,Object>> 集合。

**①、相关命令介绍**

![1120165-20180525082836433-593876169](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525082836433-593876169.png)

测试:

![1120165-20180525213950542-1792125541](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525213950542-1792125541.png)

**②、典型使用场景**

查询的时间复杂度是O(1)，用于缓存一些信息。

## 4.list数据类型

list 列表，它是简单的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边），它的底层实际上是个链表。

　　列表有两个特点：

　　**一、有序**

　　**二、可以重复**

　　这两个特点要注意和后面介绍的集合和有序集合相对比。

**①、相关命令介绍**

![1120165-20180525214044910-2026624164](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525214044910-2026624164.png)

![1120165-20180525214115346-1562218773](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525214115346-1562218773.png)

测试：

![1120165-20180525220954036-1057045680](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525220954036-1057045680.png)

**②、典型使用场景**

一、栈

通过命令 lpush+lpop

二、队列

命令 lpush+rpop

三、有限集合

命令 lpush+ltrim

四、消息队列

命令 lpush+brpop

## 5.set数据类型

Redis 的 set 是 string 类型的无序集合。

相对于列表，集合也有两个特点：

一、无序

二、不可重复

**①、相关命令介绍**

![1120165-20180525221025780-633049034](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525221025780-633049034.png)

![1120165-20180525221046544-512036810](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525221046544-512036810.png)

测试:

![1120165-20180525233646456-1578709654](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525233646456-1578709654.png)

**②、典型使用场景**

利用集合的交并集特性，比如在社交领域，我们可以很方便的求出多个用户的共同好友，共同感兴趣的领域等。

## 6.zset数据类型

zset（sorted set 有序集合），和上面的set 数据类型一样，也是 string 类型元素的集合，但是它是有序的。

**①、相关命令介绍**

![1120165-20180525233739895-2031411316](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525233739895-2031411316.png)

测试:

![1120165-20180525234200567-580910330](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525234200567-580910330.png)

**②、典型使用场景**

和set数据结构一样，zset也可以用于社交领域的相关业务，并且还可以利用zset 的有序特性，还可以做类似排行榜的业务。

## 7.Redis5.0新数据结构-stream

　Redis的作者在Redis5.0中，放出一个新的数据结构，Stream。Redis Stream 的内部，其实也是一个队列，每一个不同的key，对应的是不同的队列，每个队列的元素，也就是消息，都有一个msgid，并且需要保证msgid是严格递增的。在Stream当中，消息是默认持久化的，即便是Redis重启，也能够读取到消息。那么，stream是如何做到多播的呢？其实非常的简单，与其他队列系统相似，Redis对不同的消费者，也有消费者Group这样的概念，不同的消费组，可以消费同一个消息，对于不同的消费组，都维护一个Idx下标，表示这一个消费群组消费到了哪里，每次进行消费，都会更新一下这个下标，往后面一位进行偏移。

## 8.系统相关命令

![1120165-20180525234729480-1178536531](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525234729480-1178536531.png)

## 9.key相关命令

关于 key 的命令应该说是最常用的，需要大家记住。![1120165-20180525234315385-857227355](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525234315385-857227355.png)

![1120165-20180525234334105-751375191](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525234334105-751375191.png)

测试:

![1120165-20180525234615296-1573443943](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180525234615296-1573443943.png)

这里在介绍一个命令 ：

```redis
OBJECT ENCODING    key  
```

这是用来显示这五种数据类型的底层数据结构

![1120165-20180527215124137-711404655](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/1120165-20180527215124137-711404655.png)

上面的命令我们给string 数据类型 k1 赋值 str，给 k2 赋值 123，通过 OBJECT ENCODING 显示底层实现的数据类型分别是 embstr 和 int。这跟它们底层的数据结构有关。