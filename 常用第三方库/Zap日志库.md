# 配置

Zap提供了两种类型的日志记录器—`Sugared Logger`和 `Logger`。

在性能很好但不是很关键的上下文中，使用 `SugaredLogger`。它比其他结构化日志记录包快4-10倍，并且支持结构化和printf风格的日志记录。

在每一微秒和每一次内存分配都很重要的上下文中，使用 `Logger`。它甚至比 `SugaredLogger`更快，内存分配次数也更少，但它只支持强类型的结构化日志记录。

### Logger

通过调用 `zap.NewProduction()`/`zap.NewDevelopment()`或者 `zap.Example()`创建一个Logger。

上面的每一个函数都将创建一个logger。唯一的区别在于它将记录的信息不同。例如`production logger`默认记录调用函数信息、日期和时间等。

通过Logger调用Info/Error等。

默认情况下日志都会打印到应用程序的console界面。

```
var logger *zap.Logger

func main() {
	InitLogger()
  	defer logger.Sync()
	simpleHttpGet("www.google.com")
	simpleHttpGet("http://www.google.com")
}

func InitLogger() {
	logger, _ = zap.NewProduction()
}

func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		logger.Error(
			"Error fetching url..",
			zap.String("url", url),
			zap.Error(err))
	} else {
		logger.Info("Success..",
			zap.String("statusCode", resp.Status),
			zap.String("url", url))
		resp.Body.Close()
	}
}

```

在上面的代码中，我们首先创建了一个Logger，然后使用Info/ Error等Logger方法记录消息。

日志记录器方法的语法是这样的：

其中 `MethodXXX`是一个可变参数函数，可以是Info / Error/ Debug / Panic等。每个方法都接受一个消息字符串和任意数量的 `zapcore.Field`场参数。

每个 `zapcore.Field`其实就是一组键值对参数。

### Sugared Logger

现在让我们使用Sugared Logger来实现相同的功能。

我们通过调用主logger的 `. Sugar()`方法来获取一个 `SugaredLogger`。

与 Logger 不同，SugaredLogger 并不坚持结构化日志记录。对于每个日志级别，它公开了四种方法：

```
以 log.Print 样式日志记录的日志级别命名的方法
以“w”结尾的方法，用于松散类型的结构化日志记录
以“f”结尾的方法用于 log.Printf 风格的日志记录
以“ln”结尾的方法用于 log.Println 风格的日志记录
```

例如，InfoLevel 的方法有：

```
Info(...any)           Print-style logging
Infow(...any)          Structured logging (read as "info with")
Infof(string, ...any)  Printf-style logging
Infoln(...any)         Println-style logging
```

```
func (s *SugaredLogger) Info(args ...interface{})
Info 在InfoLevel记录提供的参数。当两个参数都不是字符串时，会在参数之间添加空格。

func (s *SugaredLogger) Infof(template string, args ...interface{})
Infof 根据格式说明符格式化消息并将其记录在InfoLevel中。

func (s *SugaredLogger) Infoln(args ...interface{})
Infoln 在InfoLevel记录一条消息。参数之间始终添加空格。v1.22.0中添加。

func (s *SugaredLogger) Infow(msg string, keysAndValues ...interface{})
Infow 记录一条带有一些附加上下文的消息。可变参数键值对的处理方式与 With 中的情况相同。
```

# 定制logger

### 日志写入文件

我们要做的第一个更改是把日志写入文件，而不是打印到应用程序控制台。

我们将使用 `zap.New(…)`方法来手动传递所有配置，而不是使用像 `zap.NewProduction()`这样的预置方法来创建logger。

`func New(core zapcore.Core, options ...Option) *Logger `

