# Viper

## 安装

`go get github.com/spf13/viper `

## 什么是Viper？

Viper是适用于Go应用程序（包括 `Twelve-Factor App`）的完整配置解决方案。它被设计用于在应用程序中工作，并且可以处理所有类型的配置需求和格式。它支持以下特性：

* 设置默认值
* 从 `JSON`、`TOML`、`YAML`、`HCL`、`envfile`和 `Java properties`格式的配置文件读取配置信息
* 实时监控和重新读取配置文件（可选）
* 从环境变量中读取
* 从远程配置系统（etcd或Consul）读取并监控配置变化
* 从命令行参数读取配置
* 从buffer读取配置
* 显式配置值

## 为什么选择Viper?

在构建现代应用程序时，你无需担心配置文件格式；你想要专注于构建出色的软件。Viper的出现就是为了在这方面帮助你的。

Viper能够为你执行下列操作：

1. 查找、加载和反序列化 `JSON`、`TOML`、`YAML`、`HCL`、`INI`、`envfile`和 `Java properties`格式的配置文件。
2. 提供一种机制为你的不同配置选项设置默认值。
3. 提供一种机制来通过命令行参数覆盖指定选项的值。
4. 提供别名系统，以便在不破坏现有代码的情况下轻松重命名参数。
5. 当用户提供了与默认值相同的命令行或配置文件时，可以很容易地分辨出它们之间的区别。

Viper会按照下面的优先级。每个项目的优先级都高于它下面的项目:

* 显示调用 `Set`设置值
* 命令行参数（flag）
* 环境变量
* 配置文件
* key/value存储
* 默认值

**重要：** 目前Viper配置的键（Key）是大小写不敏感的。目前正在讨论是否将这一选项设为可选。

#### 建立默认值

一个好的配置系统应该支持默认值。键不需要默认值，但如果没有通过配置文件、环境变量、远程配置或命令行标志（flag）设置键，则默认值非常有用。

例如：

`func viper.SetDefault(key string, value any)`

`SetDefault` 为这个键设置默认值。`SetDefault` 对于键是不区分大小写的。只有在用户未通过标志、配置或环境变量提供值时，才会使用默认值。

