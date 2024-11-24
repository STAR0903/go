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

# RBAC 模型

`ACL`模型在用户和资源都比较少的情况下没什么问题，但是用户和资源量一大，`ACL`就会变得异常繁琐。想象一下，每次新增一个用户，都要把他需要的权限重新设置一遍是多么地痛苦。`RBAC`（role-based-access-control）模型通过引入角色（`role`）这个中间层来解决这个问题。每个用户都属于一个角色，例如开发者、管理员、运维等，每个角色都有其特定的权限，权限的增加和删除都通过角色来进行。这样新增一个用户时，我们只需要给他指派一个角色，他就能拥有该角色的所有权限。修改角色的权限时，属于这个角色的用户权限就会相应的修改。

### 快速上手

在 `casbin`中使用 `RBAC`模型需要在模型文件中添加 `role_definition`模块：

```model.conf
[role_definition]
g = _, _

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

`g = _,_`定义了用户——角色，角色——角色的映射关系，前者是后者的成员，拥有后者的权限。然后在匹配器中，我们不需要判断 `r.sub`与 `p.sub`完全相等，只需要使用 `g(r.sub, p.sub)`来判断请求主体 `r.sub`是否属于 `p.sub`这个角色即可。最后我们修改策略文件添加用户——角色定义：

```policy.csv
p, admin, data, read, allow
p, admin, data, write, allow

g, alice, admin


p, user, data, read, allow
p, user, data, write, deny

g, bob, user
```

上面的 `policy.csv`文件规定了，`alice`属于 `admin`管理员，`bob`属于 `user`开发者，使用 `g`来定义这层关系。另外 `admin`对数据 `data`用 `read`和 `write`权限，而 `user`对数据 `data`只有 `read`权限。

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
	check(e, "alice", "data", "read")
	check(e, "alice", "data", "write")
	check(e, "bob", "data", "read")
	check(e, "bob", "data", "write")

}

```

```
alice data read : true
alice data write : true
bob data read : true
bob data write : false
```

### 多个 `RBAC`

`casbin`支持同时存在多个 `RBAC`系统，即用户和资源都有角色：

```model.conf
[role_definition]
g=_,_
g2=_,_

[matchers]
m = g(r.sub, p.sub) && g2(r.obj, p.obj) && r.act == p.act
```

上面的模型文件定义了两个 `RBAC`系统 `g`和 `g2`，我们在匹配器中使用 `g(r.sub, p.sub)`判断请求主体属于特定组，`g2(r.obj, p.obj)`判断请求资源属于特定组，且操作一致即可放行。

策略文件:

```policy.csv
p, admin, prod, read
p, admin, prod, write
p, admin, dev, read
p, admin, dev, write
p, developer, dev, read
p, developer, dev, write
p, developer, prod, read
g, dajun, admin
g, lizi, developer
g2, prod.data, prod
g2, dev.data, dev
```

先看角色关系，即最后 4 行，`dajun`属于 `admin`角色，`lizi`属于 `developer`角色，`prod.data`属于生产资源 `prod`角色，`dev.data`属于开发资源 `dev`角色。`admin`角色拥有对 `prod`和 `dev`类资源的读写权限，`developer`只能拥有对 `dev`的读写权限和 `prod`的读权限。

```golang
check(e, "dajun", "prod.data", "read")
check(e, "dajun", "prod.data", "write")
check(e, "lizi", "dev.data", "read")
check(e, "lizi", "dev.data", "write")
check(e, "lizi", "prod.data", "write")
```

第一个函数中 `e.Enforce()`方法在实际执行的时候先获取 `dajun`所属角色 `admin`，再获取 `prod.data`所属角色 `prod`，根据文件中第一行 `p, admin, prod, read`允许请求。最后一个函数中 `lizi`属于角色 `developer`，而 `prod.data`属于角色 `prod`，所有策略都不允许，故该请求被拒绝：

```cmd
dajun CAN read prod.data
dajun CAN write prod.data
lizi CAN read dev.data
lizi CAN write dev.data
lizi CANNOT write prod.data
```

### 多层角色

`casbin`还能为角色定义所属角色，从而实现多层角色关系，这种权限关系是可以传递的。例如 `dajun`属于高级开发者 `senior`，`seinor`属于开发者，那么 `dajun`也属于开发者，拥有开发者的所有权限。我们可以定义开发者共有的权限，然后额外为 `senior`定义一些特殊的权限。

模型文件不用修改，策略文件改动如下：

