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

#### 表名默认是结构体的复数形式

```go
type User struct {} // 默认表名是`users`

// 直接将结构体User的表名设置为`profiles`
func (User) TableName() string {
  return "profiles"
}

// 根据不同的条件设置设置User的表名
func (u User) TableName() string {
    if u.Role == "admin" {
        return "admin_users"
    } else {
        return "users"
    }
}

// 全局禁用表名复数
db.SingularTable(true) // 如果设置为true,`User`的默认表名为`user`,使用`TableName`设置的表名不受影响
```

#### 表名添加前/后缀或者自定义规则

```go
gorm.DefaultTableNameHandler = func (db *gorm.DB, defaultTableName string) string  {
    return "prefix_" + defaultTableName;
}
```

#### 重设列名

```go
// 默认列名为蛇形小写
type User struct {
  ID uint             // 列名为 `id`
  Name string         // 列名为 `name`
  Birthday time.Time  // 列名为 `birthday`
  CreatedAt time.Time // 列名为 `created_at`
}

// 重设列名
type Animal struct {
    AnimalId    int64     `gorm:"column:beast_id"`         // 设置列名为`beast_id`
    Birthday    time.Time `gorm:"column:day_of_the_beast"` // 设置列名为`day_of_the_beast`
    Age         int64     `gorm:"column:age_of_the_beast"` // 设置列名为`age_of_the_beast`
}
```

#### ID默认为主键

```go
type User struct {
  ID   uint  // 默认字段`ID`为主键
  Name string
}

// 使用tag`primary_key`用来设置主键
type Animal struct {
  AnimalId int64 `gorm:"primary_key"` // 设置AnimalId为主键
  Name     string
  Age      int64
}
```

#### CreatedAt存储创建时间

```go
db.Create(&user) // 将会设置`CreatedAt`为当前时间

// 要更改它的值, 你需要使用`Update`
db.Model(&user).Update("CreatedAt", time.Now())
```

### UpdatedAt存储修改时间

```go
db.Save(&user) // 将会设置`UpdatedAt`为当前时间
db.Model(&user).Update("name", "jinzhu") // 将会设置`UpdatedAt`为当前时间
```

### DeletedAt存储删除时间

> 删除具有`DeletedAt`字段的数据，不会真正的从数据表中杉树，只是将字段`DeletedAt`设置为删除时的时间，并在查询时无法找到记录

### 数据表关联

#### 设置表关联

```go
// User 关联 CreditCard, UserID 为外键
type User struct {
    gorm.Model
    CreditCard   CreditCard
}

type CreditCard struct {
    gorm.Model
    UserID   uint
    Number   string
}

// SELECT * FROM credit_cards WHERE user_id = CreditCard,UserID; 
// 当字段名与变量的类型名相同时
db.Model(&user).Related(&card)

// 当字段名与变量的类型名相同时，需要设置所关联的字段名称
// CreditCardStruct是user的字段名称，这意味着获得user的CreditCard关系并将其填充到变量
type User struct {
    gorm.Model
    CreditCardStruct   CreditCard
}
db.Model(&user).Related(&card, "CreditCardStruct")
```

#### 默认设置外键

```go
// `User`关联`Profile`, `ProfileID`字段为外键
type User struct {
  gorm.Model
  Profile   Profile
  ProfileID int
}

type Profile struct {
  gorm.Model
  Name string
}

//  SELECT * FROM profiles WHERE id = User.ProfileID; 利用user中的ProfileID查询Profile的数据，可转化为下面的写法
db.Model(&user).Related(&profile)
```

#### 自定义外键名称

```go
type Profile struct {
    gorm.Model
    Name string
}
// 使用tag设置外键名称
type User struct {
    gorm.Model
    Profile      Profile `gorm:"ForeignKey:ProfileRefer"` // 使用ProfileRefer作为外键
    ProfileRefer int
}
```

#### 自定义外键关联字段

```go
type Profile struct {
    gorm.Model
    Refer string
    Name  string
}
// 使用tag设置外键名称和所关联的外键名称
type User struct {
    gorm.Model
    Profile   Profile `gorm:"ForeignKey:ProfileID;AssociationForeignKey:Refer"`
    ProfileID int
}
```