```
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

#### 读取配置文件

Viper需要最少知道在哪里查找配置文件的配置。Viper支持 `JSON`、`TOML`、`YAML`、`HCL`、`envfile`和 `Java properties`格式的配置文件。Viper可以搜索多个路径，但目前单个Viper实例只支持单个配置文件。Viper不默认任何配置搜索路径，将默认决策留给应用程序。

下面是一个如何使用Viper搜索和读取配置文件的示例。不需要任何特定的路径，但是至少应该提供一个配置文件预期出现的路径。

```
viper.SetConfigFile("./config.yaml") // 指定配置文件路径
viper.SetConfigName("config") // 配置文件名称(无扩展名)
viper.SetConfigType("yaml") // 如果配置文件的名称中没有扩展名，则需要配置此项
viper.AddConfigPath("/etc/appname/")   // 查找配置文件所在的路径
viper.AddConfigPath("$HOME/.appname")  // 多次调用以添加多个搜索路径
viper.AddConfigPath(".")               // 还可以在工作目录中查找配置
err := viper.ReadInConfig() // 查找并读取配置文件
if err != nil { // 处理读取配置文件的错误
	panic(fmt.Errorf("Fatal error config file: %s \n", err))
}
```

```
func viper.SetConfigFile(in string)
SetConfigFile 明确界定了配置文件的路径、名称以及扩展名。Viper 将采用此设定，并且不会检查任何配置路径。 
func viper.SetConfigName(in string)
配置文件名称,不包括扩展名
func viper.SetConfigType(in string)
当配置文件的名称中没有扩展名时，通过此函数明确配置文件的类型
func viper.AddConfigPath(in string)
添加一个搜索配置文件的路径，可以多次调用
func viper.ReadInConfig() error
查找并读取配置文件
```

在加载配置文件出错时，你可以像下面这样处理找不到配置文件的特定情况：

```
if err := viper.ReadInConfig(); err != nil {
    if _, ok := err.(viper.ConfigFileNotFoundError); ok {
        // 配置文件未找到错误；如果需要可以忽略
    } else {
        // 配置文件被找到，但产生了另外的错误
    }
}// 配置文件找到并成功解析
```

思考

当你使用如下方式读取配置时，viper会从 `./conf`目录下查找任何以 `config`为文件名的配置文件，如果同时存在 `./conf/config.json`和 `./conf/config.yaml`两个配置文件的话，`viper`会从哪个配置文件加载配置呢？

```
viper.SetConfigName("config")
viper.AddConfigPath("./conf")
```

`答案：./conf/config.json`

在上面两个语句下搭配使用 `viper.SetConfigType("yaml")`指定配置文件类型可以实现预期的效果吗？

`答案：不行`

```
解释：viper.ReadInConfig()查找并读取配置文件时只关注了viper.SetConfigName()和viper.AddConfigPath()设定的相关内容
```

#### 写入配置文件

从配置文件中读取配置文件是有用的，但是有时你想要存储在运行时所做的所有修改。为此，可以使用下面一组命令，每个命令都有自己的用途:

```
WriteConfig 
将当前的 viper配置写入预定义的路径并覆盖（如果存在的话）。如果没有预定义的路径，则报错。
SafeWriteConfig 
将当前的 viper配置写入预定义的路径。如果没有预定义的路径，则报错。如果存在，将不会覆盖当前的配置文件。WriteConfigAs
将当前的 viper配置写入给定的文件路径。将覆盖给定的文件(如果它存在的话)。
SafeWriteConfigAs 
将当前的 viper配置写入给定的文件路径。不会覆盖给定的文件，会返回一个错误信息，但是不影响程序运行(如果它存在的话)。
```

根据经验，标记为 `safe`的所有方法都不会覆盖任何文件，而是直接创建（如果不存在），而默认行为是创建或截断。

一个小示例：

```
// 将当前配置写入“viper.AddConfigPath()”和“viper.SetConfigName”设置的预定义路径
viper.WriteConfig()
viper.SafeWriteConfig()
viper.WriteConfigAs("/path/to/my/.config")
viper.SafeWriteConfigAs("/path/to/my/.config") 
viper.SafeWriteConfigAs("/path/to/my/.other_config")
```

#### 监控并重新读取配置文件

`Viper`支持在运行时实时读取配置文件的功能。

需要重新启动服务器以使配置生效的日子已经一去不复返了，通过 `watchConfig`， `viper`驱动的应用程序可以在运行时读取配置文件的更新，而不会错过任何消息。

可选地，你可以为Viper提供一个回调函数`（OnConfigChange）`，以便在每次发生更改时运行。

**确保在调用 `WatchConfig()`之前添加了所有的配置路径。**

```
func main() {
	viper.SetConfigName("config")
	viper.AddConfigPath("./")
	if err := viper.ReadInConfig(); err != nil {
		panic(err)
	}
	viper.WatchConfig()
	//为什么改变（或者说保存）一次打印两次
	//编译器问题，使用记事本会只打印一次，但是不改变仅仅保存也会打印一次  
	viper.OnConfigChange(func(in fsnotify.Event) {
		fmt.Println("changed!!!", in.Op.String())
	})
	r := gin.Default()
	r.GET("/user", func(ctx *gin.Context) {
		ctx.String(http.StatusOK, viper.GetString("user"))
	})
	r.Run()
}
```

```
// md5对字符串加密，即使改变一个字节的内容MD5也有很大区别
// GetMD5 获取byte对应MD5
func GetMD5(s []byte) string {
   m := md5.New()
   m.Write([]byte(s))
   return hex.EncodeToString(m.Sum(nil))
}

// 获取文件byte
func ReadFileMd5(sfile string) (string, error) {
   ssconfig, err := os.ReadFile(sfile)
   if err != nil {
      return "", err
   }
   return GetMD5(ssconfig), nil
}

func Init() {
    // 设置配置文件路径
    filepath := "xxxx"
    viper.SetConfigFile(filepath)

    // 获取文件MD5
    confMD5, err := ReadFileMd5(filepath)
    if err != nil {
        log.Fatal(err)
    }
    // 读取配置文件
    if err := viper.ReadInConfig(); err != nil {
        log.Fatal(err)
    }

    // 设置监控文件
    viper.WatchConfig()

    // 设置配置文件修改回调
    viper.OnConfigChange(func(e fsnotify.Event) {
        // 配置文件发生变更之后会调用的回调函数
        tconfMD5, err := ReadFileMd5(filepath)
        if err != nil {
            logger.Fatal(err)
        }
        // 比对当前MD5与之前是否相同
        if tconfMD5 == confMD5 {
            return
        }
        // 这说明文件发生了改变.
        confMD5 = tconfMD5
  
        log.Println("Config file changed!")
    })
}
```

#### 从io.Reader读取配置

Viper预先定义了许多配置源，如文件、环境变量、标志和远程K/V存储，但你不受其约束。你还可以实现自己所需的配置源并将其提供给viper。

```
viper.SetConfigType("yaml") // 或者 viper.SetConfigType("YAML")