```policy.csv
p, senior, data, write
p, developer, data, read
g, dajun, senior
g, senior, developer
g, lizi, developer
```

上面 `policy.csv`文件定义了高级开发者 `senior`对数据 `data`有 `write`权限，普通开发者 `developer`对数据只有 `read`权限。同时 `senior`也是 `developer`，所以 `senior`也继承其 `read`权限。`dajun`属于 `senior`，所以 `dajun`对 `data`有 `read`和 `write`权限，而 `lizi`只属于 `developer`，对数据 `data`只有 `read`权限。

### `RBAC` domain

在 `casbin`中，角色可以是全局的，也可以是特定 `domain`（领域）或 `tenant`（租户），可以简单理解为 **组** 。例如 `dajun`在组 `tenant1`中是管理员，拥有比较高的权限，在 `tenant2`可能只是个弟弟。

使用 `RBAC domain`需要对模型文件做以下修改：

```model.conf
[request_definition]
r = sub, dom, obj, act

[policy_definition]
p = sub, dom, obj, act

[role_definition]
g = _,_,_

[matchers]
m = g(r.sub, p.sub, r.dom) && r.dom == p.dom && r.obj == p.obj && r.act == p.obj
```

`g=_,_,_`表示前者在后者中拥有中间定义的角色，在匹配器中使用 `g`要带上 `dom`。

```
p, admin, tenant1, data1, read
p, admin, tenant2, data2, read
g, dajun, admin, tenant1
g, dajun, developer, tenant2
```

在 `tenant1`中，只有 `admin`可以读取数据 `data1`。在 `tenant2`中，只有 `admin`可以读取数据 `data2`。`dajun`在 `tenant1`中是 `admin`，但是在 `tenant2`中不是。

# ABAC

`RBAC`模型对于实现比较规则的、相对静态的权限管理非常有用。但是对于特殊的、动态的需求，`RBAC`就显得有点力不从心了。例如，我们在不同的时间段对数据 `data`实现不同的权限控制。正常工作时间 `9:00-18:00`所有人都可以读写 `data`，其他时间只有数据所有者能读写。这种需求我们可以很方便地使用 `ABAC`（attribute base access list）模型完成：

```model.conf
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[matchers]
m = r.sub.Hour >= 9 && r.sub.Hour < 18 || r.sub.Name == r.obj.Owner

[policy_effect]
e = some(where (p.eft == allow))
```

该规则不需要策略文件：

```golang
type Object struct {
  Name  string
  Owner string
}

type Subject struct {
  Name string
  Hour int
}

func check(e *casbin.Enforcer, subSubject, objObject, actstring) {
  ok, _ := e.Enforce(sub, obj, act)
  ifok{
    fmt.Printf("%s CAN %s %s at %d:00\n", sub.Name, act, obj.Name, sub.Hour)
  } else{
    fmt.Printf("%s CANNOT %s %s at %d:00\n", sub.Name, act, obj.Name, sub.Hour)
  }
}

func main() {
  e, err := casbin.NewEnforcer("./model.conf")
  if err != nil {
    log.Fatalf("NewEnforecer failed:%v\n", err)
  }

  o := Object{"data", "dajun"}
  s1 := Subject{"dajun", 10}
  check(e, s1, o, "read")

  s2 := Subject{"lizi", 10}
  check(e, s2, o, "read")

  s3 := Subject{"dajun", 20}
  check(e, s3, o, "read")

  s4 := Subject{"lizi", 20}
  check(e, s4, o, "read")
}
```

显然 `lizi`在 `20:00`不能 `read`数据 `data`：

```cmd
dajun CAN read data at 10:00
lizi CAN read data at 10:00
dajun CAN read data at 20:00
lizi CANNOT read data at 20:00
```

我们知道，在 `model.conf`文件中可以通过 `r.sub`和 `r.obj`，`r.act`来访问传给 `Enforce`方法的参数。

使用 `ABAC`模型可以非常灵活的权限控制，但是一般情况下 `RBAC`就已经够用了。

# 优先级模型

Casbin支持参考优先级加载策略。

### 隐式优先级

这非常简单，顺序决定了策略的优先级，策略出现的越早优先级就越高。

model.conf：

```ini
[policy_effect]
e = priority(p.eft) || deny
Copy
```

### 显式优先级

策略定义中的优先级令牌名称必须是“优先级”，**较小的优先级值将具有较高的优先级**。 如果优先级有非数字字符，它将是被排在最后，而不是导致报错。 现在，明确的优先级仅支持 `添加策略` & `添加策略`，如果 `升级策略` 被调用，那么您不应该改变优先级属性。

