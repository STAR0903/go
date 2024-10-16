# sqlx

#### 下载依赖

`go get github.com/jmoiron/sqlx`

sqlx库

`go get -u github.com/go-sql-driver/mysql `

MySQL驱动程序，包括增删改查

#### 初始化连接

```
var db *sqlx.DB

func initDB() (err error) {
	dsn := "root:123456@tcp(127.0.0.1:3306)/gin-pro?charset=utf8mb4"
	// sqlx.MustConnect()连接不成功就panic
	// sqlx.Connect()函数相当于实现了sql.Open()和db.Ping()
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Println(err)
		return
	}
	// db.SetMaxOpenConns() 设置与数据库最大连接数
	// db.SetMaxIdleConns() 设置与数据库最大闲暇连接数
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return

```

#### 基本操作

###### 查询

```
// 查询单条
func queryOne() {
	//sql语句
	sqlStr := "select id, name, age from user where id=?"
	var u user
	//根据sql语句查询并且数据保存到传入的结构体，返回错误
	err := db.Get(&u, sqlStr, 1)
	if err != nil {
		fmt.Println("get:", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
}

// 查询多条
func queryMore() {
	//sql语句
	sqlStr := "select id, name, age from user where id > ?"
	var users []user
	//根据sql语句查询并且数据保存到传入的结构体切片，返回错误
	err := db.Select(&users, sqlStr, 0)
	if err != nil {
		fmt.Println("query:", err)
		return
	}
	fmt.Println(users)
}
```

###### 增删改

基本与sql保持一致

```
func insert() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "sqlx", 119)
	if err != nil {
		fmt.Println("insert:", err)
		return
	}
	new, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Println("newid:", err)
		return
	}
	fmt.Println(new)
}
func update() {
	sqlStr := "update user set name = ? where id = ?"
	_, err := db.Exec(sqlStr, "update", 3)
	if err != nil {
		fmt.Println("insert:", err)
		return
	}
}
func delete() {
	sqlStr := "delete from user where id= ?"
	_, err := db.Exec(sqlStr, 5)
	if err != nil {
		fmt.Println("insert:", err)
		return
	}
}
```

#### NamedExec

###### 基本运用

`DB.NamedExec`方法用来绑定SQL语句与结构体或map中的同名字段

```
func insert() (err error) {
	sqlStr := "INSERT INTO user (name,age) VALUES (:name,:age)"
	_, err = db.NamedExec(sqlStr,
		map[string]interface{}{
			"name": "demo13insert",
			"age":  28,
		})
	if err != nil {
		fmt.Println("insert:", err)
		return
	}
	return
}
```

###### 实现批量插入