// 任何需要将此配置添加到程序中的方法。
var yamlExample = []byte(`
Hacker: true
name: steve
hobbies:
- skateboarding
- snowboarding
- go
clothing:
  jacket: leather
  trousers: denim
age: 35
eyes : brown
beard: true
`)

viper.ReadConfig(bytes.NewBuffer(yamlExample))

viper.Get("name") // 这里会得到 "steve"
```

#### 覆盖设置

这些可能来自命令行标志，也可能来自你自己的应用程序逻辑。

```
#config.json
{"user":"cx111"}
#main.go
func main() {

	viper.SetConfigName("config")
	viper.AddConfigPath("./")

	err := viper.ReadInConfig() // 查找并读取配置文件

	if err != nil { // 处理读取配置文件的错误
		panic(err)
	}

	viper.Set("user", "set")

	str := viper.AllSettings()
	fmt.Println(str)

}
#out
map[user:set]
```

#### 注册和使用别名

别名允许多个键引用单个值： `viper.RegisterAlias`

```
func main() {

	viper.Set("user", "cx111")
	viper.RegisterAlias("user", "name")
	str := viper.GetString("name")
	fmt.Print(str)

}
```

```
#out:
cx111
```

#### 使用环境变量

Viper完全支持环境变量。这使 `Twelve-Factor App`开箱即用。有五种方法可以帮助与ENV协作:

```
func viper.AutomaticEnv()
AutomaticEnv 使 Viper 检查环境变量是否与任何现有的键（配置、默认值或标志）匹配。如果找到匹配的环境变量，它们将被加载到 Viper 中
func viper.BindEnv(input ...string) error
BindEnv 将一个 Viper 键绑定到一个环境变量。环境变量是区分大小写的。如果只提供了一个键，它将使用与该键匹配的环境键（大写形式）。如果提供了更多的参数，它们将表示应该绑定到此键的环境变量名，并按照指定的顺序获取。当未提供环境名称时，如果设置了 EnvPrefix ，则会使用它
func viper.SetEnvPrefix(in string)
SetEnvPrefix定义了环境变量将使用的前缀。例如，如果前缀是“spf”，那么环境注册表将查找以“SPF_”开头的环境变量
func viper.SetEnvKeyReplacer(r *strings.Replacer)
SetEnvKeyReplacer 在 Viper 对象上设置字符串替换器，可用于将环境变量映射到不匹配的键上
func viper.AllowEmptyEnv(allowEmptyEnv bool)
AllowEmptyEnv 指示 Viper 把已设置但为空的环境变量当作有效数值来考虑，而非回退。因为向后兼容性的缘故，其默认值是 false
```

**使用ENV变量时，务必要意识到Viper将ENV变量视为区分大小写。**

```
Viper提供了一种机制来确保ENV变量是惟一的。

通过使用 SetEnvPrefix，你可以告诉Viper在读取环境变量时使用前缀。BindEnv和 AutomaticEnv都将使用这个前缀。

BindEnv使用一个或两个参数。第一个参数是键名称，第二个是环境变量的名称。环境变量的名称区分大小写。如果没有提供ENV变量名，那么Viper将自动假设ENV变量与以下格式匹配：前缀+ “_” +键名全部大写。当你显式提供ENV变量名（第二个参数）时，它 不会 自动添加前缀。例如，如果第二个参数是“id”，Viper将查找环境变量“ID”。

在使用ENV变量时，需要注意的一件重要事情是，每次访问该值时都将读取它。Viper在调用 BindEnv时不固定该值。

AutomaticEnv是一个强大的助手，尤其是与 SetEnvPrefix结合使用时。调用时，Viper会在发出 viper.Get请求时随时检查环境变量。它将应用以下规则。它将检查环境变量的名称是否与键匹配（如果设置了 EnvPrefix）

SetEnvKeyReplacer允许你使用 strings.Replacer对象在一定程度上重写 Env 键。如果你希望在 Get()调用中使用 -或者其他什么符号，但是环境变量里使用 _分隔符，那么这个功能是非常有用的。可以在 viper_test.go中找到它的使用示例。