#### 一对多关联

```go
// 一个User关联多个 email
type User struct {
    gorm.Model
    Emails   []Email
}

type Email struct {
    gorm.Model
    Email   string
    UserID  uint
}

db.Model(&user).Related(&email)
```

#### 多对多关联

```go
// 多个User 和多个 Language关联, 使用 `user_languages` 表连接
type User struct {
    gorm.Model
    Languages         []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
    gorm.Model
    Name string
  	// UserID  uint 多对多下会默认创建
}

// SELECT * FROM "languages" INNER JOIN "user_languages" ON "user_languages"."language_id" = "languages"."id" WHERE "user_languages"."user_id" = Language.UserID
// 由于修改了字段名称，需要指定关联的字段名称
db.Model(&user).Related(&languages, "Languages")
```

#### 多对多自定义关联

```go
type CustomizePerson struct {
  IdPerson string             `gorm:"primary_key:true"`
  Accounts []CustomizeAccount `gorm:"many2many:PersonAccount;ForeignKey:IdPerson;AssociationForeignKey:IdAccount"`
}

type CustomizeAccount struct {
  IdAccount string `gorm:"primary_key:true"`
  Name      string
}
```

#### 多对一关联

```go
type Cat struct {
    Id    int
    Name  string
    Toy   Toy `gorm:"polymorphic:Owner;"`
  }

  type Dog struct {
    Id   int
    Name string
    Toy  Toy `gorm:"polymorphic:Owner;"`
  }

  type Toy struct {
    Id        int
    Name      string
    OwnerId   int
    OwnerType string
  }
```

#### 关联模式

```go
// 开始关联模式
var user User
db.Model(&user).Association("Languages")
// `user`是源，它需要是一个有效的记录（包含主键）
// `Languages`是关系中源的字段名。
// 如果这些条件不匹配，将返回一个错误，检查它：
// db.Model(&user).Association("Languages").Error


// Query - 查找所有相关关联
db.Model(&user).Association("Languages").Find(&languages)


// Append - 添加新的many2many, has_many关联, 会替换掉当前 has_one, belongs_to关联
db.Model(&user).Association("Languages").Append([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Append(Language{Name: "DE"})


// Delete - 删除源和传递的参数之间的关系，不会删除这些参数
db.Model(&user).Association("Languages").Delete([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Delete(languageZH, languageEN)


// Replace - 使用新的关联替换当前关联
db.Model(&user).Association("Languages").Replace([]Language{languageZH, languageEN})
db.Model(&user).Association("Languages").Replace(Language{Name: "DE"}, languageEN)


// Count - 返回当前关联的计数
db.Model(&user).Association("Languages").Count()


// Clear - 删除源和当前关联之间的关系，不会删除这些关联
db.Model(&user).Association("Languages").Clear()
```

## CRUD:读写数据

### 新增数据

#### 默认新增方式

```go
user := User{Name: "Jinzhu", Age: 18, Birthday: time.Now()}

db.NewRecord(user) // => 主键为空返回`true`

db.Create(&user)

db.NewRecord(user) // => 创建`user`后返回`false`
```

#### 设置默认值

> 通过tag设置默认值后，向数据库中插入空值，但是在查询数据时会附带上默认值

```go
type Animal struct {
    ID   int64
    Name string `gorm:"default:'galeone'"`
    Age  int64
}

var animal = Animal{Age: 99, Name: ""}
db.Create(&animal)
// INSERT INTO animals("age") values('99');
// SELECT name from animals WHERE ID=111; // 返回主键为 111
// animal.Name => 'galeone'
```

#### 创建数据之前的回调

```go
// 如需要在创建数据之前设置主键的值
func (user *User) BeforeCreate(scope *gorm.Scope) error {
  scope.SetColumn("ID", uuid.New())
  return nil
}
```

#### 扩展创建选项

