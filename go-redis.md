# go-redis

#### 准备环境

```
docker run --name redis001 -p 6379:6379 -d redis
docker run -it --network host --rm redis redis-cli
```

#### 下载依赖

`import "github.com/redis/go-redis/v9"`

#### 连接

###### 普通连接

```
func initClient() error {
	//redis.NewClient 函数连接 Redis 服务器
	rdb = redis.NewClient(&redis.Options{
		Addr:     "127.0.0.1:6379",
		Password: "",
		DB:       0,
		PoolSize: 20,
	})
	ctx := context.Background()
	_, err := rdb.Ping(ctx).Result()
	return err
}
```

```
func initClient() error {
	//redis.ParseURL("redis://<user>:<pass>@localhost:6379/<db>")
	opt, err := redis.ParseURL("redis://:@localhost:6379/22")
	if err != nil {
		return err
	}
	rdb = redis.NewClient(opt)
	return nil
}
```

```
// 声明变量
var rdb *redis.Client

func initClient() error {
	......
}

func main() {
	if err := initClient(); err != nil {
		fmt.Println("init client failed", err)
	}
	fmt.Println("init client success")
	//程序结束回收
	defer rdb.Close()
}
```

###### Redis Sentinel模式

使用下面的命令连接到由 Redis Sentinel 管理的 Redis 服务器

```
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "master-name",
    SentinelAddrs: []string{":9126", ":9127", ":9128"},
})

```

###### Redis Cluster模式

使用下面的命令连接到 Redis Cluster，go-redis 支持按延迟或随机路由命令

```
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},

    // 若要根据延迟或随机路由命令，请启用以下命令之一
    // RouteByLatency: true,
    // RouteRandomly: true,
})
```

#### 基本运用

###### 执行命令

```
context.WithTimeout() 这个函数创建了一个带有超时的上下文
context.Background() 创建了一个无根 context
context.WithTimeout() 在这个 context 上添加了一个超时，这里设置的超时时间为 500 毫秒
defer cancel() 确保在函数执行结束前调用 cancel() 函数
cancel() 这个函数会取消上下文，停止任何正在等待的协程
```

```
func doCommand() {

	//初始化ctx
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// 执行命令获取结果
	val, err := rdb.Get(ctx, "key").Result()
	fmt.Println("val:", val)
	fmt.Println("err:", err)

	// 先获取到命令对象
	cmder := rdb.Get(ctx, "key")
	fmt.Println("val:", cmder.Val())
	fmt.Println("err:", cmder.Err()) 

	// 直接执行命令获取错误   time.Hour:过期时间
	err = rdb.Set(ctx, "key", 10, time.Hour).Err()
	fmt.Println("err:", err)

	// 直接执行命令获取值
	value := rdb.Get(ctx, "key").Val()
	fmt.Println("val:", value)

}
```

###### zset相关函数

定义与作用

```
//redis.Z用于表示Zset中元素
type Z struct {
    Score  float64
    Member interface{}
}
//向 Redis 中的有序集合key中添加成员
func (c redis.cmdable) ZAdd(ctx context.Context, key string, members ...redis.Z) *redis.IntCmd
//对有序集合key中特定成员member的分数增加increment
func (c redis.cmdable) ZIncrBy(ctx context.Context, key string, increment float64, member string) *redis.FloatCmd
//按照分数从高到低的顺序获取有序集合key指定范围内[start,stop)的成员及其分数
func (c redis.cmdable) ZRevRangeWithScores(ctx context.Context, key string, start int64, stop int64) *redis.ZSliceCmd
//按照分数从低到高的顺序获取有序集合key指定范围内[Min，Max]的从下标为Offset的成员开始的Count个成员及其分数
func (c redis.cmdable) ZRangeByScoreWithScores(ctx context.Context, key string, opt *redis.ZRangeBy) *redis.ZSliceCmd
//redis.ZRangeBy
type ZRangeBy struct {
    Min, Max      string
    Offset, Count int64
}
```

示例