或者，你可以使用带有 NewWithOptions工厂函数的 EnvKeyReplacer。与 SetEnvKeyReplacer不同，它接受 StringReplacer接口，允许你编写自定义字符串替换逻辑。默认情况下，空环境变量被认为是未设置的，并将返回到下一个配置源。

若要将空环境变量视为已设置，请使用 AllowEmptyEnv方法。
```

```
func main() {

	viper.SetEnvPrefix("spf") // 将自动转为大写
	viper.BindEnv("id")

	os.Setenv("SPF_ID", "13") // 通常是在应用程序之外完成的

	id := viper.Get("id") // 13

	fmt.Println(id, viper.Get("id"))

	os.Setenv("SPF_ID", "17")

	fmt.Println(id, viper.Get("id"))

}
```

```
PS E:\Gin\procode\viper\demo6> go run ./
13 13
13 17
```

#### 使用Flags

Viper 具有绑定到标志的能力。具体来说，Viper支持[Cobra](https://github.com/spf13/cobra)库中使用的 `Pflag`。

与 `BindEnv`类似，该值不是在调用绑定方法时设置的，而是在访问该方法时设置的。这意味着你可以根据需要尽早进行绑定，即使在 `init()`函数中也是如此。

对于单个标志，`BindPFlag()`方法提供此功能。

例如：

```
serverCmd.Flags().Int("port", 1138, "Port to run Application server on")
viper.BindPFlag("port", serverCmd.Flags().Lookup("port"))
```

你还可以绑定一组现有的pflags （pflag.FlagSet）：

举个例子：

```
pflag.Int("flagname", 1234, "help message for flagname")

pflag.Parse()
viper.BindPFlags(pflag.CommandLine)

i := viper.GetInt("flagname") // 从viper而不是从pflag检索值
```

在 Viper 中使用 pflag 并不阻碍其他包中使用标准库中的 flag 包。pflag 包可以通过导入这些 flags 来处理flag包定义的flags。这是通过调用pflag包提供的便利函数 `AddGoFlagSet()`来实现的。

例如：

```
package main

import (
	"flag"
	"github.com/spf13/pflag"
)

func main() {

	// 使用标准库 "flag" 包
	flag.Int("flagname", 1234, "help message for flagname")

	pflag.CommandLine.AddGoFlagSet(flag.CommandLine)
	pflag.Parse()
	viper.BindPFlags(pflag.CommandLine)

	i := viper.GetInt("flagname") // 从 viper 检索值

	...
}
```

###### flag接口

如果你不使用 `Pflag`，Viper 提供了两个Go接口来绑定其他 flag 系统。

`FlagValue`表示单个flag。这是一个关于如何实现这个接口的非常简单的例子：

```
type myFlag struct {}
func (f myFlag) HasChanged() bool { return false }
func (f myFlag) Name() string { return "my-flag-name" }
func (f myFlag) ValueString() string { return "my-flag-value" }
func (f myFlag) ValueType() string { return "string" }
```

一旦你的 flag 实现了这个接口，你可以很方便地告诉Viper绑定它：

```
fSet := myFlagSet{
	flags: []myFlag{myFlag{}, myFlag{}},
}
viper.BindFlagValues("my-flags", fSet)
```

`FlagValueSet`代表一组 flags 。这是一个关于如何实现这个接口的非常简单的例子:

```
type myFlagSet struct {
	flags []myFlag
}

func (f myFlagSet) VisitAll(fn func(FlagValue)) {
	for _, flag := range flags {
		fn(flag)
	}
}
```

一旦你的flag set实现了这个接口，你就可以很方便地告诉Viper绑定它：

```
viper.BindFlagValue("my-flag-name", myFlag{})
```

#### 从Viper获取值

在Viper中，有几种方法可以根据值的类型获取值。存在以下功能和方法:

* `Get(key string) : interface{}`
* `GetBool(key string) : bool`
* `GetFloat64(key string) : float64`
* `GetInt(key string) : int`
* `GetIntSlice(key string) : []int`
* `GetString(key string) : string`
* `GetStringMap(key string) : map[string]interface{}`
* `GetStringMapString(key string) : map[string]string`
* `GetStringSlice(key string) : []string`
* `GetTime(key string) : time.Time`
* `GetDuration(key string) : time.Duration`
* `IsSet(key string) : bool`
* `AllSettings() : map[string]interface{}`

需要认识到的一件重要事情是，每一个Get方法在找不到值的时候都会返回零值。为了检查给定的键是否存在，提供了 `IsSet()`方法。

```
viper.Set("app", "qq")
if viper.IsSet("app") {
	fmt.Println(viper.GetString("app"))
}