```go
// 为Instert语句添加扩展SQL选项
db.Set("gorm:insert_option", "ON CONFLICT").Create(&product)
// INSERT INTO products (name, code) VALUES ("name", "code") ON CONFLICT;
```

### 查询

```go
// 获取第一条记录，按主键排序
db.First(&user)
// SELECT * FROM users ORDER BY id LIMIT 1;

// 获取最后一条记录，按主键排序
db.Last(&user)
// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// 获取所有记录
db.Find(&users)
// SELECT * FROM users;

// 使用主键获取记录
db.First(&user, 10)
```

#### Where查询条件

##### 普通sql

```go
// 获取第一个匹配记录
db.Where("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// 获取所有匹配记录
db.Where("name = ?", "jinzhu").Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu';

db.Where("name <> ?", "jinzhu").Find(&users)

// IN
db.Where("name in (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)

db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
```

##### Struct & Map

> 当使用struct查询时，GORM将只查询那些具有值的字段

```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// 主键的Slice
db.Where([]int64{20, 21, 22}).Find(&users)
// SELECT * FROM users WHERE id IN (20, 21, 22);
```

#### 指定表名

```go
// 使用User结构定义创建`deleted_users`表
db.Table("deleted_users").CreateTable(&User{})

var deleted_users []User
db.Table("deleted_users").Find(&deleted_users)
//// SELECT * FROM deleted_users;

db.Table("deleted_users").Where("name = ?", "jinzhu").Delete()
//// DELETE FROM deleted_users WHERE name = 'jinzhu';
```

#### Not条件查询

```go
db.Not("name", "jinzhu").First(&user)
// SELECT * FROM users WHERE name <> "jinzhu" LIMIT 1;

// Not In
db.Not("name", []string{"jinzhu", "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");

// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
// SELECT * FROM users WHERE id NOT IN (1,2,3);

db.Not([]int64{}).First(&user)
// SELECT * FROM users;

// Plain SQL
db.Not("name = ?", "jinzhu").First(&user)
// SELECT * FROM users WHERE NOT(name = "jinzhu");

// Struct
db.Not(User{Name: "jinzhu"}).First(&user)
// SELECT * FROM users WHERE name <> "jinzhu";
```

#### 内联条件的查询

```go
// 按主键获取
db.First(&user, 23)
// SELECT * FROM users WHERE id = 23 LIMIT 1;

// 简单SQL
db.Find(&user, "name = ?", "jinzhu")
// SELECT * FROM users WHERE name = "jinzhu";

db.Find(&users, "name <> ? AND age > ?", "jinzhu", 20)
// SELECT * FROM users WHERE name <> "jinzhu" AND age > 20;

// Struct
db.Find(&users, User{Age: 20})
// SELECT * FROM users WHERE age = 20;

// Map
db.Find(&users, map[string]interface{}{"age": 20})
// SELECT * FROM users WHERE age = 20;
```

#### Or条件查询

```go
db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';

// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2"}).Find(&users)
// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';

// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2"}).Find(&users)
```

#### 多条件查询

```go
db.Where("name <> ?","jinzhu").Where("age >= ? and role <> ?",20,"admin").Find(&users)
// SELECT * FROM users WHERE name <> 'jinzhu' AND age >= 20 AND role <> 'admin';

db.Where("role = ?", "admin").Or("role = ?", "super_admin").Not("name = ?", "jinzhu").Find(&users)
```

#### 扩展查询选项

```go
// 为Select语句添加扩展SQL选项
db.Set("gorm:query_option", "FOR UPDATE").First(&user, 10)
// SELECT * FROM users WHERE id = 10 FOR UPDATE;
```

#### Select

> 指定要从数据库检索的字段，默认情况下，将选择所有字段;

```go
db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
//// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
//// SELECT COALESCE(age,'42') FROM users;
```

#### Order

> 在从数据库检索记录时指定顺序，将重排序设置为`true`以覆盖定义的条件

```go
db.Order("age desc, name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// Multiple orders
db.Order("age desc").Order("name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// ReOrder
db.Order("age desc").Find(&users1).Order("age", true).Find(&users2)
//// SELECT * FROM users ORDER BY age desc; (users1)
//// SELECT * FROM users ORDER BY age; (users2)
```