```
zapcore.Core需要三个配置——Encoder，WriteSyncer，LogLevel
1.Encoder:编码器(如何写入日志)
zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
zap.NewProductionEncoderConfig()：这个函数用于创建一个适用于生产环境的编码器配置
zapcore.NewJSONEncoder：使用前面创建的生产环境编码器配置来创建一个 JSON 格式的编码器。JSON 编码器会将日志数据编码为 JSON 格式，以便于存储和解析
2.WriterSyncer ：指定日志将写到哪里去
file, _ := os.Create("./test.log")
writeSyncer := zapcore.AddSync(file)
zapcore.AddSync(): 它将打开的文件句柄 file 转换为一个 WriteSyncer 对象。WriteSyncer 用于指定日志的输出目的地，在这个例子中就是刚刚创建的文件 test.log
3.Log Level：哪种级别的日志将被写入，只有级别不低于指定级别的日志才会被处理和输出
```

```
func InitLogger() {
	encoder := zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
	file, _ := os.Create("./test.log")
	ws := zapcore.AddSync(file)
	core := zapcore.NewCore(encoder, ws, zapcore.DebugLevel)
	logger = zap.New(core)
}
```

### Log Encoder

现在，我们希望将编码器从JSON Encoder更改为普通Encoder。为此，我们需要将 `NewJSONEncoder()`更改为 `NewConsoleEncoder()`

```
zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())
zapcore.NewConsoleEncoder 函数使用上述创建的生产环境配置来创建一个适用于控制台输出的编码器。
这样创建的编码器在将日志数据编码为可输出到控制台的文本格式时，会遵循生产环境的一些最佳实践和默认设置。
例如，如果生产环境的编码器配置中指定了时间格式为 RFC3339 ，那么使用这个编码器输出的日志中的时间字段就会按照 RFC3339 的格式进行显示。
又比如，如果配置中指定了某些字段的优先级和排序方式，编码器在编码日志时也会按照相应的规则进行处理。
总的来说，通过这种方式创建的控制台编码器，能够在保证符合生产环境要求的前提下，将日志以合适的格式输出到控制台，方便开发者进行查看和分析。
```

### 更改时间编码

鉴于我们对配置所做的更改，有下面两个问题：

* 时间是以非人类可读的方式展示，例如1.572161051846623e+09
* 调用方函数的详细信息没有显示在日志中

我们要做的第一件事是覆盖默认的 `ProductionConfig()`，并进行以下更改:

* 修改时间编码器
* 在日志文件中使用大写字母记录日志级别

```
func NewProductionEncoderConfig() zapcore.EncoderConfig {
	return zapcore.EncoderConfig{
		TimeKey:        "ts",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
		EncodeTime:     zapcore.EpochTimeEncoder,
		EncodeDuration: zapcore.SecondsDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
}
```

```
//非人类可读[EncodeTime:zapcore.EpochTimeEncoder]
encoder := zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())
//人类可读[EncodeTime:zapcore.ISO8601TimeEncoder]
cfg := zapcore.EncoderConfig{
		TimeKey:        "ts",
		LevelKey:       "level",
		NameKey:        "logger",
		CallerKey:      "caller",
		FunctionKey:    zapcore.OmitKey,
		MessageKey:     "msg",
		StacktraceKey:  "stacktrace",
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.LowercaseLevelEncoder,
		EncodeTime:     zapcore.ISO8601TimeEncoder,
		EncodeDuration: zapcore.SecondsDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
	}
encoder := zapcore.NewConsoleEncoder(cfg)
```

### 添加调用者详细信息

添加将调用函数信息记录到日志中的功能，在 `zap.New(..)`函数中添加一个 `Option`

`logger := zap.New(core, zap.AddCaller())   //哪里调用了logger`

### AddCallerSkip

`logger := zap.New(core, zap.AddCaller(), zap.AddCallerSkip(1))`

