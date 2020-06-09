# Go操作MySQL

## 连接数据库

### 下载驱动

Go中自带的`database/sql`包中只提供了必要的接口，并没有提供所需要的数据库驱动，所以需要下载驱动：

> 下载MySQL驱动
>
 ```go
 go get -u github.com/go-sql-driver/mysql
 ```

驱动下载完成后需要进行导入使用：

```go
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```

### 初始化连接

使用`open`函数连接数据库：

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

> 使用`sql.Open()`获得数据库的句柄。`database/sql`程序包在后台管理一个连接池，并且在需要它们之前不会打开任何连接。因此`sql.Open()`不会直接打开连接。使用`sql.Open()`只会验证DSN格式是否正确，连接数据库失败时（用户名，密码不正确）不会返回错误。如果要在进行查询之前进行检查，可以使用[`db.Ping()`](http://golang.org/pkg/database/sql/#DB.Ping)。
>
> DSN格式如下：
>
> ​	username:password@protocol(address)/dbname?param=value

验证数据库是否连接成功：
```go
func (db *DB) Ping() error
```

我们还可以设置最大连接数和最大闲置连接数：

```go
// 设置最大连接数
func (db *DB) SetMaxOpenConns(n int)
// 设置最大闲置连接数
func (db *DB) SetMaxIdleConns(n int)
```

示例代码：

```go
	dsn := "root:root@tcp(127.0.0.1:3306)/sql_test"
	//连接数据库
	db, err = sql.Open("mysql", dsn) // 不会校验数据库账号密码是否正确
	if err != nil {  // dsn格式不正确时会报错
		fmt.Printf("dsn %s invalid, err:%v\n",dsn, err)
		return err
	}
	err = db.Ping() // 尝试连接数据库
	if err != nil {
		fmt.Printf("open %s failed, err:%v\n",dsn, err)
		return err
	}
	fmt.Println("连接数据库成功")
```

## 查询

### 查询单条数据

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

> `db.QueryRow`执行的查询预期最多返回一行。`QueryRow`始终返回非`nil`值。错误将一直延迟到调用`Row`的`Scan`方法。如果查询未选择任何行，则“行扫描”将返回`ErrNoRows`。否则，*行扫描将扫描所选的第一行，并丢弃其余的行。

示例代码：

```go
sqlStr := "select id, name, age from user where id=?"
var u User
// Scan方法：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
err := db.QueryRow(sqlStr, id).Scan(&u.id, &u.name, &u.age)
if err != nil {
   fmt.Printf("scan failed, err:%v\n", err)
   return
}
fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
```

### 查询多条数据

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

> 查询执行返回行的查询，通常为SELECT。args用于查询中的任何占位符参数。

示例代码：

```go
	sqlStr := "select * from user where id>?"
	rows, err := db.Query(sqlStr, 1)
	if err != nil {
    fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 关闭rows，释放连接
	defer rows.Close()
	// 循环读取结果集
	for rows.Next() {
		var u1 User
		err := rows.Scan(&u1.id, &u1.name, &u1.age)
		if err != nil {

		}
		fmt.Printf("id:%d name:%s age:%d\n", u1.id, u1.name, u1.age)
	}
```

## 插入、修改和删除

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

> Exec执行查询返回`Result`是对已执行的SQL命令的总结,而不是返回任何行。args用于查询中的任何占位符参数。用于执行插入，修改和删除等。

插入数据示例：

```go
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "张三", 22)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	newID, err := ret.LastInsertId() // 获取新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", newID)
```

修改数据示例：

```go
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 24, 1)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 获取影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
```

删除数据示例：

```go
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 1)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 获取影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
```

## 预处理

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

> Prepare为以后的查询或执行创建一个准备好的语句。可以从返回的语句中同时运行多个查询或执行。当不再需要该语句时，调用方必须调用该语句的Close方法。

查询示例代码：

```go
sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(1)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
```

修改示例代码：

插入修改和删除非常类似，只是·sql·不同

```go
sqlStr := "update user set age=? where id=?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec(33, 1)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	_, err = stmt.Exec(44, 2)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	fmt.Println("update success.")
```

## 事务

开始事务：

```go
func (db *DB) Begin() (*Tx, error)
```

提交事务：

```go
func (tx *Tx) Commit() error
```

事务回滚：

```go
func (tx *Tx) Rollback() error
```

示例代码：

```go
tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Printf("begin trans failed, err:%v\n", err)
		return
	}
	sqlStr1 := "Update user set age=18 where id=?"
	_, err = tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback()
		fmt.Printf("exec sql1 failed, err:%v\n", err)
		return
	}
	sqlStr2 := "Update user set age=33 where id=?"
	_, err = tx.Exec(sqlStr2, 4)
	if err != nil {
		tx.Rollback()
		fmt.Printf("exec sql2 failed, err:%v\n", err)
		return
	}
	err = tx.Commit() // 提交事务
	if err != nil {
		tx.Rollback()
		fmt.Printf("commit failed, err:%v\n", err)
		return
	}
	fmt.Println("exec trans success!")

```