#### Limit

> 指定要检索的记录数

```go
db.Limit(3).Find(&users)
//// SELECT * FROM users LIMIT 3;

// Cancel limit condition with -1
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
//// SELECT * FROM users LIMIT 10; (users1)
//// SELECT * FROM users; (users2)
```

#### Count

> 获取模型的记录数

```go
db.Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Find(&users).Count(&count)
//// SELECT * from USERS WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (users)
//// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)

db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)
//// SELECT count(*) FROM users WHERE name = 'jinzhu'; (count)

db.Table("deleted_users").Count(&count)
//// SELECT count(*) FROM deleted_users;
```

#### Group & Having

```go
rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Rows()
for rows.Next() {
    ...
}

rows, err := db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Rows()
for rows.Next() {
    ...
}

type Result struct {
    Date  time.Time
    Total int64
}
db.Table("orders").Select("date(created_at) as date, sum(amount) as total").Group("date(created_at)").Having("sum(amount) > ?", 100).Scan(&results)
```

#### Join

> 指定连接条件

```go
rows, err := db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Rows()
for rows.Next() {
    ...
}

db.Table("users").Select("users.name, emails.email").Joins("left join emails on emails.user_id = users.id").Scan(&results)

// 多个连接与参数
db.Joins("JOIN emails ON emails.user_id = users.id AND emails.email = ?", "jinzhu@example.org").Joins("JOIN credit_cards ON credit_cards.user_id = users.id").Where("credit_cards.number = ?", "123456").Find(&user)
```

#### Pluck

> 将模型中的单个列作为地图查询，如果要查询多个列，可以使用Scan

```go
var ages []int64
db.Find(&users).Pluck("age", &ages)

var names []string
db.Model(&User{}).Pluck("name", &names)

db.Table("deleted_users").Pluck("name", &names)

// 要返回多个列，做这样：
db.Select("name, age").Find(&users)
```

#### Scan

> 将结果扫描到另一个结构中。

```go
type Result struct {
    Name string
    Age  int
}

var result Result
db.Table("users").Select("name, age").Where("name = ?", 3).Scan(&result)

// Raw SQL
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)
```

#### Offset

> 指定在开始返回记录之前要跳过的记录数

```go
db.Offset(3).Find(&users)
//// SELECT * FROM users OFFSET 3;

// Cancel offset condition with -1
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
//// SELECT * FROM users OFFSET 10; (users1)
//// SELECT * FROM users; (users2)
```

#### FirstOrInit

> 获取第一个匹配的记录，或者使用给定的条件初始化一个新的记录（仅适用于struct，map条件）

```go
// Unfound
db.FirstOrInit(&user, User{Name: "non_existing"})
// user -> User{Name: "non_existing"}

// Found
db.Where(User{Name: "Jinzhu"}).FirstOrInit(&user)
// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
db.FirstOrInit(&user, map[string]interface{}{"name": "jinzhu"})
// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
```

#### Attrs

> 如果未找到记录，则使用参数初始化结构

```go
// Unfound
db.Where(User{Name: "non_existing"}).Attrs(User{Age: 20}).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = 'non_existing';
// user -> User{Name: "non_existing", Age: 20}

db.Where(User{Name: "non_existing"}).Attrs("age", 20).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = 'non_existing';
// user -> User{Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "Jinzhu"}).Attrs(User{Age: 30}).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = jinzhu';
// user -> User{Id: 111, Name: "Jinzhu", Age: 20}
```

#### Assign

> 将参数分配给结果，不管它是否被找到

```go
// Unfound
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrInit(&user)
// user -> User{Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "Jinzhu"}).Assign(User{Age: 30}).FirstOrInit(&user)
// SELECT * FROM USERS WHERE name = jinzhu';
// user -> User{Id: 111, Name: "Jinzhu", Age: 30}
```

#### FirstOrCreate

> 获取第一个匹配的记录，或创建一个具有给定条件的新记录（仅适用于struct, map条件）

