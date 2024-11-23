# 概述

Casbin是一个强大的、高效的开源访问控制框架，其权限管理机制支持多种访问控制模型。

**如果这个笔记不能帮助解决问题，继续学习[Casbin](https://v1.casbin.org/docs/zh-CN/syntax-for-models)**

权限实际上就是控制**谁**能对**什么资源**进行什么操作。`casbin`将访问控制模型抽象到一个基于 PERM（Policy，Effect，Request，Matchers） 元模型的配置文件（模型文件）中。因此切换或更新授权机制只需要简单地修改配置文件。

`policy`是策略或者说是规则的定义。它定义了具体的规则。

`request`是对访问请求的抽象，它与 `e.Enforce()`函数的参数是一一对应的

`matcher`匹配器会将请求与定义的每个 `policy`一一匹配，生成多个匹配结果。

`effect`根据对请求运用匹配器得出的所有结果进行汇总，来决定该请求是**允许**还是 **拒绝** 。

Casbin 可以：

* 支持自定义请求的格式，默认的请求格式为{subject, object, action}。
* 具有访问控制模型model和策略policy两个核心概念。
* 支持RBAC中的多层角色继承，不止主体可以有角色，资源也可以具有角色。
* 支持内置的超级用户 例如：root或administrator。超级用户可以执行任何操作而无需显式的权限声明。
* 支持多种内置的操作符，如 keyMatch，方便对路径式的资源进行管理，如 /foo/bar 可以映射到 /foo*

Casbin 不能：

* 身份认证 authentication（即验证用户的用户名、密码），casbin只负责访问控制。应该有其他专门的组件负责身份认证，然后由casbin进行访问控制，二者是相互配合的关系。
* 管理用户列表或角色列表。 Casbin 认为由项目自身来管理用户、角色列表更为合适， 用户通常有他们的密码，但是 Casbin 的设计思想并不是把它作为一个存储密码的容器。 而是存储RBAC方案中用户和角色之间的映射关系。

# 快速上手（ACL）

### 模型文件

```
#model.conf

[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act

[policy_effect]
e = some(where (p.eft == allow))

```

上面模型文件规定了权限由 `sub,obj,act`三要素组成，只有在策略列表中有和它完全相同的策略时，该请求才能通过。匹配器的结果可以通过 `p.eft`获取，`some(where (p.eft == allow))`表示只要有一条策略允许即可。

### 策略文件

```
#policy.csv

p,star,data1,read
p,for001,data1,write,read
p,qq,data2,read

```

策略文件，即谁能对什么资源进行什么操作。

### 实现代码

```
package main

import (
	"fmt"
	casbin "github.com/casbin/casbin/v2"
)

func check(e *casbin.Enforcer, sub, obj, act string) {
	ok, err := e.Enforce(sub, obj, act)
	if err != nil {
		fmt.Println("Check permission failed:", err)
	}
	fmt.Println(sub, obj, act, ":", ok) // Output: Is alice allowed to read data1? true
}

func main() {
	// Initialize a new Enforcer
	e, err := casbin.NewEnforcer("model.conf", "policy.csv")
	if err != nil {
		fmt.Println("Initialize a new Enforcer failed:", err)
		return
	}
	// Check the permission
	check(e, "alice", "data1", "read")
	check(e, "alice", "data1", "write")
	check(e, "bob", "data2", "read")
	check(e, "bob", "data2", "write")

}

```

上面例子中实现的就是 `ACL`（access-control-list，访问控制列表）。`ACL`显示定义了每个主体对每个资源的权限情况，未定义的就没有权限。

### 超级管理员

超级管理员可以进行任何操作。假设超级管理员为 `root`，我们只需要修改匹配器：

```
[matchers]
e = r.sub == p.sub && r.obj == p.obj && r.act == p.act || r.sub == "root"
```

只要访问主体是 `root`一律放行。

# Model语法

### Request定义

`[request_definition]` 部分用于request的定义，它明确了 `e.Enforce(...)` 函数中参数的含义。

```ini
[request_definition]
r = sub, obj, act
Copy
```

`sub, obj, act` 表示经典三元组: **访问实体 (Subject)，访问资源 (Object) 和访问方法 (Action)**。 但是, 你可以自定义你自己的请求表单, 如果不需要指定特定资源，则可以这样定义 `sub、act` ，或者如果有两个访问实体, 则为 `sub、sub2、obj、act`。

### Policy定义

`[policy_definition]` 部分是对policy的定义，以下文的 model 配置为例:

```ini
[policy_definition]
p = sub, obj, act
p2 = sub, act
```

这些是我们对policy规则的具体描述

```
p, alice, data1, read
p2, bob, write-all-objects
```

policy部分的每一行称之为一个策略规则， 每条策略规则通常以形如 `p`, `p2`的 `policy type`开头。 如果存在多个policy定义，那么我们会根据前文提到的 `policy type`与具体的某条定义匹配。 上面的policy的绑定关系将会在matcher中使用， 罗列如下：

```
(alice, data1, read) -> (p.sub, p.obj, p.act)
(bob, write-all-objects) -> (p2.sub, p2.act)
```

### Policy effect

[policy_effect] 部分是对policy生效范围的定义， 原语定义了当多个policy rule同时匹配访问请求request时,该如何对多个决策结果进行集成以实现统一决策。**目前为止你必须使用内置的 policy effects，不能自定义。**

##### 内在政策

支持的内在政策效应是：

| Policy effect                                                | 意义             | 示例                                                                    |
| ------------------------------------------------------------ | ---------------- | ----------------------------------------------------------------------- |
| some(where (p.eft == allow))                                 | allow-override   | [ACL, RBAC, etc.](https://v1.casbin.org/docs/en/supported-models#examples) |
| !some(where (p.eft == deny))                                 | deny-override    | [Deny-override](https://v1.casbin.org/docs/en/supported-models#examples)   |
| some(where (p.eft == allow)) && !some(where (p.eft == deny)) | allow-and-deny   | [Allow-and-deny](https://v1.casbin.org/docs/en/supported-models#examples)  |
| priority(p.eft)                                              | priority         | [Priority](https://v1.casbin.org/docs/en/supported-models#examples)        |
| subjectPriority(p.eft)                                       | 基于角色的优先级 | [主题优先级](https://v1.casbin.org/docs/en/supported-models#examples)      |

##### 示例分析

```
[policy_effect]
e = some(where (p.eft == allow))
```

该Effect原语表示如果**存在任意一个决策结果为allow的匹配规则，则最终决策结果为allow**，即allow-override。 其中p.eft 表示策略规则的决策结果，可以为allow 或者deny，当不指定规则的决策结果时,取默认值allow 。 通常情况下，policy的p.eft默认为allow。

```
[policy_effect]
e = !some(where (p.eft == deny))
```

该Effect原语表示**不存在任何决策结果为deny的匹配规则，则最终决策结果为allow** ，即deny-override。 some 量词判断是否存在一条策略规则满足匹配器。 any 量词则判断是否所有的策略规则都满足匹配器 (此处未使用)。 policy effect还可以利用逻辑运算符进行连接：

```
[policy_effect]
e = some(where (p.eft == allow)) && !some(where (p.eft == deny))
```

该Effect原语表示当**至少存在一个决策结果为allow的匹配规则，且不存在决策结果为deny的匹配规则时，则最终决策结果为allow**。 这时allow授权和deny授权同时存在，但是deny优先。

```
[policy_effect]
e = priority(p.eft) || deny
```

该Effect原语表示当**一个决策结果最先出现在策略规则，这个决策结果就是最终决策结果。**

实例

```
#model.conf

[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act, eft

[role_definition]
g = _, _

[policy_effect]
e = priority(p.eft) || deny

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act

```

```
#policy.csv

p, alice, data1, read, allow
p, data1_deny_group, data1, read, deny
p, data1_deny_group, data1, write, deny
p, alice, data1, write, allow

g, alice, data1_deny_group

p, data2_allow_group, data2, read, allow
p, bob, data2, read, deny
p, data2_allow_group, data2, write, allow
p, bob, data2, write, deny

g, bob, data2_allow_group

```

```
#main.go

package main

import (
	"fmt"
	casbin "github.com/casbin/casbin/v2"
)

func check(e *casbin.Enforcer, sub, obj, act string) {
	ok, err := e.Enforce(sub, obj, act)
	if err != nil {
		fmt.Println("Check permission failed:", err)
	}
	fmt.Println(sub, obj, act, ":", ok) // Output: Is alice allowed to read data1? true
}

func main() {
	// Initialize a new Enforcer
	e, err := casbin.NewEnforcer("model.conf", "policy.csv")
	if err != nil {
		fmt.Println("Initialize a new Enforcer failed:", err)
		return
	}
	// Check the permission
	check(e, "alice", "data1", "read")
	check(e, "alice", "data1", "write")
	check(e, "bob", "data2", "read")
	check(e, "bob", "data2", "write")

}

```

```
#运行结果

alice data1 read : true
alice data1 write : false
bob data2 read : true
bob data2 write : true

```

### 匹配器

`[matchers]` 是策略匹配程序的定义。匹配程序是表达式。它定义了如何根据请求评估策略规则。

```ini
[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
Copy
```

上述匹配器是最简单的，这意味着请求中的主题、对象和行动应该与政策规则中的匹配。

您可以在匹配器中使用诸如 `+, -, *, /` 和逻辑操作员，例如 `&&, ||, !`

### 多个班级类型

如果您需要多个策略定义或多个匹配器，您可以使用 `p2`, `m2`。 事实上，以上四个部分都可以使用多个类型，语法是 `r`+number 。 例如 `r2`, `e2`。 默认情况下，这四个部分应对应一个。 如您的 `r2` 只能使用匹配器 `m2` 匹配策略 `p2`。

您可以通过 `EnforceContext` 作为 `的第一个参数来执行` 方法来指定类型， `EnforceContext` 就像这样的

```go
EnforceContext{"r2","p2","e2","m2"}
type EnforceContext struct {
    RType string
    PType string
    EType string
    MType string
}
Copy
```

示例用法，请求如下所示：

```
#model.conf

[request_definition]
r = sub, obj, act
r2 = sub, obj, act

[policy_definition]
p = sub, obj, act
p2= sub_rule, obj, act, eft

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
#RABC
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
#ABAC
m2 = eval(p2.sub_rule) && r2.obj == p2.obj && r2.act == p2.act

```

```
#policy.csv

p, data2_admin, data2, read
p2, r2.sub.Age > 18 && r2.sub.Age < 60, /data1, read, allow
p2, r2.sub.Age > 60 && r2.sub.Age < 100, /data1, read, deny

g, alice, data2_admin

```

```go
#mian.go

package main

import (
	"fmt"
	casbin "github.com/casbin/casbin/v2"
)

func main() {
	// 初始化一个新的 Enforcer
	e, err := casbin.NewEnforcer("model.conf", "policy.csv")
	if err != nil {
		fmt.Println("初始化 Enforcer 失败:", err)
		return
	}
	// 在后缀中传递参数，例如 "2" 或 "3"，它会创建 r2、p2 等定义。
	enforceContext := casbin.NewEnforceContext("2")
	// 你也可以单独指定某个类型，例如这里指定的是策略效果类型 "e"。
	enforceContext.EType = "e"
	// 如果不传入 EnforceContext，则默认使用 r、p、e、m 定义。
	fmt.Println(e.Enforce("alice", "data2", "read"))
	// 传入 EnforceContext
	fmt.Println(e.Enforce(enforceContext, struct{ Age int }{Age: 70}, "/data1", "read")) 
	fmt.Println(e.Enforce(enforceContext, struct{ Age int }{Age: 30}, "/data1", "read")) 
}

#运行结果
true <nil>
false <nil>
true <nil>

```
