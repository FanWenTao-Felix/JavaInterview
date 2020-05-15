###  **redis 都有哪些数据类型？分别在哪些场景下使用比较合适？** 

​	redis 主要有以下几种数据类型：

- string
- hash
- list
- set
- sorted set

### string

​	这是最简单的类型，就是普通的 set 和 get，做简单的 KV 缓存。

```bash
set college szuCopy to clipboardErrorCopied
```

### hash

​	这个是类似 map 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 redis 里，然后每次读写缓存的时候，可以就操作 hash 里的**某个字段**。

```bash
hset person name bingo
hset person age 20
hset person id 1
hget person nameCopy to clipboardErrorCopied
person = {
    "name": "bingo",
    "age": 20,
    "id": 1
}Copy to clipboardErrorCopied
```

### list

​	list 是有序列表，这个可以玩儿出很多花样。

​	比如可以通过 list 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

​	比如可以通过 lrange 命令，读取某个闭区间内的元素，可以基于 list 实现分页查询，这个是很棒的一个功能，基于 redis 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

```bash
# 0开始位置，-1结束位置，结束位置为-1时，表示列表的最后一个位置，即查看所有。
lrange mylist 0 -1Copy to clipboardErrorCopied
```

比如可以搞个简单的消息队列，从 list 头怼进去，从 list 尾巴那里弄出来。

```bash
lpush mylist 1
lpush mylist 2
lpush mylist 3 4 5

# 1
rpop mylistCopy to clipboardErrorCopied
```

### set

​	set 是无序集合，自动去重。

​	直接基于 set 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 jvm 内存里的 HashSet 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于 redis 进行全局的 set 去重。

​	可以基于 set 玩儿交集、并集、差集的操作，比如交集吧，可以把两个人的粉丝列表整一个交集，看看俩人的共同好友是谁？对吧。

​	把两个大 V 的粉丝都放在两个 set 中，对两个 set 做交集。

```bash
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
sdiff yourSet mySetCopy to clipboardErrorCopied
```

### sorted set

​	sorted set 是排序的 set，去重但可以排序，写进去的时候给一个分数，自动根据分数排序。

```bash
zadd board 85 zhangsan
zadd board 72 lisi
zadd board 96 wangwu
zadd board 63 zhaoliu

# 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
zrevrange board 0 3

# 获取某用户的排名
zrank board zhaoliu
```

