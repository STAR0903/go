# 介绍

Swagger本质上是一种用于描述使用JSON表示的RESTful API的接口描述语言。Swagger与一组开源软件工具一起使用，以设计、构建、记录和使用RESTful Web服务。Swagger包括自动文档，代码生成和测试用例生成。

在前后端分离的项目开发过程中，如果后端同学能够提供一份清晰明了的接口文档，那么就能极大地提高大家的沟通效率和开发效率。可是编写接口文档历来都是令人头痛的，而且后续接口文档的维护也十分耗费精力。

最好是有一种方案能够既满足我们输出文档的需要又能随代码的变更自动更新，而Swagger正是那种能帮我们解决接口文档问题的工具。

这里以gin框架为例，使用[gin-swagger](https://github.com/swaggo/gin-swagger)库以使用Swagger 2.0自动生成RESTful API文档。

想要使用 `gin-swagger`为你的代码自动生成接口文档，一般需要下面三个步骤：

1. 按照swagger要求给接口代码添加声明式注释，具体参照[声明式注释格式](https://swaggo.github.io/swaggo.io/declarative_comments_format/)。
2. 使用swag工具扫描代码自动生成API接口文档数据
3. 使用gin-swagger渲染在线接口文档页面

# 添加注释

### main 函数注释

在程序入口main函数上以注释的方式写下项目相关介绍信息。

```
// @title 文档标题
// @version 1.0 文档版本
// @description 描述信息
// @termsOfService 链接到描述服务条款的页面的url

// @contact.name 联系人信息
// @contact.url 联系地址
// @contact.email 联系邮箱

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 接口服务host
// @BasePath base path
```

示例：

```
// @title Go Ldap Admin
// @version 1.0
// @description 基于Go+Vue实现的openLDAP后台管理项目
// @termsOfService https://github.com/eryajf/go-ldap-admin

// @contact.name 项目作者：二丫讲梵 、 swagger作者：南宫乘风
// @contact.url https://github.com/eryajf/go-ldap-admin
// @contact.email https://github.com/eryajf/go-ldap-admin

// @host 127.0.0.1:8888
// @BasePath /api
// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name Authorization
```

### 接口函数注释

在你代码中处理请求的接口函数（通常位于controller层）按如下方式写上注释：

```
// 接口函数名称
// @Summary 简介
// @Description 详细描述
// @Tags 标签
// @Accept 接受请求数据类型
// @Produce 返回响应数据类型
// @Param 参数格式，从左到右分别为：参数名、入参类型、数据类型、是否为必填字段、注释
// @Success 响应成功，从左到右分别为：状态码、参数类型、数据类型、注释
// @Failure 响应失败，从左到右分别为：状态码、参数类型、数据类型、注释
// @Router url [get/post/update/delete]
```

```
关于 @Param
参数名：参数的名字
参数类型： query、path、body、header，formData
query 表示带在url之后的参数
path 表示请求路径上得参数
body 表示是一个raw数据请求，当Accept是JSON格式时，我们使用该字段指定接收的JSON类型
header 表示带在header信息中得参数
formData 表示是post请求的数据
参数数据类型：
object (struct) 自定义struct
array  数组
string (string)
integer (int, uint, uint32, uint64)
number (float32)
boolean (bool)

```

示例：

```
// GetPostListHandler2 
// @Summary 升级版帖子列表接口
// @Description 可按社区按时间或分数排序查询帖子列表接口
// @Tags 帖子相关接口
// @Accept application/json
// @Produce application/json
// @Param Authorization header string false "Bearer 用户令牌"
// @Param object query models.ParamPostList false "查询参数"
// @Success 200 {object} _ResponsePostList
// @Router /posts2 [get]
func GetPostListHandler2(c *gin.Context) {
	// GET请求参数(query string)：/api/v1/posts2?page=1&size=10&order=time
	// 初始化结构体时指定初始参数
	p := &models.ParamPostList{
		Page:  1,
		Size:  10,
		Order: models.OrderTime,
	}

	if err := c.ShouldBindQuery(p); err != nil {
		zap.L().Error("GetPostListHandler2 with invalid params", zap.Error(err))
		ResponseError(c, CodeInvalidParam)
		return
	}
	data, err := logic.GetPostListNew(p)
	// 获取数据
	if err != nil {
		zap.L().Error("logic.GetPostList() failed", zap.Error(err))
		ResponseError(c, CodeServerBusy)
		return
	}
	ResponseSuccess(c, data)
	// 返回响应
}
```

# 生成数据

编写完注释后，使用以下命令安装swag工具：

`go get -u github.com/swaggo/swag/cmd/swag`

在项目根目录执行以下命令，使用swag工具生成接口文档数据。

`swag init`

执行完上述命令后，如果你写的注释格式没问题，此时你的项目根目录下会多出一个 `docs`文件夹。

```
./docs
├── docs.go
├── swagger.json
└── swagger.yaml
```

# 渲染文档数据

然后在项目代码中注册路由的地方按如下方式引入 `gin-swagger`相关内容：

```
import (
	_ "bluebell/docs"  // 导入把你上一步生成的docs

	gs "github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"

	"github.com/gin-gonic/gin"
)

```

注册swagger api相关路由

```
`r.GET("/swagger/*any", gs.WrapHandler(swaggerFiles.Handler))`
```

把你的项目程序运行起来，打开浏览器访问[http://localhost:8080/swagger/index.html](http://localhost:8080/swagger/index.html)就能看到Swagger 2.0 Api文档了。

`gin-swagger`同时还提供了 `DisablingWrapHandler`函数，方便我们通过设置某些环境变量来禁用Swagger。例如：

```
r.GET("/swagger/*any", gs.DisablingWrapHandler(swaggerFiles.Handler, "NAME_OF_ENV_VARIABLE"))
```

此时如果将环境变量 `NAME_OF_ENV_VARIABLE`设置为任意值，则 `/swagger/*any`将返回404响应，就像未指定路由时一样。

# 实时更新

更改 .air.conf 文档

![1729070694597](image/swagger/1729070694597.png)

![1729070704474](image/swagger/1729070704474.png)
