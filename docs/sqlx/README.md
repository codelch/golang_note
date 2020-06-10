# SQLX使用指南

Go中`database/sql`包操作数据库比较复杂，使用第三方库`sqlx`能够简化操作，提高开发效率。

## 安装SQLX

```go
go get github.com/jmoiron/sqlx
```

下载完成后导入使用：

```go
import (
	"database/sql"
	"github.com/jmoiron/sqlx"
)
```



## 连接数据库

```go
db, err = sqlx.Connect(driverName, dataSourceName strin)
db = sqlx.MustConnect(driverName, dataSourceName strin) // 连接不成功直接panic
```

示例代码：

```go
  dsn := "root:root@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		fmt.Printf("connect DB failed, err:%v\n", err)
		return
	}
	// 设置最大连接数
	db.SetMaxOpenConns(20)
	// 设置最大空闲数
	db.SetMaxIdleConns(10)
```

## 查询数据

### 查询单条数据

```go
func (db *DB) Get(dest interface{}, query string, args ...interface{}) error 
```

示例代码

```go
	sqlStr := "select id, name, age from user where id=?"
	var u user
	err := db.Get(&u, sqlStr, 1) // 直接传入结构体，自动赋值
	if err != nil {
		fmt.Printf("get failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.ID, u.Name, u.Age)
```

### 查询多条数据

```go
func (db *DB) Select(dest interface{}, query string, args ...interface{})
func (db *DB) NamedQuery(query string, arg interface{}) (*Rows, error)
```

示例代码：

```go
	sqlStr := "select id, name, age from user where id > ?"
	var users []user
	err := db.Select(&users, sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	fmt.Printf("users:%#v\n", users)
```

```go
rows, err = db.NamedQuery(`SELECT * FROM users WHERE name=:name`, map[string]interface{}{"name": "Q1mi"})

type User struct {
	Name string `db:"name"`
	Age  int    `db:"age"`
}
u := User{
	Name: "张三",
}
// 使用结构体命名查询，根据结构体字段的 db tag进行映射
rows, err = db.NamedQuery(`SELECT * FROM users WHERE name=:name`, u)
```

### 插入、更新和删除

```go
func (db *DB) exec(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (Result, error) 
func (db *DB) NamedExec(query string, arg interface{}) (sql.Result, error)
```

插入示例代码：

```go
sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "张三", 22)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	newID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", newID)
```

```go
_, err = db.NamedExec(`INSERT INTO users (name,age) VALUES (:name,:age)`,
		map[string]interface{}{
			"name": "张三",
			"age": 33,
		})
```

修改示例代码：

```go
sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 33, 1)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
```

删除示例代码：

```go
sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 6)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
```



