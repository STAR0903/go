# 介绍

[`cron`](https://link.segmentfault.com/?enc=NxJoGE84AH5XoP3WUUoxNw%3D%3D.krapOva%2Fl%2FJ%2BiLPBRL9LCXjfWhIM%2BTH4XhLZwMFsvhQ%3D)一个用于管理定时任务的库，用 Go 实现 Linux 中 `crontab`这个命令的效果。

[学习更多](https://segmentfault.com/a/1190000023029219)：指定时区，自定义 `Logger`，Job 包装器......

# 快速上手

```
package main

import (
	"fmt"
	"github.com/robfig/cron/v3"
)

func main() {
	job := cron.New()
	job.AddFunc("@every 1s", func() {
		fmt.Println("hello world !")
	})
	job.Start()
	select {}
}
```

该示例代码运行后，每秒打印一次。

# 时间表达式

### 时间格式

与Linux 中 `crontab`命令相似，`cron`库支持用 **5** 个空格分隔的域来表示时间。这 5 个域含义依次为：

* `Minutes`：分钟，取值范围 `[0-59]`，支持特殊字符 `* / , -`；
* `Hours`：小时，取值范围 `[0-23]`，支持特殊字符 `* / , -`；
* `Day of month`：每月的第几天，取值范围 `[1-31]`，支持特殊字符 `* / , - ?`；
* `Month`：月，取值范围 `[1-12]`或者使用月份名字缩写 `[JAN-DEC]`，支持特殊字符 `* / , -`；
* `Day of week`：周历，取值范围 `[0-6]`或名字缩写 `[JUN-SAT]`，支持特殊字符 `* / , - ?`。

注意，月份和周历名称都是不区分大小写的，也就是说 `SUN/Sun/sun`表示同样的含义（都是周日）。

特殊字符含义如下：

* `*`：使用 `*`的域可以匹配任何值。 例如将月份域设置为 `*`，表示每个月；
* `/`：用来指定范围的 **步长** 。例如将小时域设置为 `3-59/15`表示第 3 分钟触发，以后每隔 15 分钟触发一次，因此第 2 次触发为第 18 分钟，第 3 次为 33 分钟...直到分钟大于 59；
* `,`：用来列举一些离散的值和多个范围。例如将周历的域（第 5 个）设置为 `MON,WED,FRI`表示周一、三和五；
* `-`：用来表示范围。例如将小时的域（第 1 个）设置为 `9-17`表示上午 9 点到下午 17 点（包括 9 和 17）；
* `?`：只能用在月历和周历的域中，用来代替 `*`，表示每月/周的任意一天。

  ```
  示例：
  30 * * * *
  分钟域为 30，其他域都是*表示任意。每小时的 30 分触发；
  30 3-6,20-23 * * *
  分钟域为 30，小时域的3-6,20-23表示 3 点到 6 点和 20 点到 23 点。3,4,5,6,20,21,22,23 时的 30 分触发；
  0 0 1 1 *
  1月 1 号的 0 时 0 分触发
  ```

##### 代码演示

```
func main() {
  c := cron.New()

  c.AddFunc("30 * * * *", func() {
    fmt.Println("Every hour on the half hour")
  })

  c.AddFunc("30 3-6,20-23 * * *", func() {
    fmt.Println("On the half hour of 3-6am, 8-11pm")
  })

  c.AddFunc("0 0 1 1 *", func() {
    fmt.Println("Jun 1 every year")
  })

  c.Start()

  for {
    time.Sleep(time.Second)
  }
}
```

### 预定义时间规则

为了方便使用，`cron`预定义了一些时间规则：

* `@yearly`：也可以写作 `@annually`，表示每年第一天的 0 点。等价于 `0 0 1 1 *`；
* `@monthly`：表示每月第一天的 0 点。等价于 `0 0 1 * *`；
* `@weekly`：表示每周第一天的 0 点，注意第一天为周日，即周六结束，周日开始的那个 0 点。等价于 `0 0 * * 0`；
* `@daily`：也可以写作 `@midnight`，表示每天 0 点。等价于 `0 0 * * *`；
* `@hourly`：表示每小时的开始。等价于 `0 * * * *`。

##### 代码演示

```
func main() {
  c := cron.New()

  c.AddFunc("@hourly", func() {
    fmt.Println("Every hour")
  })

  c.AddFunc("@daily", func() {
    fmt.Println("Every day on midnight")
  })

  c.AddFunc("@weekly", func() {
    fmt.Println("Every week")
  })

  c.Start()

  for {
    time.Sleep(time.Second)
  }
}
```

### 固定时间间隔

`cron`支持固定时间间隔，格式为：

```
@every <duration>
```

含义为每隔 `duration`触发一次。`<duration>`会调用 `time.ParseDuration()`函数解析，所以 `ParseDuration`支持的格式都可以。例如 `1h30m10s`。在**快速上手**部分，我们已经演示了 `@every`的用法了，这里就不赘述了。

# Job接口

除了直接将无参函数作为回调外，`cron`还支持 `Job`接口：

```
#cron.go
type Job interface {
  Run()
}
```

我们定义一个实现接口 `Job`的结构：

```
type GreetingJob struct {
  Name string
}

func (g GreetingJob) Run() {
  fmt.Println("Hello ", g.Name)
}
```

调用 `cron`对象的 `AddJob()`方法将 `GreetingJob`对象添加到定时管理器中：

```
func main() {
  c := cron.New()
  c.AddJob("@every 1s", GreetingJob{"dj"})
  c.Start()

  time.Sleep(5 * time.Second)
}
```

```
Output：
Hello  dj
Hello  dj
Hello  dj
Hello  dj
Hello  dj
```

使用自定义的结构可以让任务携带状态（`Name`字段）。

### 源码分析

实际上 `AddFunc()`方法内部也调用了 `AddJob()`方法。首先，`cron`基于 `func()`类型定义一个新的类型 `FuncJob`：

```
type FuncJob func()
```

然后让 `FuncJob`实现 `Job`接口：

```
func (f FuncJob) Run() {
  f()
}
```

在 `AddFunc()`方法中，将传入的回调转为 `FuncJob`类型，然后调用 `AddJob()`方法：

```
func (c *Cron) AddFunc(spec string, cmd func()) (EntryID, error) {
  return c.AddJob(spec, FuncJob(cmd))
}
```