```
zap.AddCallerSkip(skip int) 用于在使用 Uber 的 Zap 日志库时，跳过指定数量的调用者层级
当在日志记录中添加调用者信息（通常是记录日志的代码所在的文件和行号）时，zap.AddCallerSkip(skip) 会使得日志中显示的调用者信息跳过 skip 个层级
当你的日志记录函数经过了一层额外的封装或中间函数时，使用 zap.AddCallerSkip(1) 可以让日志直接显示更接近实际业务逻辑的调用位置，而不是显示封装函数或中间函数的位置
举个例子，如果你有一个封装了日志记录的函数 logWrapper，在这个函数内部使用了 Zap 记录日志，并且使用了 zap.AddCallerSkip(1)，那么最终记录的日志中显示的调用者将是调用 logWrapper 的地方，而不是 logWrapper 函数本身的位置
这样可以更准确地反映出业务代码中实际产生日志的位置，方便在查看日志时快速定位到相关的业务逻辑代码。但需要注意的是，过度使用 zap.AddCallerSkip 可能会导致难以追踪日志的真正来源，所以应该谨慎使用，并确保对其效果有清晰的理解
```

### 日志输出

##### 日志输出到多个位置

我们可以将日志同时输出到文件和终端

```
func io.MultiWriter(writers ...io.Writer) io.Writer
可以传递任意数量的 io.Writer 参数给 MultiWriter 函数。
灵活性：可以自由选择将数据同时输出到多个目标，例如同时输出到文件、控制台、网络等。
可扩展性：能够随时添加或移除 io.Writer 对象，以适应不同的日志记录需求变化。
统一性：通过使用相同的日志记录方式，可以保持输出的一致性，方便对日志进行管理和分析。
```

```
//只输出到文档
file, _ := os.Create("./test.log")
ws := zapcore.AddSync(file)
core := zapcore.NewCore(encoder, ws, zapcore.DebugLevel)
//输出到文件和终端
file, _ := os.Create("./test.log")
ws := io.MultiWriter(file, os.Stdout)
core := zapcore.NewCore(encoder, zapcore.AddSync(ws), zapcore.DebugLevel)
```

##### err日志单独输出

有时候我们除了将全量日志输出到 `xx.log`文件中之外，还希望将 `ERROR`级别的日志单独输出到一个名为 `xx.err.log`的日志文件中。我们可以通过以下方式实现

```
func zapcore.NewTee(cores ...zapcore.Core) zapcore.Core
zapcore.NewTee 函数用于将多个 zapcore.Core 对象衔接在一起，创建一个可以同时向多个子核心写入日志的复合核心对象
它接受不定数量的 core 参数，然后返回一个新的 core 对象。这个新的核心对象会将接收到的日志操作同时分发到所有衔接的子核心上
func zapcore.NewCore(enc zapcore.Encoder, ws zapcore.WriteSyncer, enab zapcore.LevelEnabler) zapcore.Core
zapcore.Core需要三个配置——Encoder，WriteSyncer，LogLevel
LogLevel：哪种级别的日志将被写入，只有级别不低于指定级别的日志才会被处理和输出
```

```
func InitLogger() {
	encoder := zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())
	// test.log记录全量日志
	logF, _ := os.Create("./test.log")
	c1 := zapcore.NewCore(encoder, zapcore.AddSync(logF), zapcore.DebugLevel)
	// test.err.log记录ERROR级别的日志
	errF, _ := os.Create("./test.err.log")
	c2 := zapcore.NewCore(encoder, zapcore.AddSync(errF), zap.ErrorLevel)
	// 使用NewTee将c1和c2合并到core
	core := zapcore.NewTee(c1, c2)
	logger = zap.New(core, zap.AddCaller())
}
```

使用 `NewTee`将 `c1`和 `c2`合并到 `core`，还可以使用切片的方式

```
var coreArr []zapcore.Core
coreArr = append(coreArr, c1)
coreArr = append(coreArr, c2)
logger := zap.New(zapcore.NewTee(coreArr...), zap.AddCaller())
```

##### 自定义的日志输出

`LevelEnablerFunc` 是一个非常灵活且简便的方式，可以通过匿名函数根据日志级别来动态启用或禁用日志。它为在复杂的日志输出配置中根据级别控制日志的记录提供了非常方便的手段。

```
// 日志级别
highPriority := zap.LevelEnablerFunc(func(level zapcore.Level) bool {
	return level >= zap.ErrorLevel
})
lowPriority := zap.LevelEnablerFunc(func(level zapcore.Level) bool {
	return level < zap.ErrorLevel && level >= zap.DebugLevel
})
```