**注意** ：该功能需1.3.1版本以上，并且1.3.1版本目前还有点问题，sql语句最后不能有空格和 `;`，详见[issues/690](https://github.com/jmoiron/sqlx/issues/690)。

使用 `NamedExec`实现批量插入的代码如下：

```
// BatchInsertUsers3 使用NamedExec实现批量插入
func BatchInsertUsers3(users []*User) error {
	_, err := DB.NamedExec("INSERT INTO user (name, age) VALUES (:name, :age)", users)
	return err
}

```

#### NamedQuery

与 `DB.NamedExec`同理，这里是支持查询

```
func Query() {
	sqlStr := "SELECT * FROM user WHERE name=:name"
	// 使用map做命名查询
	rows, err := db.NamedQuery(sqlStr, map[string]interface{}{"name": "star"})
	if err != nil {
		fmt.Println("query:", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var u user
		//rows.StructScan()  查询结果直接存入结构体
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Println("scan :", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}

	u := user{
		Name: "star",
	}
	// 使用结构体命名查询，根据结构体字段的 db tag 进行映射
	rows, err = db.NamedQuery(sqlStr, u)
	if err != nil {
		fmt.Println("query:", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var u user
		err := rows.StructScan(&u)
		if err != nil {
			fmt.Println("scan :", err)
			continue
		}
		fmt.Printf("user:%#v\n", u)
	}
}
```

#### 事务操作

对于事务操作，我们可以使用 `sqlx`中提供的 `db.Beginx()`和 `tx.Exec()`方法。示例代码如下：

```
func transactionDemo2()(err error) {
	// 开启事务
	tx, err := db.Beginx()
	if err != nil {
		fmt.Printf("begin trans failed, err:%v\n", err)
		return err
	}
	//defer判读是否有painc或者error，有则回滚，无则提交
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			fmt.Println("rollback")
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // err is nil; if Commit returns error update err
			fmt.Println("commit")
		}
	}()

	sqlStr1 := "Update user set age=20 where id=?"

	rs, err := tx.Exec(sqlStr1, 1)
	if err!= nil{
		return err
	}
	n, err := rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	sqlStr2 := "Update user set age=50 where i=?"
	rs, err = tx.Exec(sqlStr2, 5)
	if err!=nil{
		return err
	}
	n, err = rs.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr1 failed")
	}
	return err
}
```

```
var (
	db *sqlx.DB
	tx *sqlx.Tx
)

type exchange struct {
	In  bool
	Out bool
	Who string
}

func initDB() (err error) {
	dsn := "root:123456@tcp(127.0.0.1:3306)/gin-pro?charset=utf8mb4"
	// MustConnect连接不成功就panic
	// sqlx.Connect()函数相当于实现了sql.Open()和db.Ping()
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Println(err)
		return
	}
	// db.SetMaxOpenConns() 设置与数据库最大连接数
	// db.SetMaxIdleConns() 设置与数据库最大闲暇连接数
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return
}

// 查询单条
func queryOne(who string) exchange {
	//sql语句
	sqlStr := "select `in`,`out`,`who` from exchange where who=?"
	var u exchange
	//根据sql语句查询并且数据保存到传入的结构体，返回错误
	err := tx.Get(&u, sqlStr, who)
	if err != nil {
		fmt.Println("get:", err)
		return u
	}
	return u
}
func queryOneAfter(who string) exchange {
	//sql语句
	sqlStr := "select `in`,`out`,`who` from exchange where who=?"
	var u exchange
	//根据sql语句查询并且数据保存到传入的结构体，返回错误
	err := db.Get(&u, sqlStr, who)
	if err != nil {
		fmt.Println("get:", err)
		return u
	}
	return u
}
func update(in bool, out bool, who string) (err error) {
	sqlStr := "update `exchange` set `in`=?,`out`=? where `who` = ?"
	_, err = tx.Exec(sqlStr, in, out, who)
	if err != nil {
		fmt.Println("insert:", err)
		return
	}
	return
}

func main() {
	err := initDB()
	if err != nil {
		fmt.Println("initDB:", err)
	}
	defer func() {
		if p := recover(); p != nil {
			tx.Rollback()
			panic(p) // re-throw panic after Rollback
		} else if err != nil {
			fmt.Println("rollback")
			tx.Rollback() // err is non-nil; don't change it
		} else {
			err = tx.Commit() // err is nil; if Commit returns error update err
			fmt.Println("commit")
		}
	}()
	var a, b exchange
	tx, err = db.Beginx()
	err = update(true, false, "a")
	err = update(false, true, "b")
	a = queryOne("a")
	b = queryOne("b")
	fmt.Println("a:", a.In, a.Out, a.Who)
	fmt.Println("b:", b.In, b.Out, b.Who)
	//通过条件判断回滚还是提交
	//一般tx.Rollback()用于错误处理相关，这里只是演示效果
	if a.In {
		tx.Rollback()
		fmt.Println("rollback")
	} else {
		tx.Commit()
	}
	a = queryOneAfter("a")
	b = queryOneAfter("b")
	fmt.Println("a:", a.In, a.Out, a.Who)
	fmt.Println("b:", b.In, b.Out, b.Who)
}
```

#### sqlx.In

`sqlx.In`是 `sqlx`提供的一个非常方便的函数。

###### 批量插入

前提是需要我们的结构体实现 `driver.Valuer`接口：

```
func (u User) Value() (driver.Value, error) {
	return []interface{}{u.Name, u.Age}, nil
}
func insert() error {
	var users []interface{}
	for i := 0; i < 5; i++ {
		var user User
		user.Age = i
		user.Name = "demo14insert"
		users = append(users, user)
	}
	fmt.Println(users)
	query, args, _ := sqlx.In(
		"INSERT INTO user (name, age) VALUES  (?),(?),(?),(?),(?)",
		users..., // 如果arg实现了 driver.Valuer, sqlx.In 会通过调用 Value()来展开它
	)
	fmt.Println(query) // 查看生成的querystring,等价于sql语句模板
	fmt.Println(args)  // 查看生成的args
	_, err := db.Exec(query, args...)
	return err
}

```

`"INSERT INTO user (name, age) VALUES  (?),(?),(?),(?),(?)"`

传入多少个结构体就要有多少个(?)

###### in查询

查询id在给定id集合中的数据。

```
// QueryByIDs 根据给定ID查询
func QueryByIDs(ids []int)(users []User, err error){
	// 动态填充id
	query, args, err := sqlx.In("SELECT name, age FROM user WHERE id IN (?)", ids)
	if err != nil {
		return
	}
	// sqlx.In 返回带 `?` bindvar的查询语句, 我们使用Rebind()重新绑定。
	// 重新生成对应数据库的查询语句（如PostgreSQL 用 `$1`, `$2` bindvar）
	query = DB.Rebind(query)

	err = DB.Select(&users, query, args...)
	return
}

```



###### in查询和FIND_IN_SET函数

查询id在给定id集合的数据并维持给定id集合的顺序。

```
// QueryAndOrderByIDs 按照指定id查询并维护顺序
func QueryAndOrderByIDs(ids []int)(users []User, err error){
	// 动态填充id
	strIDs := make([]string, 0, len(ids))
	for _, id := range ids {
		strIDs = append(strIDs, fmt.Sprintf("%d", id))
	}
	query, args, err := sqlx.In("SELECT name, age FROM user WHERE id IN (?) ORDER BY FIND_IN_SET(id, ?)", ids, strings.Join(strIDs, ","))
	if err != nil {
		return
	}

	// sqlx.In 返回带 `?` bindvar的查询语句, 我们使用Rebind()重新绑定它
	query = DB.Rebind(query)

	err = DB.Select(&users, query, args...)
	return
}

```

#### bindvars（绑定变量）

查询占位符 `?`在内部称为 **bindvars（查询占位符）** ,它非常重要。你应该始终使用它们向数据库发送值，因为它们可以防止SQL注入攻击。`database/sql`不尝试对查询文本进行任何验证；它与编码的参数一起按原样发送到服务器。除非驱动程序实现一个特殊的接口，否则在执行之前，查询是在服务器上准备的。因此 `bindvars`是特定于数据库的。

`bindvars`的一个常见误解是，它们用来在sql语句中插入值。它们其实仅用于参数化，不允许更改SQL语句的结构。