```go
// Unfound
db.FirstOrCreate(&user, User{Name: "non_existing"})
// INSERT INTO "users" (name) VALUES ("non_existing");
// user -> User{Id: 112, Name: "non_existing"}

// Found
db.Where(User{Name: "Jinzhu"}).FirstOrCreate(&user)
// user -> User{Id: 111, Name: "Jinzhu"}
```

> 获取第一个匹配的记录，或创建一个具有给定条件的新记录（仅适用于struct, map条件）

```go
// Unfound
db.FirstOrCreate(&user, User{Name: "non_existing"})
//// INSERT INTO "users" (name) VALUES ("non_existing");
//// user -> User{Id: 112, Name: "non_existing"}

// Found
db.Where(User{Name: "Jinzhu"}).FirstOrCreate(&user)
//// user -> User{Id: 111, Name: "Jinzhu"}
```

> 将其分配给记录，而不管它是否被找到，并保存回数据库。

```go
// Unfound
db.Where(User{Name: "non_existing"}).Assign(User{Age: 20}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'non_existing';
//// INSERT INTO "users" (name, age) VALUES ("non_existing", 20);
//// user -> User{Id: 112, Name: "non_existing", Age: 20}

// Found
db.Where(User{Name: "jinzhu"}).Assign(User{Age: 30}).FirstOrCreate(&user)
//// SELECT * FROM users WHERE name = 'jinzhu';
//// UPDATE users SET age=30 WHERE id = 111;
//// user -> User{Id: 111, Name: "jinzhu", Age: 30}
```

#### Scopes

> 将当前数据库连接传递到`func(*DB) *DB`，可以用于动态添加条件

```go
func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
    return db.Where("amount > ?", 1000)
}

func PaidWithCreditCard(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func PaidWithCod(db *gorm.DB) *gorm.DB {
    return db.Where("pay_mode_sign = ?", "C")
}

func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
    return func (db *gorm.DB) *gorm.DB {
        return db.Scopes(AmountGreaterThan1000).Where("status in (?)", status)
    }
}

db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)
// 查找所有信用卡订单和金额大于1000

db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
// 查找所有COD订单和金额大于1000

db.Scopes(OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// 查找所有付费，发货订单
```

### 预加载

```go
db.Preload("Orders").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4);

db.Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4) AND state NOT IN ('cancelled');

db.Where("state = ?", "active").Preload("Orders", "state NOT IN (?)", "cancelled").Find(&users)
//// SELECT * FROM users WHERE state = 'active';
//// SELECT * FROM orders WHERE user_id IN (1,2) AND state NOT IN ('cancelled');

db.Preload("Orders").Preload("Profile").Preload("Role").Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4); // has many
//// SELECT * FROM profiles WHERE user_id IN (1,2,3,4); // has one
//// SELECT * FROM roles WHERE id IN (4,5,6); // belongs to
```

#### 自定义预加载SQL

> 您可以通过传递`func(db *gorm.DB) *gorm.DB`（与Scopes的使用方法相同）来自定义预加载SQL，例如：

```go
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("orders.amount DESC")
}).Find(&users)
//// SELECT * FROM users;
//// SELECT * FROM orders WHERE user_id IN (1,2,3,4) order by orders.amount DESC;
```

#### 嵌套预加载

```go
db.Preload("Orders.OrderItems").Find(&users)
db.Preload("Orders", "state = ?", "paid").Preload("Orders.OrderItems").Find(&users)
```

### 更新

> Save将包括执行更新SQL时的所有字段，即使它没有更改

```go
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)
//// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

#### 更新更改字段

> 如果只想更新更改的字段，可以使用`Update`, `Updates`

```go
// 更新单个属性（如果更改）
db.Model(&user).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// 使用组合条件更新单个属性
db.Model(&user).Where("active = ?", true).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;

// 使用`map`更新多个属性，只会更新这些更改的字段
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// 使用`struct`更新多个属性，只会更新这些更改的和非空白字段
db.Model(&user).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// 警告:当使用struct更新时，FORM将仅更新具有非空值的字段
// 对于下面的更新，什么都不会更新为""，0，false是其类型的空白值
db.Model(&user).Updates(User{Name: "", Age: 0, Actived: false})
```

#### 更新选择的字段

> 如果您只想在更新时更新或忽略某些字段，可以使用`Select`, `Omit`

```go
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

