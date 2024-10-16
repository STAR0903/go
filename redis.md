# redis

#### 笔记参考

`https://blog.csdn.net/HJW_233/article/details/131902164`

#### 常用命令

```
Redis默认有16个数据库（从 0 到 15），切换到第2个数据库
select 1
```

```
查看当前数据库key的数量
DBSIZE
```

```
设置一个key为username，值为mike的数据
set username mike
```

```
获取key为username的值
get username
```

```
获取所有的key
keys *
```

```
清除当前数据库
flushdb
```

```
清除所有数据库
flushall
```

```
查询key为username是否存在
exists username
```

```
指定key为username移动到1号数据库
move username 1
```

```
指定key为username10s后过期
expire username 10
```

```
查看key为username还有多久过期
 ttl username
```

```
查看key为username是什么类型
type username
```

#### 数据结构类型

###### String类型

```
0、删除key为name的值
del name
1、设置key为name的值为htt
set name htt
2、获取key为name的值
get name
3、拼接key为name的值：htt -> httstudy
append name study
4、获取key为name的值的长度
strlen name
5、设置key为view的值加1
incr view
6、设置key为view的值减1
decr view
7、设置key为view的值加10
incrby view 10
8、设置key为view的值减10
decrby view 10
9、截取下标为0-3之间的字符串，例如：abcdef，截取后abcd
getrange name 0 3
10、从下标为1进行替换字符串，例如：abcdef，替换后a000efg
setrange name 1 000
11、设置key为name的值为hello，10s后过期
setex name 10 hello
12、如果不存在key为title的，值设置为redis，如果存在，则set失败
setnx title redis
13、一次性设置多个值
mset k1 v1 k2 v2 k3 v3
mset user:1:name htt user:1:age 2
14、一次性获取多个值
mget k1 k2 k3
mget user:1:name user:1:age
15、如果k1已经存在，则k1，k4全部设置失败，参考事务的原子性操作
msetnx k1 v1 k4 v4
16、如果不存在key为username的值，则返回nil，然后set进去；如果存在值，则获取原来的值并设置新的值
getset username htt
```

###### List集合类型

List按插入顺序排序的字符串列表，可以在列表的头部（左边）或尾部（右边）添加元素

```
1、将一个值或者多个值插入到key为list的list列表的头部
lpush list 1
2、将一个值或者多个值插入到key为list的list列表的尾部
rpush list 4
3、通过区间获取具体的值
lrange key start stop (stop=-1时表示最后一个元素)
lrange list 0 -1
4、获取并且移除list的第一个元素
lpop list
5、获取并且移除list的最后一个元素
rpop list
6、通过下标获得list当中的某一个值
lindex list 0
7、获取list的长度
llen list
8、移除list集合指定个数的value，移除1个值为2的，精确匹配
lrem list 1 2
9、截取list集合中下标为1到下标为2之间的元素集合，并覆盖原来的list集合
ltrim list 1 2
10、更新list集合当中下标为0的值为bbb，如果下标0的值不存在，则报错
lset list 0 bbb
11、将一个某一个具体的值[aaa]插入到某一个具体元素（如果该元素有多个，默认第一个）[kkk]的前面或者后面
linsert list BEFORE kkk aaa
linsert list AFTER kkk aaa
```

###### Set集合类型

Set 是 string 类型的无序集合，集合成员是唯一的（集合中没有重复的数据）

不管 Set 中包含多少元素，添加、删除和查找的时间复杂都是 O(1)

```
1、往key为set的set集合中添加一个元素
sadd set hello
2、查看key为set的set集合中所有元素
smembers set
3、查看key为set的set集合中是否存在某元素
sismember set world
4、随机抽取出1个元素
srandmember set
5、随机抽取出2个元素
srandmember set 2
6、随机删除set集合中某个元素
spop set
7、移动set集合中的world元素到set2集合中
smove set set2 world
8、作set2集合减去set集合的差集
sdiff set2 set
9、set和set2的交集
sinter set set2
10、set和set2作并集并去重
sunion set set2
```

###### Hash集合类型

hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象

```
1、往key为hash的hash集合中存放键值对数据
hset hash username mike
2、从key为hash的hash集合中获取数据
hget hash username
3、同时往hash集合中添加多个值
hmset hash username jack age 2
4、同时往hash集合中获取多个值
hmget hash username age
5、获取hash集合中所有的键值对
hgetall hash
6、删除hash集合中指定的key字段
hdel hash username
7、获取hash集合的长度
hlen hash
8、判断hash集合中指定字段是否存在
hexists hash username
9、获取hash集合中所有的key
hvals hash
10、获取hash集合中所有的值
hkeys hash
11、指定hash集合中指定增量
hincrby hash views 1
12、如果不存在则直接设置值，存在则设置失败
hsetnx hash password 123456
```

###### Zset有序集合类型

Zset是 string 类型元素的集合，且不允许重复的成员

每个元素都会关联一个 double 类型的分数。Redis 正是通过分数来为集合中的成员进行从小到大的排序，有序集合的成员是唯一的,但分数 ( score ) 却可以重复

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)

```
1、添加一个值
zadd zset 1 first
2、添加多个值
zadd key score1 member1 score2 member2
zadd zset 2 second 3 third 4 four
3、获取zset集合中所有元素
zrange zset 0 -1
4、给zset集合中的元素从小到大排序，-inf：负无穷，+inf：正无穷
zrangebyscore zset -inf +inf
5、从小到大排序并输出键值
zrangebyscore zset -inf +inf withscores
[limit]是一个可选参数，用于限制返回结果的数量，offset 是起始位置，count 是返回的成员数量
zrangebyscore key min max [withscores] [limit offset count]
6、指定负无穷到1的范围
zrangebyscore zset -inf 1 withscores
7、移除zset集合中指定的元素
zrem zset four
8、查看zset集合中元素个数
zcard zset
9、反转指定范围
zrevrange zset 1 2
```
