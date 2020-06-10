# GORM操作MySQL

## 安装

```go
go get -u github.com/jinzhu/gorm
```

安装成功后导入：

```go
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
)
```

## 初始化数据库

### 连接数据库

```go
	db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
  defer db.Close()
```

### 自动迁移

> 自动迁移模式会根据结构体信息自动创建表和更新表中字段信息，将表保持更新到最新。
>
> **注意**：自动迁移**仅仅**会创建表，缺少列和索引，并且不会改变现有列的类型或删除未使用的列以保护数据。

```go
// 开启自动迁移，自动创建表，并更新结构体中字段
db.AutoMigrate(&User{})
db.AutoMigrate(&User{}, &Product{}, &Order{})

// 创建表时添加表后缀，引擎使用InnoDB
db.Set("gorm:table_options", "ENGINE=InnoDB").AutoMigrate(&User{})
```

### 检查表是否存在

```go
// 检查模型`User`表是否存在
db.HasTable(&User{})

// 检查表`users`是否存在
db.HasTable("users")
```

### 创建表

```go
// 为模型`User`创建表
db.CreateTable(&User{})

// 创建表`users'时将“ENGINE = InnoDB”附加到SQL语句
db.Set("gorm:table_options", "ENGINE=InnoDB").CreateTable(&User{})
```

### 删除表

```go
// 删除模型`User`的表
db.DropTable(&User{})

// 删除表`users`
db.DropTable("users")

// 删除模型`User`的表和表`products`
db.DropTableIfExists(&User{}, "products")
```

### 修改列

修改列的类型为给定值

```go
// 修改模型`User`的description列的数据类型为`text`
db.Model(&User{}).ModifyColumn("description", "text")
```

### 删除列

```go
// 删除模型`User`的description列
db.Model(&User{}).DropColumn("description")
```

### 添加外键

```go
// 添加主键
// 1st param : 外键字段
// 2nd param : 外键表(字段)
// 3rd param : ONDELETE
// 4th param : ONUPDATE
db.Model(&User{}).AddForeignKey("city_id", "cities(id)", "RESTRICT", "RESTRICT")
```

### 索引

```go
// 为`name`列添加索引`idx_user_name`
db.Model(&User{}).AddIndex("idx_user_name", "name")

// 为`name`, `age`列添加索引`idx_user_name_age`
db.Model(&User{}).AddIndex("idx_user_name_age", "name", "age")

// 添加唯一索引
db.Model(&User{}).AddUniqueIndex("idx_user_name", "name")

// 为多列添加唯一索引
db.Model(&User{}).AddUniqueIndex("idx_user_name_age", "name", "age")

// 删除索引
db.Model(&User{}).RemoveIndex("idx_user_name")
```

## 构建模型

### 模型定义

我们来看一个模型的例子：

```go
type User struct {
    gorm.Model
    Birthday     time.Time
    Age          int
    Name         string  `gorm:"size:255"`       // string默认长度为255, 使用这种tag重设。
    Num          int     `gorm:"AUTO_INCREMENT"` // 自增

    CreditCard        CreditCard      // One-To-One (拥有一个 - CreditCard表的UserID作外键)
    Emails            []Email         // One-To-Many (拥有多个 - Email表的UserID作外键)

    BillingAddress    Address         // One-To-One (属于 - 本表的BillingAddressID作外键)
    BillingAddressID  sql.NullInt64

    ShippingAddress   Address         // One-To-One (属于 - 本表的ShippingAddressID作外键)
    ShippingAddressID int

    IgnoreMe          int `gorm:"-"`   // 忽略这个字段
    Languages         []Language `gorm:"many2many:user_languages;"` // Many-To-Many , 'user_languages'是连接表
}

type Email struct {
    ID      int
    UserID  int     `gorm:"index"` // 外键 (属于), tag `index`是为该列创建索引
    Email   string  `gorm:"type:varchar(100);unique_index"` // `type`设置sql类型, `unique_index` 为该列设置唯一索引
    Subscribed bool
}

type Address struct {
    ID       int
    Address1 string         `gorm:"not null;unique"` // 设置字段为非空并唯一
    Address2 string         `gorm:"type:varchar(100);unique"`
    Post     sql.NullString `gorm:"not null"`
}

type Language struct {
    ID   int
    Name string `gorm:"index:idx_name_code"` // 创建索引并命名，如果找到其他相同名称的索引则创建组合索引
    Code string `gorm:"index:idx_name_code"` // `unique_index` also works
}

type CreditCard struct {
    gorm.Model
    UserID  uint
    Number  string
}
```

> 可以看到模型就是结构体：
>
> * 结构体名称就是表的名称（默认，可以修改）；
> * 结构体字段名就是表中的字段名（驼峰命名会转为蛇形命名）；
> * 结构体字段类型相当于表字段类型（默认，可以修改）；
> * 结构体中引用其他结构体相当于创建表关联，并创建外键，关联可以一对一，也可以一对多；
> * 自定义信息可以写在tag中，包括：
>   * 字段属性，如：主键，自增，唯一，非空，类型，大小等
>   * 重新定义字段名称
>   * 重新定义字段属性
>   * 在表中忽略字段
>   * 关联表的名称
>   * 定义索引
>   * ...

### 模型的约定

#### gorm.Model 结构体

> `gorm.Model`是gorm包中定义的基本模型，包括字段`ID`，`CreatedAt`，`UpdatedAt`，`DeletedAt`，你可以将它嵌入你的模型，或者只写你想要的字段

```go
// 基本模型的定义
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}

// 添加字段 `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt`
type User struct {
  gorm.Model
  Name string
}

// 只需要字段 `ID`, `CreatedAt`
type User struct {
  ID        uint
  CreatedAt time.Time
  Name      string
}
```