##### 日志输出原码分析

```
#zap/zapcore/core.go

// NewCore creates a Core that writes logs to a WriteSyncer.
func NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core {
	return &ioCore{
		LevelEnabler: enab,
		enc:          enc,
		out:          ws,
	}
}type ioCore struct {
	LevelEnabler
	enc Encoder
	out WriteSyncer
}
```

```
#zap/zapcore/level.go

// Enabled returns true if the given level is at or above this level.
func (l Level) Enabled(lvl Level) bool {
	return lvl >= l
}

// LevelEnabler decides whether a given logging level is enabled when logging a
// message.
//
// Enablers are intended to be used to implement deterministic filters;
// concerns like sampling are better implemented as a Core.
//
// Each concrete Level value implements a static LevelEnabler which returns
// true for itself and all higher logging levels. For example WarnLevel.Enabled()
// will return true for WarnLevel, ErrorLevel, DPanicLevel, PanicLevel, and
// FatalLevel, but return false for InfoLevel and DebugLevel.
type LevelEnabler interface {
	Enabled(Level) bool
}
```

```
c1 := zapcore.NewCore(encoder, zapcore.AddSync(logF), zapcore.DebugLevel)
// Level 实现了 LevelEnabler 接口 ，通过比较日志级别与设定级别的大小决定是否输出日志
```

```
#zap/level.go

// LevelEnablerFunc is a convenient way to implement zapcore.LevelEnabler with
// an anonymous function.
//
// It's particularly useful when splitting log output between different
// outputs (e.g., standard error and standard out). For sample code, see the
// package-level AdvancedConfiguration example.
type LevelEnablerFunc func(zapcore.Level) bool

// Enabled calls the wrapped function.
func (f LevelEnablerFunc) Enabled(lvl zapcore.Level) bool { return f(lvl) }

```

```
highPriority := zap.LevelEnablerFunc(func(level zapcore.Level) bool {
	return level >= zap.ErrorLevel
})
zapcore.NewCore(encoder, zapcore.NewMultiWriteSyncer(errorFileWriteSyncer, zapcore.AddSync(os.Stdout)), highPriority)
// LevelEnablerFunc 实现了 LevelEnabler 接口 ，通过自定义规则决定是否输出日志
```

# 日志切割归档

这个日志程序中唯一缺少的就是日志切割归档功能。

Zap本身不支持切割归档日志文件我们,将使用第三方库[Lumberjack](https://github.com/natefinch/lumberjack)来实现。

目前只支持按文件大小切割，原因是按时间切割效率低且不能保证日志数据不被破坏详情戳[chlick](https://github.com/natefinch/lumberjack/issues/54)

想按日期切割可以使用[这个](github.com/lestrrat-go/file-rotatelogs)库，虽然目前不维护了，但也够用了

```
// 使用file-rotatelogs按天切割日志

import rotatelogs "github.com/lestrrat-go/file-rotatelogs"

l, _ := rotatelogs.New(
	filename+".%Y%m%d%H%M",
	rotatelogs.WithMaxAge(30*24*time.Hour),    // 最长保存30天
	rotatelogs.WithRotationTime(time.Hour*24), // 24小时切割一次
)
zapcore.AddSync(l)
```

### 安装

`go get gopkg.in/natefinch/lumberjack.v2`

### 运用

```
Lumberjack Logger采用以下属性作为输入:
Filename: 日志文件的位置
MaxSize：在进行切割之前，日志文件的最大大小（以MB为单位）
MaxBackups：保留旧文件的最大个数
MaxAges：保留旧文件的最大天数
Compress：是否压缩/归档旧文件
```

```
//未加入
file, _ := os.Create("./test.log")
ws := zapcore.AddSync(file)
//已加入
lumberJackLogger := &lumberjack.Logger{
		Filename:   "./test.log",
		MaxSize:    1,
		MaxBackups: 5,
		MaxAge:     30,
		Compress:   false,
}
ws := zapcore.AddSync(lumberJackLogger)
```
