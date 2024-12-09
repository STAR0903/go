# 安装

```
go get -u gorm.io/gorm
go get -u gorm.io/driver/sqlite
```

[点击学习更多](https://gorm.io/zh_CN/)

# 约定

### 使用 `ID` 作为主键

默认情况下，GORM 会使用 `ID` 作为表的主键。

```
type User struct {
  ID   string // 默认情况下，名为 `ID` 的字段会作为表的主键
  Name string
}
```

你可以通过标签 `primaryKey` 将其它字段设为主键。

```
// 将 `UUID` 设为主键
type Animal struct {
  ID     int64
  UUID   string `gorm:"primaryKey"`
  Name   string
  Age    int64
}
```

通过将多个字段设为主键，以创建复合主键，例如：

```
type Product struct {
  ID           string `gorm:"primaryKey"`
  LanguageCode string `gorm:"primaryKey"`
  Code         string
  Name         string
}
```

 **注意：** 默认情况下，整型 `PrioritizedPrimaryField` 启用了 `AutoIncrement`，要禁用它，您需要为整型字段关闭 `autoIncrement`：

```
type Product struct {
  CategoryID uint64 `gorm:"primaryKey;autoIncrement:false"`
  TypeID     uint64 `gorm:"primaryKey;autoIncrement:false"`
}
```

### 表名

GORM 使用结构体名的 `蛇形命名` 作为表名。对于结构体 `User`，根据约定，其表名为 `users`

您可以实现 `Tabler` 接口来更改默认表名，例如：

```
type Tabler interface {
    TableName() string
}
// TableName 会将 User 的表名重写为 `profiles`
func (User) TableName() string {
  return "profiles"
}
```

**注意：**`TableName` 不支持动态变化，它会被缓存下来以便后续使用。想要使用动态表名，你可以使用 `Scopes`

### 列名

根据约定，数据表的列名使用的是 struct 字段名的 `蛇形命名`

```
type User struct {
  ID        uint      // 列名是 `id`
  Name      string    // 列名是 `name`
  Birthday  time.Time // 列名是 `birthday`
  CreatedAt time.Time // 列名是 `created_at`
}
```

您可以使用 `column` 标签或 [`命名策略`](https://gorm.io/zh_CN/docs/conventions.html#naming_strategy) 来覆盖列名

```
type Animal struct {
  AnimalID int64     `gorm:"column:beast_id"`         // 将列名设为 `beast_id`
  Birthday time.Time `gorm:"column:day_of_the_beast"` // 将列名设为 `day_of_the_beast`
  Age      int64     `gorm:"column:age_of_the_beast"` // 将列名设为 `age_of_the_beast`
}
```

### 时间戳追踪

GORM 约定使用 `CreatedAt`、`UpdatedAt` 追踪创建/更新时间。如果您定义了这种字段，GORM 在创建、更新时会自动填充 [当前时间](https://gorm.io/zh_CN/docs/gorm_config.html#now_func)

要使用不同名称的字段，您可以配置 `autoCreateTime`、`autoUpdateTime` 标签。

如果您想要保存 UNIX（毫/纳）秒时间戳，而不是 time，您只需简单地将 `time.Time` 修改为 `int` 即可

```
type User struct {
  CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
  UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
  Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
  Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
  Created   int64 `gorm:"autoCreateTime"`      // 使用时间戳秒数填充创建时间
}
```

##### CreatedAt

对于有 `CreatedAt` 字段的模型，创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

```
// 后续想要修改该值，可以使用 `Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

你可以通过将 `autoCreateTime` 标签置为 `false` 来禁用时间戳追踪，例如：

```
type User struct {
  CreatedAt time.Time `gorm:"autoCreateTime:false"`
}
```

##### UpdatedAt

对于有 `UpdatedAt` 字段的模型，更新记录时，将该字段的值设为当前时间。创建记录时，如果该字段值为零值，则将该字段的值设为当前时间

你可以通过将 `autoUpdateTime` 标签置为 `false` 来禁用时间戳追踪，例如：

```
type User struct {
  UpdatedAt time.Time `gorm:"autoUpdateTime:false"`
}
```

**注意** **：**GORM 支持拥有多种类型的时间追踪字段。可以根据 UNIX（毫/纳）秒，查看 [Model](https://gorm.io/zh_CN/docs/models.html#time_tracking) 获取详情

# 结构体定义

### 字段级权限控制

可导出的字段在使用 GORM 进行 CRUD 时拥有全部的权限，此外，GORM 允许您用标签控制字段级别的权限。这样您就可以让一个字段的权限是只读、只写、只创建、只更新或者被忽略

**注意：** 使用 GORM Migrator 创建表时，不会创建被忽略的字段

```
type User struct {
  Name string `gorm:"<-:create"` // 允许读和创建
  Name string `gorm:"<-:update"` // 允许读和更新
  Name string `gorm:"<-"`        // 允许读和写（创建和更新）
  Name string `gorm:"<-:false"`  // 允许读，禁止写
  Name string `gorm:"->"`        // 只读（除非有自定义配置，否则禁止写）
  Name string `gorm:"->;<-:create"` // 允许读和创建
  Name string `gorm:"->:false;<-:create"` // 仅创建（禁止从 db 读）
  Name string `gorm:"-"`  // 通过 struct 读写会忽略该字段
  Name string `gorm:"-:all"`        // 通过 struct 读写、迁移会忽略该字段
  Name string `gorm:"-:migration"`  // 通过 struct 迁移会忽略该字段
}
```

### 嵌入结构体

对于匿名字段，GORM 会将其字段包含在父结构体中，例如：

```
type Author struct {
  Name  string
  Email string
}

type Blog struct {
  Author
  ID      int
  Upvotes int32
}
// equals
type Blog struct {
  ID      int64
  Name    string
  Email   string
  Upvotes int32
}
```

对于正常的结构体字段，你也可以通过标签 `embedded` 将其嵌入，例如：

```
type Author struct {
    Name  string
    Email string
}

type Blog struct {
  ID      int
  Author  Author `gorm:"embedded"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID    int64
  Name  string
  Email string
  Upvotes  int32
}
```

并且，您可以使用标签 `embeddedPrefix` 来为 db 中的字段名添加前缀，例如：

```
type Blog struct {
  ID      int
  Author  Author `gorm:"embedded;embeddedPrefix:author_"`
  Upvotes int32
}
// 等效于
type Blog struct {
  ID          int64
  AuthorName string
  AuthorEmail string
  Upvotes     int32
}
```

### 字段标签

| 标签名                 | 说明                                                                                                                                                                                                                                                                                                                                                        |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| column                 | 指定 db 列名                                                                                                                                                                                                                                                                                                                                                |
| type                   | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| serializer             | 指定将数据序列化或反序列化到数据库中的序列化器, 例如:`serializer:json/gob/unixtime`                                                                                                                                                                                                                                                                       |
| size                   | 定义列数据类型的大小或长度，例如 `size: 256`                                                                                                                                                                                                                                                                                                              |
| primaryKey             | 将列定义为主键                                                                                                                                                                                                                                                                                                                                              |
| unique                 | 将列定义为唯一键                                                                                                                                                                                                                                                                                                                                            |
| default                | 定义列的默认值                                                                                                                                                                                                                                                                                                                                              |
| precision              | 指定列的精度                                                                                                                                                                                                                                                                                                                                                |
| scale                  | 指定列大小                                                                                                                                                                                                                                                                                                                                                  |
| not null               | 指定列为 NOT NULL                                                                                                                                                                                                                                                                                                                                           |
| autoIncrement          | 指定列为自动增长                                                                                                                                                                                                                                                                                                                                            |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                                                                                                                                                                                                                                                                                                                            |
| embedded               | 嵌套字段                                                                                                                                                                                                                                                                                                                                                    |
| embeddedPrefix         | 嵌入字段的列名前缀                                                                                                                                                                                                                                                                                                                                          |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano`                                                                                                                                                                                                              |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪时间戳秒数，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli`                                                                                                                                                                                                        |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看[索引](https://gorm.io/zh_CN/docs/indexes.html) 获取详情                                                                                                                                                                                                                                           |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                                                                                                                                                                                                                                                                                                                       |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://gorm.io/zh_CN/docs/constraints.html) 获取详情                                                                                                                                                                                                                                                       |
| <-                     | 设置字段写入的权限，`<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限                                                                                                                                                                                                                                              |
| ->                     | 设置字段读的权限，`->:false` 无读权限                                                                                                                                                                                                                                                                                                                     |
| -                      | 忽略该字段，`-` 表示无读写，`-:migration` 表示无迁移权限，`-:all` 表示无读写迁移权限                                                                                                                                                                                                                                                                  |
| comment                | 迁移时为字段添加注释                                                                                                                                                                                                                                                                                                                                        |

# 连接数据库

GORM 官方支持的数据库类型有：MySQL, PostgreSQL, SQLite, SQL Server 和 TiDB。

这里只以个人常用的MySQL为例。

```
import (
  "gorm.io/driver/mysql"
  "gorm.io/gorm"
)
func main() {
  dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
  db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

**注意：** 想要正确的处理 `time.Time` ，您需要带上 `parseTime` 参数， ([更多参数](https://github.com/go-sql-driver/mysql#parameters)) 要支持完整的 UTF-8 编码，您需要将 `charset=utf8` 更改为 `charset=utf8mb4` 查看 [此文章](https://mathiasbynens.be/notes/mysql-utf8mb4) 获取详情

MySQL 驱动程序提供了 [一些高级配置](https://github.com/go-gorm/mysql) 可以在初始化过程中使用。

# 创建

### 创建表

### AutoMigrate

`func (db *DB) AutoMigrate(dst ...interface{}) error`

```
为给定模型运行自动迁移的方法，用于同步数据库表结构与模型结构。
主要功能包括：
如果表不存在，则自动创建表。
如果表存在，但字段与模型定义不一致，则更新表结构。
不会删除或修改现有表中的数据。
```

```
// 创建单个表
err = db.AutoMigrate(&User{})
// 创建多个表
err = db.AutoMigrate(
	&model.User{},
	&model.Role{},
	&model.Group{},
	&model.Menu{},
	&model.Api{},
	&model.OperationLog{},
	&model.FieldRelation{},
)
```

### 创建记录

##### 创建单项

```
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}
result := db.Create(&user) // 通过数据的指针来创建
```

```
会返回主键和result
user.ID             // 返回插入数据的主键
result.Error        // 返回 error
result.RowsAffected // 返回插入记录的条数
```

##### 创建多项

```
users := []*User{
    {Name: "Jinzhu", Age: 18, Birthday: time.Now()},
    {Name: "Jackson", Age: 19, Birthday: time.Now()},
}

result := db.Create(users) // pass a slice to insert multiple row

result.Error        // returns error
result.RowsAffected // returns inserted records count
```

**注意：**你无法向 ‘create’ 传递结构体，所以你应该传入数据的指针

##### 指定字段

创建记录并为指定字段赋值。

```
db.Select("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`name`,`age`,`created_at`) VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")
```

创建记录并忽略传递给 `Omit` 的字段值

```
db.Omit("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`birthday`,`updated_at`) VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")
```

# 查询

### 根据主键检索

如果主键是数字类型，您可以使用 [内联条件](https://gorm.io/zh_CN/docs/query.html#inline_conditions) 来检索对象。 当使用字符串时，需要额外的注意来避免SQL注入；查看 [Security](https://gorm.io/zh_CN/docs/security.html) 部分来了解详情。

```
db.First(&user, 10)
// SELECT * FROM users WHERE id = 10;

db.First(&user, "10")
// SELECT * FROM users WHERE id = 10;

db.Find(&users, []int{1,2,3})
// SELECT * FROM users WHERE id IN (1,2,3);
```

如果主键是字符串(例如像uuid)，查询将被写成如下：

```
db.First(&user, "id = ?", "1b74413f-f3b8-409f-ac47-e8c062e3472a")
// SELECT * FROM users WHERE id = "1b74413f-f3b8-409f-ac47-e8c062e3472a";
```

当目标对象有一个主键值时，将使用主键构建查询条件，例如：

```
var user = User{ID: 10}
db.First(&user)
// SELECT * FROM users WHERE id = 10;

var result User
db.Model(User{ID: 10}).First(&result)
// SELECT * FROM users WHERE id = 10;
```

**NOTE:**  如果您使用 gorm 的特定字段类型（例如 `gorm.DeletedAt`），它将运行不同的查询来检索对象。

```
type User struct {
  ID           string `gorm:"primarykey;size:16"`
  Name         string `gorm:"size:24"`
  DeletedAt    gorm.DeletedAt `gorm:"index"`
}

var user = User{ID: 15}
db.First(&user)
//  SELECT * FROM `users` WHERE `users`.`id` = '15' AND `users`.`deleted_at` IS NULL ORDER BY `users`.`id` LIMIT 1
```

### 检索全部对象

```
result := db.Find(&users)
// SELECT * FROM users;
```

```
result.RowsAffected // returns found records count, equals `len(users)`
result.Error        // returns error
```

### 条件

##### String 条件

```
// 获取第一条符合的记录
db.Where("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE name = 'jinzhu' ORDER BY id LIMIT 1;

// 获取所有复合记录
db.Where("name <> ?", "jinzhu").Find(&users)
// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name IN ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

如果对象设置了主键，条件查询将不会覆盖主键的值，而是用 And 连接条件。 例如：

```
var user = User{ID: 10}
db.Where("id = ?", 20).First(&user)
// SELECT * FROM users WHERE id = 10 and id = 20 ORDER BY id ASC LIMIT 1
```

这个查询将会给出 `record not found`错误 所以，在你想要使用例如 `user` 这样的变量从数据库中获取新值前，需要将例如 `id` 这样的主键设置为nil。

##### Struct & Map 条件

```
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 ORDER BY id LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&users)
// SELECT * FROM users WHERE id IN (20, 21, 22);
```

**NOTE：** 当使用 struct 查询时，GORM 只会查询非零字段，这意味着如果您的字段值为 `0`, `''`, `false`或其他零值，则不会用于构建查询条件，例如：

```
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu";
```

要在查询条件中包含零值，可以使用映射，它将包含所有键值作为查询条件，例如：

```
db.Where(map[string]interface{}{"Name": "jinzhu", "Age": 0}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;
```

##### 指定结构体查询字段

使用结构体进行搜索时，您可以通过将相关字段名称或数据库名称传递给Where()来指定在查询条件中使用结构体中的哪些特定值，例如：

```
db.Where(&User{Name: "jinzhu"}, "name", "Age").Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;

db.Where(&User{Name: "jinzhu"}, "Age").Find(&users)
// SELECT * FROM users WHERE age = 0;
```

##### 内联条件

查询条件可以内联到First和Find等方法中，其方式与Where类似。

```
// 如果是非整数类型则通过主键获取
db.First(&user, "id = ?", "string_primary_key")
// SELECT * FROM users WHERE id = 'string_primary_key';

// 普通 SQL
db.Find(&user, "name = ?", "jinzhu")
// SELECT * FROM users WHERE name = "jinzhu";

db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20) )
// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// 结构体
db.Find(&users, User{Age: 20})
// SELECT * FROM users WHERE age = 20;

// Map
db.Find(&users, map[string]interface{}{"age": 20})
// SELECT * FROM users WHERE age = 20;
```

### 选择特定字段

Select允许您指定要从数据库检索的字段。否则，GORM 将默认选择所有字段。

```
db.Select("name", "age").Find(&users)
// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
// SELECT COALESCE(age,'42') FROM users;
```

### Limit & Offset 限制和偏移

Limit指定要检索的最大记录数Offset指定开始返回记录之前要跳过的记录数

```
db.Limit(3).Find(&users)
// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
// SELECT * FROM users LIMIT 10; (users1)
// SELECT * FROM users; (users2)

db.Offset(3).Find(&users)
// SELECT * FROM users OFFSET 3;

db.Limit(10).Offset(5).Find(&users)
// SELECT * FROM users OFFSET 5 LIMIT 10;

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
// SELECT * FROM users OFFSET 10; (users1)
// SELECT * FROM users; (users2)
```

有关如何制作分页器的详细信息，请参阅[分页](https://gorm.io/zh_CN/docs/scopes.html#pagination)

# 更新

### 保存所有字段

`Save` 会保存所有的字段，即使字段是零值

```
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

`Save `是一个组合函数。 如果保存值不包含主键，它将执行 `Create`，否则它将执行 `Update` (包含所有字段)。

```
db.Save(&User{Name: "jinzhu", Age: 100})
// INSERT INTO `users` (`name`,`age`,`birthday`,`update_at`) VALUES ("jinzhu",100,"0000-00-00 00:00:00","0000-00-00 00:00:00")

db.Save(&User{ID: 1, Name: "jinzhu", Age: 100})
// UPDATE `users` SET `name`="jinzhu",`age`=100,`birthday`="0000-00-00 00:00:00",`update_at`="0000-00-00 00:00:00" WHERE `id` = 1
```

**NOTE：**不要将 `Save` 和 `Model`一同使用, 这是未定义的行为 。

### 更新单个列

当使用 `Update` 更新单列时，需要有一些条件，否则将会引起 `ErrMissingWhereClause` 错误，查看 [阻止全局更新](https://gorm.io/zh_CN/docs/update.html#block_global_updates) 了解详情。 当使用 `Model` 方法，并且它有主键值时，主键将会被用于构建条件，例如：

```
// 根据条件更新
db.Model(&User{}).Where("active = ?", true).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE active=true;

// user 的 ID 是 `111`
db.Model(&user).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// 根据条件和 model 的值进行更新
db.Model(&user).Where("active = ?", true).Update("name", "hello")
// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
```

### 更新多列

`Updates` 方法支持 `struct` 和 `map[string]interface{}` 参数。当使用 `struct` 更新时，默认情况下GORM 只会更新非零值的字段

```
// 根据 `struct` 更新属性，只会更新非零值的字段
db.Model(&user).Updates(User{Name: "hello", Age: 18, Active: false})
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// 根据 `map` 更新属性
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

**注意：** 使用 struct 更新时, GORM 将只更新非零值字段。 你可能想用 `map` 来更新属性，或者使用 `Select` 声明字段来更新

### 更新选定字段

如果您想要在更新时选择、忽略某些字段，您可以使用 `Select`、`Omit`

```
// 选择 Map 的字段
// user 的 ID 是 `111`:
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// 选择 Struct 的字段（会选中零值的字段）
db.Model(&user).Select("Name", "Age").Updates(User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;

// 选择所有字段（选择包括零值字段的所有字段）
db.Model(&user).Select("*").Updates(User{Name: "jinzhu", Role: "admin", Age: 0})

// 选择除 Role 外的所有字段（包括零值字段的所有字段）
db.Model(&user).Select("*").Omit("Role").Updates(User{Name: "jinzhu", Role: "admin", Age: 0})
```

# 删除

### 删除一条记录

删除一条记录时，删除对象需要指定主键，否则会触发 [批量删除](https://gorm.io/zh_CN/docs/delete.html#batch_delete)，例如：

```
// email 的 ID 是 `10`
db.Delete(&email)
// DELETE from emails where id = 10;

// 带额外条件的删除
db.Where("name = ?", "jinzhu").Delete(&email)
// DELETE from emails where id = 10 AND name = "jinzhu";
```

### 根据主键删除

GORM 允许通过主键(可以是复合主键)和内联条件来删除对象，它可以使用数字（如以下例子。也可以使用字符串——译者注）。查看 [查询-内联条件（Query Inline Conditions）](https://gorm.io/zh_CN/docs/query.html#inline_conditions) 了解详情。

```
db.Delete(&User{}, 10)
// DELETE FROM users WHERE id = 10;

db.Delete(&User{}, "10")
// DELETE FROM users WHERE id = 10;

db.Delete(&users, []int{1,2,3})
// DELETE FROM users WHERE id IN (1,2,3);
```

### 软删除

如果你的模型包含了 `gorm.DeletedAt`字段（该字段也被包含在 `gorm.Model`中），那么该模型将会自动获得软删除的能力！

当调用 `Delete`时，GORM并不会从数据库中删除该记录，而是将该记录的 `DeleteAt`设置为当前时间，而后的一般查询方法将无法查找到此条记录。

```
// user's ID is `111`
db.Delete(&user)
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// Batch Delete
db.Where("age = ?", 20).Delete(&User{})
// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// Soft deleted records will be ignored when querying
db.Where("age = 20").Find(&user)
// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
```

如果你并不想嵌套 `gorm.Model`，你也可以像下方例子那样开启软删除特性：

```
type User struct {
  ID      int
  Deleted gorm.DeletedAt
  Name    string
}
```

##### 查找被软删除的记录

你可以使用 `Unscoped`来查询到被软删除的记录

```
db.Unscoped().Where("age = 20").Find(&users)
// SELECT * FROM users WHERE age = 20;
```

##### 永久删除

你可以使用 `Unscoped`来永久删除匹配的记录

```
db.Unscoped().Delete(&order)
// DELETE FROM orders WHERE id=10;
```