#### 更新更改字段但不进行Callbacks

> 以上更新操作将执行模型的`BeforeUpdate`, `AfterUpdate`方法，更新其`UpdatedAt`时间戳，在更新时保存它的`Associations`，如果不想调用它们，可以使用`UpdateColumn`, `UpdateColumns`

```go
// 更新单个属性，类似于`Update`
db.Model(&user).UpdateColumn("name", "hello")
//// UPDATE users SET name='hello' WHERE id = 111;

// 更新多个属性，与“更新”类似
db.Model(&user).UpdateColumns(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18 WHERE id = 111;
```

#### Batch Updates 批量更新

> `Callbacks`在批量更新时不会运行

```go
db.Table("users").Where("id IN (?)", []int{10, 11}).Updates(map[string]interface{}{"name": "hello", "age": 18})
//// UPDATE users SET name='hello', age=18 WHERE id IN (10, 11);

// 使用struct更新仅适用于非零值，或使用map[string]interface{}
db.Model(User{}).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18;

// 使用`RowsAffected`获取更新记录计数
db.Model(User{}).Updates(User{Name: "hello", Age: 18}).RowsAffected
```

#### 使用SQL表达式更新

```go
DB.Model(&product).Update("price", gorm.Expr("price * ? + ?", 2, 100))
//// UPDATE "products" SET "price" = price * '2' + '100', "updated_at" = '2013-11-17 21:34:10' WHERE "id" = '2';

DB.Model(&product).Updates(map[string]interface{}{"price": gorm.Expr("price * ? + ?", 2, 100)})
//// UPDATE "products" SET "price" = price * '2' + '100', "updated_at" = '2013-11-17 21:34:10' WHERE "id" = '2';

DB.Model(&product).UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
//// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = '2';

DB.Model(&product).Where("quantity > 1").UpdateColumn("quantity", gorm.Expr("quantity - ?", 1))
//// UPDATE "products" SET "quantity" = quantity - 1 WHERE "id" = '2' AND quantity > 1;
```

#### 在Callbacks中更改更新值

> 如果要使用`BeforeUpdate`, `BeforeSave`更改回调中的更新值，可以使用`scope.SetColumn`，例如

```go
func (user *User) BeforeSave(scope *gorm.Scope) (err error) {
  if pw, err := bcrypt.GenerateFromPassword(user.Password, 0); err == nil {
    scope.SetColumn("EncryptedPassword", pw)
  }
}
```

#### 额外更新选项

```go
// 为Update语句添加额外的SQL选项
db.Model(&user).Set("gorm:update_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Update("name, "hello")
//// UPDATE users SET name='hello', updated_at = '2013-11-17 21:34:10' WHERE id=111 OPTION (OPTIMIZE FOR UNKNOWN);
```

### 删除/软删除

> 删除记录时，需要确保其主要字段具有值，GORM将使用主键删除记录，如果主要字段为空，GORM将删除模型的所有记录

```go
// 删除存在的记录
db.Delete(&email)
//// DELETE from emails where id=10;

// 为Delete语句添加额外的SQL选项
db.Set("gorm:delete_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Delete(&email)
//// DELETE from emails where id=10 OPTION (OPTIMIZE FOR UNKNOWN);
```

#### 批量删除

```go
// 删除所有匹配记录
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
//// DELETE from emails where email LIKE "%jinhu%";

db.Delete(Email{}, "email LIKE ?", "%jinzhu%")
//// DELETE from emails where email LIKE "%jinhu%";
```

#### 软删除

> 如果模型有`DeletedAt`字段，它将自动获得软删除功能！ 那么在调用`Delete`时不会从数据库中永久删除，而是只将字段`DeletedAt`的值设置为当前时间。