```
func zsetDemo() {

	// key
	zsetKey := "language"
	// value
	languages := []redis.Z{
		{Score: 90.0, Member: "Golang"},
		{Score: 98.0, Member: "Java"},
		{Score: 95.0, Member: "Python"},
		{Score: 97.0, Member: "JavaScript"},
		{Score: 99.0, Member: "C/C++"},
	}

	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	// ZAdd
	err := rdb.ZAdd(ctx, zsetKey, languages...).Err()
	if err != nil {
		fmt.Printf("zadd failed, err:%v\n", err)
		return
	}
	fmt.Println("zadd success")

	// 把Golang的分数加10
	newScore, err := rdb.ZIncrBy(ctx, zsetKey, 10.0, "Golang").Result()
	if err != nil {
		fmt.Printf("zincrby failed, err:%v\n", err)
		return
	}
	fmt.Printf("Golang's score is %f now.\n", newScore)

	// 取分数最高的3个
	ret := rdb.ZRevRangeWithScores(ctx, zsetKey, 0, 2).Val()
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
	fmt.Println("-----------------------------------")

	// 取95~100分的
	op := &redis.ZRangeBy{
		Min: "95",
		Max: "100",
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx, zsetKey, op).Result()
	if err != nil {
		fmt.Printf("zrangebyscore failed, err:%v\n", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
	fmt.Println("-----------------------------------")

	// 取95~100分的从下标为3开始取，取2个
	op2 := &redis.ZRangeBy{
		Min:    "95",
		Max:    "100",
		Offset: 3,
		Count:  2,
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx, zsetKey, op2).Result()
	if err != nil {
		fmt.Printf("zrangebyscore failed, err:%v\n", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
}
```

###### 执行任意命令

```
func doCommand() {

	//初始化ctx
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	//第二个参数是命令名，如 SET、GET、INCR
	//后续参数是命令需要的参数，例如对于 SET 命令，需要提供键名和值
	val, err := rdb.Do(ctx, "hset", "user1", "username", "cx111", "age", 18, "hobby", "read").Result()
	fmt.Println(val)
	DealError(err)
	val, err = rdb.Do(ctx, "hgetall", "user1").Result()
	fmt.Println(val)
	DealError(err)

}
```

###### redis.Nil

```
go-redis 库提供了一个 redis.Nil 错误来表示 Key 不存在的错误。因此在使用 go-redis 时需要注意对返回错误的判断。在某些场景下我们应该区别处理 redis.Nil 和其他不为 nil 的错误
```

```
// 错误处理
func DealError(err error) {
	if err == nil {
		return
	} else {
		if err == redis.Nil {
			fmt.Println("key不存在")
		} else {
			fmt.Println(err)
		}
	}
}
```

#### key相关的批量操作

###### `Keys ( )`

```
在Redis中可以使用KEYS prefix* 命令按前缀查询所有符合条件的 key，go-redis库中提供了Keys方法实现类似查询key的功能
func (c redis.cmdable) Keys(ctx context.Context, pattern string) *redis.StringSliceCmd
```

```
例如使用以下命令查询以user:为前缀的所有key（user:cart:00、user:order:2023等）
vals, err := rdb.Keys(ctx, "user:*").Result()
```

###### `Scan ( )`

```
// scanKeysDemo1 按前缀查找所有key示例
func scanKeysDemo1() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	var cursor uint64
	for {
		var keys []string
		var err error
		// 将redis中所有以prefix:为前缀的key都扫描出来(可以分多次)
		keys, cursor, err = rdb.Scan(ctx, cursor, "prefix:*", 0).Result()
		if err != nil {
			panic(err)
		}

		for _, key := range keys {
			fmt.Println("key", key)
		}

		if cursor == 0 { // no more keys
			break
		}
	}
}
```

```
iter := rdb.SScan(ctx, "set-key", 0, "prefix:*", 0).Iterator()
iter := rdb.HScan(ctx, "hash-key", 0, "prefix:*", 0).Iterator()
iter := rdb.ZScan(ctx, "sorted-hash-key", 0, "prefix:*", 0).Iterator(
```

###### `Iterator ( )`

针对这种需要遍历大量key的场景，`go-redis`中提供了一个简化方法——`Iterator`，其使用示例如下

```
// scanKeysDemo2 按前缀扫描key示例
func scanKeysDemo2() {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()
	// 按前缀扫描key
	iter := rdb.Scan(ctx, 0, "prefix:*", 0).Iterator()
	for iter.Next(ctx) {
		fmt.Println("keys", iter.Val())
	}
	if err := iter.Err(); err != nil {
		panic(err)
	}
}

```

#### Pipeline

Redis Pipeline 允许通过使用单个 client-server-client 往返执行多个命令来提高性能。区别于一个接一个地执行100个命令，你可以将这些命令放入 pipeline 中，然后使用1次读写操作像执行单个命令一样执行它们。这样做的好处是节省了执行命令的网络往返时间（RTT）

在下面的示例代码中演示了使用 pipeline 通过一个 write + read 操作来执行多个命令