model.conf：

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = priority, sub, obj, act, eft

[role_definition]
g = _, _

[policy_effect]
e = priority(p.eft) || deny

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
Copy
```

policy.csv

```csv
p, 10, data1_deny_group, data1, read, deny
p, 10, data1_deny_group, data1, write, deny
p, 10, data2_allow_group, data2, read, allow
p, 10, data2_allow_group, data2, write, allow


p, 1, alice, data1, write, allow
p, 1, alice, data1, read, allow
p, 1, bob, data2, read, deny

g, bob, data2_allow_group
g, alice, data1_deny_group
```

### 基于角色和用户层次结构优先级

角色和用户的继承结构只能是多棵树，而不是图。 如果一个用户有多个角色，您必须确保用户在不同树上有相同的等级。 **如果两种角色具有相同的等级，那么出现早的策略（相应的角色）就显得更加优先**。 更多详情请看 [casbin#833](https://github.com/casbin/casbin/pull/833)、[casbin#831](https://github.com/casbin/casbin/issues/831)

model.conf：

```ini
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act, eft

[role_definition]
g = _, _

[policy_effect]
e = subjectPriority(p.eft) || deny

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act 
```

policy.csv

```csv
p, root, data1, read, deny
p, admin, data1, read, deny

p, editor, data1, read, deny
p, subscriber, data1, read, deny

p, jane, data1, read, allow
p, alice, data1, read, allow

g, admin, root

g, editor, admin
g, subscriber, admin

g, jane, editor
g, alice, subscriber 
Copy
```

请求：

```
jane, data1, read --> true //jane在最底部,所以优先级高于editor, admin 和 root
alice, data1, read --> true
```

像这样的角色层次结构：

```
role: root
 └─ role: admin
    ├─ role editor
    │  └─ user: jane
    │
    └─ role: subscriber
       └─ user: john
```

优先级类似于：

```
role: root # 自动优先级: 30
 --role: admin# 自动优先级: 20
     --role: editor # 自动优先级: 10
     --role: subscriber # 自动优先级: 10
```

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

# 存储

### Model存储

与 policy 不同，model 只能加载，不能保存。 因为我们认为 model 不是动态组件，不应该在运行时进行修改，所以我们没有实现一个 API 来将 model 保存到存储中。

但是，好消息是，我们提供了三种等效的方法来静态或动态地加载模型：

##### .CONF 文件加载

```
#model.conf