if viper.IsSet("user") {
	fmt.Println(viper.GetString("user"))
} else {
	fmt.Println("no set")
}
```

###### 访问嵌套的键

访问器方法也接受深度嵌套键的格式化路径。例如，如果加载下面的JSON文件：

```
{
    "host": {
        "address": "localhost",
        "port": 5799
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}
```

Viper可以通过传入 `.`分隔的路径来访问嵌套字段：

```
GetString("datastore.metric.host") // (返回 "127.0.0.1")
```

这遵守上面建立的优先规则；搜索路径将遍历其余配置注册表，直到找到为止。(译注：因为Viper支持从多种配置来源，例如磁盘上的配置文件>命令行标志位>环境变量>远程Key/Value存储>默认值，我们在查找一个配置的时候如果在当前配置源中没找到，就会继续从后续的配置源查找，直到找到为止。)

例如，在给定此配置文件的情况下，`datastore.metric.host`和 `datastore.metric.port`均已定义（并且可以被覆盖）。如果另外在默认值中定义了 `datastore.metric.protocol`，Viper也会找到它。

然而，如果 `datastore.metric`被直接赋值覆盖（被flag，环境变量，`set()`方法等等…），那么 `datastore.metric`的**所有子键**都将变为未定义状态，它们被高优先级配置级别“遮蔽”（shadowed）了。

最后，如果存在与分隔的键路径匹配的键，则返回其值。

###### 提取子树

从Viper中提取子树。

例如，`viper`实例现在代表了以下配置：

```
app:
  cache1:
    max-items: 100
    item-size: 64
  cache2:
    max-items: 200
    item-size: 80
```


执行后：

```
subv := viper.Sub("app.cache1")
```

`subv`现在就代表：

```
max-items: 100
item-size: 64
```

```
func main() {

	viper.SetConfigName("app")
	viper.AddConfigPath("./")

	err := viper.ReadInConfig()

	if err != nil {
		panic(err)
	}

	sub1 := viper.Sub("app.cache1")
	sub2 := viper.Sub("app.cache2")

	a1 := sub1.GetInt("max-items")
	a2 := sub2.GetInt("max-items")
	fmt.Println(a1, a2)

}
```

###### 反序列化

你还可以选择将所有或特定的值解析到结构体、map等。

有两种方法可以做到这一点：

* `Unmarshal(rawVal interface{}) : error`
* `UnmarshalKey(key string, rawVal interface{}) : error`

举个例子：

```
#main.go
type App struct {
	User   string
	CanDo  string `mapstructure:"can-do"`
	Want   string
	Latest string
	More   string
}
func main() {

	var m App

	viper.SetConfigName("app")
	viper.AddConfigPath("./")
	viper.ReadInConfig()

	err := viper.Unmarshal(&m)
	fmt.Println(err)
	fmt.Print(m)
}

#app.yaml
  user: cx111
  can-do: make you happy
  want: your money
  latest: v.1.1.1
  more: hi

#out
<nil>
{cx111 make you happy your money v.1.1.1 hi}
```

Viper还支持解析到嵌入的结构体：

```
#main.go
type User struct {
	Name      string `mapstructure:"user"`
	AppConfig `mapstructure:"app"`
}
type AppConfig struct {
	CanDo  string `mapstructure:"can-do"`
	Want   string
	Latest string
}
func main() {

	var m User

	viper.SetConfigName("app")
	viper.AddConfigPath("./")
	viper.ReadInConfig()

	err := viper.Unmarshal(&m)
	fmt.Println(err)
	fmt.Print(m)
}

#app.yaml
user: cx111
app:
  can-do: make you happy
  want: your money
  latest: v.1.1.1

#out
<nil>
{cx111 {make you happy your money v.1.1.1}}
```

Viper在后台使用[github.com/mitchellh/mapstructure](https://github.com/mitchellh/mapstructure)来解析值，其默认情况下使用 `mapstructure`tag。

**注意** 当我们需要将viper读取的配置反序列到我们定义的结构体变量中时，一定要使用 `mapstructure`tag哦！