```go
db.Delete(&user)
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// 批量删除
db.Where("age = ?", 20).Delete(&User{})
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// 软删除的记录将在查询时被忽略
db.Where("age = 20").Find(&user)
//// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;

// 使用Unscoped查找软删除的记录
db.Unscoped().Where("age = 20").Find(&users)
//// SELECT * FROM users WHERE age = 20;

// 使用Unscoped永久删除记录
db.Unscoped().Delete(&order)
//// DELETE FROM orders WHERE id=10;
```

### 关联

> 默认情况下，当创建/更新记录时，GORM将保存其关联，如果关联具有主键，GORM将调用Update来保存它，否则将被创建。

```go
user := User{
    Name:            "jinzhu",
    BillingAddress:  Address{Address1: "Billing Address - Address 1"},
    ShippingAddress: Address{Address1: "Shipping Address - Address 1"},
    Emails:          []Email{
                                        {Email: "jinzhu@example.com"},
                                        {Email: "jinzhu-2@example@example.com"},
                   },
    Languages:       []Language{
                     {Name: "ZH"},
                     {Name: "EN"},
                   },
}

db.Create(&user)
//// BEGIN TRANSACTION;
//// INSERT INTO "addresses" (address1) VALUES ("Billing Address - Address 1");
//// INSERT INTO "addresses" (address1) VALUES ("Shipping Address - Address 1");
//// INSERT INTO "users" (name,billing_address_id,shipping_address_id) VALUES ("jinzhu", 1, 2);
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu@example.com");
//// INSERT INTO "emails" (user_id,email) VALUES (111, "jinzhu-2@example.com");
//// INSERT INTO "languages" ("name") VALUES ('ZH');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 1);
//// INSERT INTO "languages" ("name") VALUES ('EN');
//// INSERT INTO user_languages ("user_id","language_id") VALUES (111, 2);
//// COMMIT;

db.Save(&user)
```

#### 创建/更新时跳过保存关联

> 默认情况下保存记录时，GORM也会保存它的关联，你可以通过设置`gorm:save_associations`为`false`跳过它。

```go
db.Set("gorm:save_associations", false).Create(&user)

db.Set("gorm:save_associations", false).Save(&user)
```

####  tag设置跳过保存关联

> 您可以使用tag来配置您的struct，以便在创建/更新时不会保存关联

```go
type User struct {
  gorm.Model
  Name      string
  CompanyID uint
  Company   Company `gorm:"save_associations:false"`
}

type Company struct {
  gorm.Model
  Name string
}
```

## 回调

> 您可以将回调方法定义为模型结构的指针，在创建，更新，查询，删除时将被调用，如果任何回调返回错误，gorm将停止未来操作并回滚所有更改。

### 创建对象

> 创建过程中可用的回调

```go
// begin transaction 开始事物
BeforeSave
BeforeCreate
// save before associations 保存前关联
// update timestamp `CreatedAt`, `UpdatedAt` 更新`CreatedAt`, `UpdatedAt`时间戳
// save self 保存自己
// reload fields that have default value and its value is blank 重新加载具有默认值且其值为空的字段
// save after associations 保存后关联
AfterCreate
AfterSave
// commit or rollback transaction 提交或回滚事务
```

### 更新对象

> 更新过程中可用的回调

```go
// begin transaction 开始事物
BeforeSave
BeforeUpdate
// save before associations 保存前关联
// update timestamp `UpdatedAt` 更新`UpdatedAt`时间戳
// save self 保存自己
// save after associations 保存后关联
AfterUpdate
AfterSave
// commit or rollback transaction 提交或回滚事务
```

### 删除对象

> 删除过程中可用的回调

```go
// begin transaction 开始事物
BeforeDelete
// delete self 删除自己
AfterDelete
// commit or rollback transaction 提交或回滚事务
```

### 查询对象

> 查询过程中可用的回调

```go
// load data from database 从数据库加载数据
// Preloading (edger loading) 预加载（加载）
AfterFind
```

###  回调示例