[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
```

##### 代码加载

模型可以从代码中动态初始化，不需要使用 `.CONF`。下面是RBAC模型的一个例子：

```
package main

import (
	"fmt"
	"github.com/casbin/casbin/v2"
	"github.com/casbin/casbin/v2/model"
	fileadapter "github.com/casbin/casbin/v2/persist/file-adapter"
)

func check(e *casbin.Enforcer, sub, obj, act string) {
	ok, err := e.Enforce(sub, obj, act)
	if err != nil {
		fmt.Println("Check permission failed:", err)
	}
	fmt.Println(sub, obj, act, ":", ok) // Output: Is alice allowed to read data1? true
}
func main() {
	// 从Go代码初始化模型
	m := model.NewModel()
	m.AddDef("r", "r", "sub, obj, act")
	m.AddDef("p", "p", "sub, obj, act, eft")
	m.AddDef("g", "g", "_, _")
	m.AddDef("e", "e", "some(where (p.eft == allow))")
	m.AddDef("m", "m", "g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act")

	// 从CSV文件adapter加载策略规则
	// 使用自己的 adapter 替换,不能使用 casbin.NewEnforcer(m, "./policy.csv")
	a := fileadapter.NewAdapter("./policy.csv")

	// 创建enforcer
	e, _ := casbin.NewEnforcer(m, a)
	check(e, "alice", "data1", "read")
	check(e, "alice", "data1", "write")
	check(e, "bob", "data2", "read")
	check(e, "bob", "data2", "write")
}

```

##### 字符串加载

或者您可以从多行字符串加载整个模型文本。这种方法的优点是您不需要维护模型文件。

```go
import (
    "github.com/casbin/casbin/v2"
    "github.com/casbin/casbin/v2/model"
)

// 从字符串初始化模型
text :=
`
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj && r.act == p.act
`
m, _ := model.NewModelFromString(text)

// 从CSV文件adapter加载策略规则
// 使用自己的 adapter 替换
a := fileadapter.NewAdapter("./policy.csv")

// 创建执行者。
e := casbin.NewEnforcer(m, a)
```

### Policy存储

在Casbin中，策略存储作为[ 适配器 ](https://v1.casbin.org/docs/zh-CN/adapters)来实现。

### CSV 文件载入

`CSV` 文件示例（不常用）

```
p, alice, data1, read
p, bob, data2, write
p, data2_admin, data2, read
p, data2_admin, data2, write
g, alice, data2_admin
Copy
```

如果你的文件包含逗号 `,` , 你应该用双引号把它包裹, 例如:

```
p, alice, "data1,data2", read --correcy
p, alice, data1,data2, read --insur ("data1,data2" 应该是一个整体)
```

如果您的文件包含逗号 `,` 和双引号 `"`, 你应该用双引号将字段放在一起, 并将任何嵌入的双引号加倍。

```
p, alice, data, "r.act in (""get"", ""post"")" --correct
p, alice, data, "r.act in ("get", "post")" --insur --unction (should use "" to fescape "")
```

### [适配器](https://v1.casbin.org/docs/zh-CN/adapters)API

| 方法                   | 类型   | 描述                           |
| ---------------------- | ------ | ------------------------------ |
| LoadPolicy()           | 基本的 | 从存储中加载所有策略规则       |
| SavePolicy()           | 基本的 | 保存所有策略规则到存储         |
| AddPolicy()            | 可选的 | 添加策略规则到存储             |
| RemovePolicy()         | 可选的 | 从存储中删除策略规则           |
| RemoveFilteredPolicy() | 可选的 | 从存储中删除匹配过滤规则的策略 |

### 数据库存储格式

**您的策略文件**

```
p, data2_admin, data2, read
p, data2_admin, data2, write
g, alice, admin, admin
```

**相应的数据库结构(比如 MySQL)**

| id | ptype | v0          | v1     | v2   | v3 | v4 | v5 |
| -- | ----- | ----------- | ------ | ---- | -- | -- | -- |
| 1  | p     | data2_admin | 数据2  | 可读 |    |    |    |
| 2  | p     | data2_admin | 数据2  | 可写 |    |    |    |
| 3  | g     | Alice       | 管理员 |      |    |    |    |

**每一列的含义**

* `id`: 只存在于数据库中作为主键。 不作为 `Casbin策略的一部分`。它生成的方式取决于特定的适配器
* `ptype`: 它对应 `p`, `g`, `g2`, 等等。
* `v0-v5`: 列名称没有特定的含义, 并对应 `策略csv` 中的值。 列数取决于您自己定义的数量。 理论上，可以有无限的列数。 但通常在适配器中只有 **6** 列。 如果您觉得还不够，请向相应的适配器仓库提交问题。

**支持多种包/第三方库操作数据库存储，具体点击[适配器](https://v1.casbin.org/docs/zh-CN/adapters)**

##### [Gorm Adapter](https://github.com/casbin/gorm-adapter?tab=readme-ov-file)

```
package main

import (
	"github.com/casbin/casbin/v2"
	gormadapter "github.com/casbin/gorm-adapter/v3"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// 初始化一个 Gorm 适配器，并将其用于 Casbin 的执行器：
	// 适配器将使用名为 "casbin" 的 MySQL 数据库。
	// 如果数据库不存在，适配器将自动创建它。
	// 你也可以使用已经存在的 gorm 实例，通过 gormadapter.NewAdapterByDB(gormInstance)
	a, _ := gormadapter.NewAdapter("mysql", "mysql_username:mysql_password@tcp(127.0.0.1:3306)/") // 你的数据库驱动和数据源。
	e, _ := casbin.NewEnforcer("examples/rbac_model.conf", a)

	// 或者你可以使用一个已存在的数据库 "abc"，像这样：
	// 适配器将使用名为 "casbin_rule" 的表。
	// 如果表不存在，适配器将自动创建它。
	// a := gormadapter.NewAdapter("mysql", "mysql_username:mysql_password@tcp(127.0.0.1:3306)/abc", true)

	// 从数据库加载策略。
	e.LoadPolicy()

	// 检查权限。
	e.Enforce("alice", "data1", "read")

	// 修改策略。
	// e.AddPolicy(...)
	// e.RemovePolicy(...)

	// 将策略保存回数据库。
	e.SavePolicy()
}

```

# API

如果想要了解更多方法或者函数，继续学习[API](https://v1.casbin.org/docs/zh-CN/api-overview)
