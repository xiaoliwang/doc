# Redis 开发运维实践规范

## 简介

该文档主要包含开发过程中的一些规范和运维过程中的一些经验



## 开发设计规范

### 数据库库设计

使用以下命令切换数据库：

```redis
select db-index
```

默认数据库为0，默认数据库数量为16个。约定以下规则：

1. 使用 db0 作为共享数据库，存放内容为所有项目均可访问的共享内存信息。
2. db1 ~ db9 为私有数据库，仅项目本身之间可以共享内存信息。
3. db10 ~ db15 为群组数据库，存放指定项目组的共享内存信息。




### Key 设计

key 的格式约定为：`type:object:[attr]:[attr_value]:[field]`。其中 `:` 为分隔符，`[]` 中的内容为可选项。使用 `_` 作为单词间的连接符。键名中不能包含

> key 中除了分隔符和连接符只能包含数字和字母
>
> 不推荐使用含义不清的key，key的长度限制在 100 个字节以内

其中 type 为 key 的类型。主要有以下几种类型：

- 锁：lock
- 缓存：cache
- session：session
- 计数器：counter
- 队列：list
- 验证数据：validate

object 为对象名。如果某个对象用于多处不同目的，也在此进行区分，使用 `_` 进行分隔。

attr 为对象属性名，attr_value 为属性值。当对象名无法制定唯一对象时，使用该附加条件进行约束。该键值对可以存在多个。

field 为缓存对象属性名。如果缓存的不是整个对象而是对象的某个属性，使用该字段进行区分。

以下是对 user 表的一个例子：

| id   | name   | email          | password |
| ---- | ------ | -------------- | -------- |
| 1    | xiaoli | xiaoli@163.com | 1234     |

对 user 中的 email 字段进行缓存时，key 可以设置为 `cache:user:id:1:email`。

对 user 更新上锁时， key可以设置为 `lock:user:id:1` 。 



### 分布式锁设计

对于 2.6.12 版本及以上版本。

```redis
set key value [EX seconds] [NX]
```

以下主要讨论较早的版本。容易想到使用以下语句：

```redis
# 上锁
setnx lock:temp 1;
expire lock:temp time;

# 解锁
delete lock:temp;
```

但是这里会出现一个致命的问题。如果上锁之后，进程因为某些原因突然挂掉了，就会出现死锁的情况。经过重新设计，我们将该方案优化为以下版本：

```redis
# 上锁
timeout = current_time + lock_time + 1 
if (setnx lock:temp timeout || (now > get lock:temp && now > getset lock:temp timeout) {
	expire lock:temp timeout
	// 上锁成功...
}

# 解锁
if (now < get lock:temp) delete lock:temp
```

这样就完美解决之前死锁的问题。



### 计数器设计

*Redis* 的 `incr` 支持原子性自增操作，可以使用该命令快速实现一个简单的计数器。以音频播放量为例：

```redis
# 设置 key 为：counter:sound:id:1:view_count
incr key; # 每次访问的时候自增 1
```

但是在实际应用中，我们需要将具体的播放量持久化。这时候如果我们直接去取 key 的值就会可能导致数据不一致的情况。以下为一个数据不一致的例子：

| 并发1      | 并发2      | 持久化进程       | key 的值 | 持久化的播放量 | 记录的播放量 | 实际播放量 |
| -------- | -------- | ----------- | ------ | ------- | ------ | ----- |
| incr key |          |             | 1      | 1234    | 1235   | 1235  |
|          |          | get key     | 1      | 1234    | 1235   | 1235  |
|          | incr key |             | 2      | 1234    | 1236   | 1236  |
|          |          | del key     | null   | 1234    | 1234   | 1236  |
|          |          | Persistence | 0      | 1235    | 1235   | 1236  |
|          | incr key |             | 1      | 1235    | 1236   | 1237  |

从上面的例子易知，在高并发的情况下，会导致播放量数据的丢失。其原因在于持久化进程在进行持久化过程时，获取和重置并不是原子操作，中间可能会有其他进程对播放量的数据进行修改。由此可知，只需要保证持久化进程的获取和重置操作为原子操作，就能保证持久化时的数据一致性。以下提供两个解决方案可供参考。

当我们只有一个持久化进程时，我们可以对计数器 key 进行分段（segment）。我们只对不会再有其他进程进行修改的计数器进行持久化操作。例如我们在设计 key 时，在后面加上当日的日期或当日零点的时间戳。我们只对昨日的计数器进行持久化操作。具体代码如下。

```redis
# 设置当日 key1 为：counter:sound:id:1:date:20170808:view_count
# 设置昨日 key2 为：counter:sound:id:1:date:20170807:view_count
incr key1;
get key2;
del key2;
```

另一种情况为我们拥有多个持久化进程，或者本身计数进程就有持久化功能。这时候我们可以使用 `getset` 命令：

```redis
# 设置 key 为：counter:sound:id:1:view_count
incr key;
getset key 0;
```

在对大量连续 `key` 的数据进行计数时，我们可以使用 `hash ` 来替代简单的 `string` 。因为 `hash` 结构会对其中的元素进行压缩存储，从而节约大量内存。而且经测试，当 `hash field` 的数量小于 1000 时性能最优。下面我们看一个例子，对音频的播放量进行缓存计数：

最开始，我们使用 `string` 来计数，语句为 `incr counter:sound_views:id:123456`。

当使用 `hash` 优化时，我们可以使用 `HINCRBY counter:sound_views:segids:123 456 1` 进行替代。同样在进行持久化的时候，我们需要考虑数据一致性问题。



### 注意事项

scan 可能会存在重复的值，中途添加或删除的键则不保证其是否遍历到。

randomkey 随机返回一个key

pipeline 不等待返回