```go
func (u *User) BeforeUpdate() (err error) {
    if u.readonly() {
        err = errors.New("read only user")
    }
    return
}

// 如果用户ID大于1000，则回滚插入
func (u *User) AfterCreate() (err error) {
    if (u.Id > 1000) {
        err = errors.New("user id is already greater than 1000")
    }
    return
}
```

> gorm中的保存/删除操作正在事务中运行，因此在该事务中所做的更改不可见，除非提交。 如果要在回调中使用这些更改，则需要在同一事务中运行SQL。 所以你需要传递当前事务到回调，像这样：

```go
func (u *User) AfterCreate(tx *gorm.DB) (err error) {
    tx.Model(u).Update("role", "admin")
    return
}
func (u *User) AfterCreate(scope *gorm.Scope) (err error) {
  scope.DB().Model(u).Update("role", "admin")
    return
}
```

###  高级用法

### 错误处理

> 执行任何操作后，如果发生任何错误，GORM将其设置为`*DB`的`Error`字段

```go
if err := db.Where("name = ?", "jinzhu").First(&user).Error; err != nil {
    // 错误处理...
}

// 如果有多个错误发生，用`GetErrors`获取所有的错误，它返回`[]error`
db.First(&user).Limit(10).Find(&users).GetErrors()

// 检查是否返回RecordNotFound错误
db.Where("name = ?", "hello world").First(&user).RecordNotFound()

if db.Model(&user).Related(&credit_card).RecordNotFound() {
    // 没有信用卡被发现处理...
}
```

### 事务

> 要在事务中执行一组操作，一般流程如下。

```go
// 开始事务
tx := db.Begin()

// 在事务中做一些数据库操作（从这一点使用'tx'，而不是'db'）
tx.Create(...)

// ...

// 发生错误时回滚事务
tx.Rollback()

// 或提交事务
tx.Commit()
```

#### 一个具体的例子

```go
func CreateAnimals(db *gorm.DB) err {
  tx := db.Begin()
  // 注意，一旦你在一个事务中，使用tx作为数据库句柄

  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
     tx.Rollback()
     return err
  }

  tx.Commit()
  return nil
}
```

### SQL构建

#### 执行原生SQL

```go
db.Exec("DROP TABLE users;")
db.Exec("UPDATE orders SET shipped_at=? WHERE id IN (?)", time.Now, []int64{11,22,33})

// Scan
type Result struct {
    Name string
    Age  int
}

var result Result
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)
```

####  sql.Row & sql.Rows

> 获取查询结果为`*sql.Row`或`*sql.Rows`

```go
row := db.Table("users").Where("name = ?", "jinzhu").Select("name, age").Row() // (*sql.Row)
row.Scan(&name, &age)

rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}

// Raw SQL
rows, err := db.Raw("select name, age, email from users where name = ?", "jinzhu").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}
```

#### 迭代中使用sql.Rows的Scan

```go
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()

for rows.Next() {
  var user User
  db.ScanRows(rows, &user)
  // do something
}
```

#### 通用数据库接口sql.DB

> 从`*gorm.DB`连接获取通用数据库接口*sql.DB

```go
// 获取通用数据库对象`*sql.DB`以使用其函数
db.DB()

// Ping
db.DB().Ping()
```

#### 连接池

```go
db.DB().SetMaxIdleConns(10)
db.DB().SetMaxOpenConns(100)
```

### 复合主键

> 将多个字段设置为主键以启用复合主键

```go
type Product struct {
    ID           string `gorm:"primary_key"`
    LanguageCode string `gorm:"primary_key"`
}
```

### 日志

> Gorm有内置的日志记录器支持，默认情况下，它会打印发生的错误

```go
// 启用Logger，显示详细日志
db.LogMode(true)

// 禁用日志记录器，不显示任何日志
db.LogMode(false)

// 调试单个操作，显示此操作的详细日志
db.Debug().Where("name = ?", "jinzhu").First(&User{})
```

#### 自定义日志

> 参考GORM的默认记录器如何自定义它https://github.com/jinzhu/gorm/blob/master/logger.go

```go
db.SetLogger(gorm.Logger{revel.TRACE})
db.SetLogger(log.New(os.Stdout, "\r\n", 0))
```