```
func Demo() {

	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	pipe := rdb.Pipeline()

	a := pipe.HSet(ctx, "myworld", "want", "happy", "love", "go")
	b := pipe.HGetAll(ctx, "myworld")

	cmds, err := pipe.Exec(ctx)
	if err != nil {
		panic(err)
	}

	// 在执行pipe.Exec之后才能获取到结果
	fmt.Println(cmds)
	fmt.Println(a.Val())
	fmt.Println(b.Val())

}
```

#### 事务

Redis 是单线程执行命令的，因此单个命令始终是原子的，但是来自不同客户端的两个给定命令可以依次执行，例如在它们之间交替执行。但是，`Multi/exec`能够确保在 `multi/exec`两个语句之间的命令之间没有其他客户端正在执行命令。

在这种场景我们需要使用 TxPipeline 或 TxPipelined 方法将 pipeline 命令使用 `MULTI` 和 `EXEC`包裹起来

```
// TxPipeline demo
pipe := rdb.TxPipeline()
incr := pipe.Incr(ctx, "tx_pipeline_counter")
pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
_, err := pipe.Exec(ctx)
fmt.Println(incr.Val(), err)

// TxPipelined demo
var incr2 *redis.IntCmd
_, err = rdb.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
	incr2 = pipe.Incr(ctx, "tx_pipeline_counter")
	pipe.Expire(ctx, "tx_pipeline_counter", time.Hour)
	return nil
})
fmt.Println(incr2.Val(), err)
```

上面代码相当于在一个RTT下执行了下面的redis命令：

```
MULTI
INCR pipeline_counter
EXPIRE pipeline_counts 3600
EXEC
```

#### Watch

我们通常搭配 `WATCH`命令来执行事务操作。从使用 `WATCH`命令监视某个 key 开始，直到执行 `EXEC`命令的这段时间里，如果有其他用户抢先对被监视的 key 进行了替换、更新、删除等操作，那么当用户尝试执行 `EXEC`的时候，事务将失败并返回一个错误，用户可以根据这个错误选择重试事务或者放弃事务。

Watch方法接收一个函数和一个或多个key作为参数

`Watch(fn func(*Tx) error, keys ...string) error `

下面的代码片段演示了 Watch 方法搭配 TxPipelined 的使用示例

```
// watchDemo 在key值不变的情况下将其值+1
func watchDemo(ctx context.Context, key string) error {
	return rdb.Watch(ctx, func(tx *redis.Tx) error {
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		// 假设操作耗时5秒
		// 5秒内我们通过其他的客户端修改key，当前事务就会失败
		time.Sleep(5 * time.Second)
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			pipe.Set(ctx, key, n+1, time.Hour)
			return nil
		})
		return err
	}, key)
}
```

将上面的函数执行并打印其返回值，如果我们在程序运行后的5秒内修改了被 watch 的 key 的值，那么该事务操作失败，返回 `redis: transaction failed`错误。

最后我们来看一个 go-redis 官方文档中使用 `GET` 、`SET`和 `WATCH`命令实现一个 INCR 命令的完整示例

```
// 此处rdb为初始化的redis连接客户端
const routineCount = 100

// 设置5秒超时
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// increment 是一个自定义对key进行递增（+1）的函数
// 使用 GET + SET + WATCH 实现，类似 INCR
increment := func(key string) error {
	txf := func(tx *redis.Tx) error {
		// 获得当前值或零值
		n, err := tx.Get(ctx, key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		// 实际操作（乐观锁定中的本地操作）
		n++

		// 仅在监视的Key保持不变的情况下运行
		_, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
			// pipe 处理错误情况
			pipe.Set(ctx, key, n, 0)
			return nil
		})
		return err
	}

	// 最多重试100次
	for retries := routineCount; retries > 0; retries-- {
		err := rdb.Watch(ctx, txf, key)
		if err != redis.TxFailedErr {
			return err
		}
		// 乐观锁丢失
	}
	return errors.New("increment reached maximum number of retries")
}

// 开启100个goroutine并发调用increment
// 相当于对key执行100次递增
var wg sync.WaitGroup
wg.Add(routineCount)
for i := 0; i < routineCount; i++ {
	go func() {
		defer wg.Done()

		if err := increment("counter3"); err != nil {
			fmt.Println("increment error:", err)
		}
	}()
}
wg.Wait()

n, err := rdb.Get(ctx, "counter3").Int()
fmt.Println("最终结果：", n, err)
```

在这个示例中使用了 `redis.TxFailedErr` 来检查事务是否失败
