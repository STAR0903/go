# Gin-zap日志

#### gin默认的中间件

首先我们来看一个最简单的gin项目：

```
func main() {
	r := gin.Default()
	r.GET("/hello", func(c *gin.Context) {
		c.String("hello liwenzhou.com!")
	})
	r.Run(
}
```

接下来我们看一下 `gin.Default()`的源码：

```
func Default() *Engine {
	debugPrintWARNINGDefault()
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```

也就是我们在使用 `gin.Default()`的同时是用到了gin框架内的两个默认中间件 `Logger()`和 `Recovery()`

其中 `Logger()`是把gin框架本身的日志输出到标准输出（我们本地开发调试时在终端输出的那些日志就是它的功劳），而 `Recovery()`是在程序出现panic的时候恢复现场并写入500响应的

#### 基于zap的中间件

我们可以模仿 `Logger()`和 `Recovery()`的实现，使用我们的日志库来接收gin框架默认输出的日志。

这里以zap为例，我们实现两个中间件如下：

```
// GinLogger 接收gin框架默认的日志
func GinLogger(logger *zap.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery
		c.Next()

		cost := time.Since(start)
		logger.Info(path,
			zap.Int("status", c.Writer.Status()),
			zap.String("method", c.Request.Method),
			zap.String("path", path),
			zap.String("query", query),
			zap.String("ip", c.ClientIP()),
			zap.String("user-agent", c.Request.UserAgent()),
			zap.String("errors", c.Errors.ByType(gin.ErrorTypePrivate).String()),
			zap.Duration("cost", cost),
		)
	}
}

// GinRecovery recover掉项目可能出现的panic
func GinRecovery(logger *zap.Logger, stack bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				// Check for a broken connection, as it is not really a
				// condition that warrants a panic stack trace.
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") ||
							strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}

				httpRequest, _ := httputil.DumpRequest(c.Request, false)
				if brokenPipe {
					logger.Error(c.Request.URL.Path,
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
					// If the connection is dead, we can't write a status to it.
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
					return
				}

				if stack {
					logger.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
						zap.String("stack", string(debug.Stack())),
					)
				} else {
					logger.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
				}
				c.AbortWithStatus(http.StatusInternalServerError)
			}
		}()
		c.Next()
	}
}
```

如果不想自己实现，可以使用github上有别人封装好的[https://github.com/gin-contrib/zap](https://github.com/gin-contrib/zap)

这样我们就可以在gin框架中使用我们上面定义好的两个中间件来代替gin框架默认的 `Logger()`和 `Recovery()`

```
//gin默认中间件
r := gin.Default()
//zap中间件
r := gin.New()
r.Use(GinLogger(logger), GinRecovery(logger, true))
```
