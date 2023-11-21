---
title: gorm库使用异常总结
date: 2023-06-13 11:58:49
categories: [go]
tags:
- go
- gorm
---

# gorm 笔记

## gorm.Model
```go
// gorm.Model 的定义
type Model struct {
  ID        uint           `gorm:"primaryKey"`
  DeletedAt gorm.DeletedAt `gorm:"index"`
}
```
可以将它嵌入到其他结构体中



## 主键
默认使用 ID 作为主键

```go
type User struct {
  ID   string // 默认情况下，名为 `ID` 的字段会作为表的主键
  Name string
}
```

也可以通过标签 `primaryKey` 将其它字段设为主键
```go
// 将 `UUID` 设为主键
type Animal struct {
  ID     int64
  UUID   string `gorm:"primaryKey"`
  Name   string
}
```

### 复合主键
```go
type Product struct {
  ID           string `gorm:"primaryKey"`
  LanguageCode string `gorm:"primaryKey"`
  CategoryID   uint64 `gorm:"primaryKey;autoIncrement:false"` // 禁用复合主键
  Code         string
}
```
> 用 `autoIncrement` 禁用 


## 表名
### 复数表名

在定义模型结构体时,如果不指定表名,会默认使用结构体名称的复数作为表名， 比如 对于结构体 `User`，根据约定，其表名为 `users`

可以实现 `Tabler` 接口来更改默认表名，例如：
```go
type Tabler interface {
    TableName() string
}

// TableName 会将 User 的表名重写为 `profiles`
func (User) TableName() string {
  return "profiles"
}
```

### 动态表名
由于 `TableName` 不支持动态变化，它会被缓存下来以便后续使用。 可以使用 `Scopes` 实现动态表名
```go
func UserTable(user User) func (tx *gorm.DB) *gorm.DB {
  return func (tx *gorm.DB) *gorm.DB {
    if user.Admin {
      return tx.Table("admin_users")
    }

    return tx.Table("users")
  }
}

db.Scopes(UserTable(user)).Create(&user)
```

### 临时指定表名
可以使用 Table 方法临时指定表名，例如

```go
// 根据 User 的字段创建 `deleted_users` 表
db.Table("deleted_users").AutoMigrate(&User{})

// 从另一张表查询数据
var deletedUsers []User
db.Table("deleted_users").Find(&deletedUsers)
// SELECT * FROM deleted_users;
```


## 列名

### 默认字段名
列名使用的是 `struct` 字段名
```go
type User struct {
  ID        uint      // 列名是 `id`
  Birthday  time.Time // 列名是 `birthday`
  CreatedAt time.Time // 列名是 `created_at`
}
```

### 标签重命名
使用 `column` 标签覆盖列名
```go
type Animal struct {
  AnimalID int64     `gorm:"column:beast_id"`         // 将列名设为 `beast_id`
}
```

## 时间

### CreatedAt
对于有 `CreatedAt` 字段的模型，创建记录时，如果该字段值为零值，则将该字段的值设为当前时间
```go
// 将 `CreatedAt` 设为当前时间
db.Create(&user) 

// user2 的 `CreatedAt` 不会被修改
user2 := User{Name: "jinzhu", CreatedAt: time.Now()}
db.Create(&user2) 

// 想要修改该值，您可以使用 `Update`
db.Model(&user).Update("CreatedAt", time.Now())

// 可以通过将 autoCreateTime 标签置为 false 来禁用时间戳追踪
type User struct {
  CreatedAt time.Time `gorm:"autoCreateTime:false"`
}
```

### UpdatedAt
对于有 `UpdatedAt` 字段的模型，更新记录时，将该字段的值设为当前时间。创建记录时，如果该字段值为零值，则将该字段的值设为当前时间
```go
db.Save(&user) // 将 `UpdatedAt` 设为当前时间

db.Model(&user).Update("name", "jinzhu") // 会将 `UpdatedAt` 设为当前时间

db.Model(&user).UpdateColumn("name", "jinzhu") // `UpdatedAt` 不会被修改

user2 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Create(&user2) // 创建记录时，user2 的 `UpdatedAt` 不会被修改

user3 := User{Name: "jinzhu", UpdatedAt: time.Now()}
db.Save(&user3) // 更新世，user3 的 `UpdatedAt` 会修改为当前时间

// 通过将 autoUpdateTime 标签置为 false 来禁用时间戳追踪
type User struct {
  UpdatedAt time.Time `gorm:"autoUpdateTime:false"`
}
```


### DeletedAt
当模型包含了 `gorm.DeletedAt` 字段（该字段也被包含在gorm.Model中），那么该模型将会自动获得**软删除**的能力,
当调用Delete时，GORM并不会从数据库中删除该记录，而是将该记录的 `DeleteAt` 设置为当前时间，而后的一般查询方法将无法查找到此条记录。

```go
db.Delete(&user) // 将 `DeletedAt` 设为当前时间
db.Unscoped().Delete(&user) // 真正从数据库中删除该记录
```

#### 查找被软删除的记录
可以使用 `Unscoped` 来查询到被软删除的记录
```go
// SELECT * FROM users WHERE age = 20;
db.Unscoped().Where("age = 20").Find(&users)
```

#### 永久删除
可以使用 `Unscoped` 来永久删除匹配的记录
```go
// DELETE FROM orders WHERE id=10;
db.Unscoped().Delete(&order)
```

> 默认情况下，gorm.Model使用 `*time.Time` 作为 `DeletedAt` 的字段类型

#### 几种使用方法
```go
func SoftDeleteById(ctx context.Context, tx *gorm.DB, tableName string, id int64) error {
	model := tx.db.Model(gorm.Model{})
	model.Table(tableName)

	// 方式1
	tx.Table(tableName).WithContext(ctx).Where("eid = ?", eid).Delete(&biz.IntranetDomain{})

	// 方式2
	tx.Table(tableName).WithContext(ctx).Where("eid = ?", eid).Delete()
	
	// 方式3
	tx.Table(tableName).WithContext(ctx).Where("eid = ?", eid).Delete(nil)
	
	// 方式4
	model.Table(tableName).WithContext(ctx).Where("eid = ?", eid).Delete(nil)
	
	// 方式5
	type s_feedback struct{}
	table := s_feedback{}
	tx.Table(tableName).WithContext(ctx).Where("eid = ?", eid).Delete(&table)	
}
```




### 时间单位
```go
type User struct {
    CreatedAt time.Time // 在创建时，如果该字段值为零值，则使用当前时间填充
    UpdatedAt int       // 在创建时该字段值为零值或者在更新时，使用当前时间戳秒数填充
    Updated   int64 `gorm:"autoUpdateTime:nano"` // 使用时间戳纳秒数填充更新时间
    Updated   int64 `gorm:"autoUpdateTime:milli"` // 使用时间戳毫秒数填充更新时间
}
```

> 要保存 UNIX（毫/纳）秒时间戳，而不是 time，只需将 time.Time 修改为 int


## 权限控制
用标签控制字段级别的权限
```go
type User struct {
  Name string `gorm:"<-:create"`            // 允许读和创建
  Name string `gorm:"<-:update"`            // 允许读和更新
  Name string `gorm:"<-"`                   // 允许读和写（创建和更新）
  Name string `gorm:"<-:false"`             // 允许读，禁止写
  Name string `gorm:"->"`                   // 只读（除非有自定义配置，否则禁止写）
  Name string `gorm:"->;<-:create"`         // 允许读和写
  Name string `gorm:"->:false;<-:create"`   // 仅创建（禁止从 db 读）
  Name string `gorm:"-"`                    // 通过 struct 读写会忽略该字段
  Name string `gorm:"-:all"`                // 通过 struct 读写、迁移会忽略该字段
  Name string `gorm:"-:migration"`          // 通过 struct 迁移会忽略该字段
}
```


## 结构体
### 匿名字段
```go
type User struct {
  gorm.Model
  Name string
}
// 等效于
type User struct {
  ID        uint           `gorm:"primaryKey"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt gorm.DeletedAt `gorm:"index"`

  Name string
}
```

### 嵌套结构体
通过标签 `embedded` 将其嵌入
```go
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

### 添加字段名前缀 
使用标签 `embeddedPrefix` 为字段名添加前缀
```go
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

### 默认值
通过 tag 的字段定义默认值 `default`，使用 `default:(-)` 跳过默认值
```go
type User struct {
  ID   int64
  Name string `gorm:"default:galeone"`
  Age  int64  `gorm:"default:18"`
  FullName  string `gorm:"default:(-)"`
}
```

> 当写入零值时会使用默认值



## 配置连接池
```go
// 获取通用数据库对象 sql.DB ，然后使用其提供的功能
sqlDB, err := db.DB()

// SetMaxIdleConns 用于设置连接池中空闲连接的最大数量。
sqlDB.SetMaxIdleConns(10)

// SetMaxOpenConns 设置打开数据库连接的最大数量。
sqlDB.SetMaxOpenConns(100)

// SetConnMaxLifetime 设置了连接可复用的最大时间。
sqlDB.SetConnMaxLifetime(time.Hour)
```

## 插入注意点
### 指定插入字段
```go
db.Select("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`name`,`age`,`created_at`) VALUES ("jinzhu", 18, "2020-07-04 11:05:21.775")
```
### 插入时指定要省略的字段
```go
db.Omit("Name", "Age", "CreatedAt").Create(&user)
// INSERT INTO `users` (`birthday`,`updated_at`) VALUES ("2020-01-01 00:00:00.000", "2020-07-04 11:05:21.775")
```

### 块写入
#### 切片写入
```go
var users = []User{{Name: "jinzhu1"}, {Name: "jinzhu2"}, {Name: "jinzhu3"}}
db.Create(&users)
```
#### 批量写入


```go
// 初始化时设置 batch size 1000
db := db.Session(&gorm.Session{CreateBatchSize: 1000})

var users = []User{{Name: "jinzhu_1"}, ...., {Name: "jinzhu_10000"}}
db.Create(&users)
// 在使用中覆盖 batch size 100
db.CreateInBatches(users, 100)
```
#### map写入
```go
db.Model(&User{}).Create(map[string]interface{}{
  "Name": "jinzhu", "Age": 18,
})

// batch insert from `[]map[string]interface{}{}`
db.Model(&User{}).Create([]map[string]interface{}{
  {"Name": "jinzhu_1", "Age": 18},
  {"Name": "jinzhu_2", "Age": 20},
})
```

> 从 map 创建时，不会调用hooks，不会保存关联，也不会回填主键

## hooks
允许为 `BeforeSave、BeforeCreate、AfterSave` 和 `AfterCreate` 实现用户自定义钩子。创建时将调用这些钩子方法
```go
if err = db.Callback().Create().Before("gorm:before_create").Register("log:sql_before_create", func(db *gorm.DB) {
    // 在执行SQL语句之前记录日志
    fmt.Println("执行的SQL语句：", db.Statement.SQL.String())
}); err != nil {
    fmt.Errorf("gorm:before_create: %s", err)
}

func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    u.UUID = uuid.New()

    if u.Role == "admin" {
        return errors.New("invalid role")
    }
    return
}
```

如果想跳过钩子方法，可以使用 `SkipHooks` 会话模式：
```go
DB.Session(&gorm.Session{SkipHooks: true}).Create(&users)
DB.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
```


## 查询
### 检索单个对象
提供了 `First`、`Take`、`Last` 方法，以便从数据库中检索单个对象。当查询数据库时它添加了 LIMIT 1 条件，且没有找到记录时，它会返回 `ErrRecordNotFound` 错误

```go
// 获取第一条记录（主键升序）
db.First(&user) // SELECT * FROM users ORDER BY id LIMIT 1;

// 获取一条记录，没有指定排序字段
db.Take(&user)  // SELECT * FROM users LIMIT 1;

// 获取最后一条记录（主键降序）
db.Last(&user)  // SELECT * FROM users ORDER BY id DESC LIMIT 1;
```

 
`First` 和 `Last` 方法将分别查找按主键排序的第一条和最后一条记录。只有将目标结构的**指针**作为参数传递给这两个方法或使用 `db.Model()` 指定模型时，这两个方法才会起作用。此外，如果没有为相关模型定义主键，那么模型将按第一个字段排序

```go
var user User

db.First(&user) // SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1

result := map[string]interface{}{}
db.Model(&User{}).First(&result)  // SELECT * FROM `users` ORDER BY `users`.`id` LIMIT 1

// doesn't work
result := map[string]interface{}{}
db.Table("users").First(&result)

// works with Take
result := map[string]interface{}{}
db.Table("users").Take(&result)

// no primary key defined, results will be ordered by first field (i.e., `Code`)
type Language struct {
  Code string
  Name string
}
db.First(&Language{}) // SELECT * FROM `languages` ORDER BY `languages`.`code` LIMIT 1
```

### 检索全部记录
```go
result := db.Find(&users) // SELECT * FROM users;
```


### 条件筛选
```go
// Struct
 db.Where(&User{Name: "jinzhu" , Age: 20 }).First(&user) // SELECT * FROM users WHERE name = "jinzhu" AND Age = 20 ORDER BY id LIMIT 1;

// 使用 struct 进行查询时，GORM 只会查询非零字段，如果字段值为0、''或false其他零值，则不会用于构建查询条件
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)  // SELECT * FROM users WHERE name = "jinzhu";

// map
db.Where( map [ string ] interface {}{ "name" : "jinzhu" , "age" : 20 }).Find(&users) // SELECT * FROM users WHERE name = "jinzhu" AND Age = 20；
// 要在查询条件中包含零值，可以使用映射
db.Where( map [ string ] interface {}{ "Name" : "jinzhu" , "Age" : 0 }).Find(&users)  // SELECT * FROM users WHERE name = "jinzhu" AND Age = 0;

// slice 
db.Where([] int64 { 20 , 21 , 22 }).Find(&users) // SELECT * FROM users WHERE id IN (20, 21, 22);
```

### 选择特定字段
```go
type User struct {
  ID     uint
  Name   string
  Age    int
  // 假设后面还有几百个字段...
}

type APIUser struct {
  ID   uint
  Name string
}

// 查询时会自动选择 `id`, `name` 字段
db.Model(&User{}).Limit(10).Find(&APIUser{})  // SELECT `id`, `name` FROM `users` LIMIT 10
```

### 带多个列的 In 查询
```go
db.Where("(name, age, role) IN ?", [][]interface{}{{"jinzhu", 18, "admin"}, {"jinzhu2", 19, "user"}}).Find(&users)
// SELECT * FROM users WHERE (name, age, role) IN (("jinzhu", 18, "admin"), ("jinzhu 2", 19, "user"));
```

## 更新
### 更新单个列
```go
// User's ID is `111`:
db.Model(&user).Update("name", "hello") // UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;
```

### 更新多列
```go
// struct 只更新非零字段
db.Model(&user).Updates(User{Name: "hello", Age: 18, Active: false}) 
// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// map
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello', age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
```

### 批量更新
没有使用Model指定具有主键值的记录，GORM将执行批处理更新
```go
// struct
db.Model(User{}).Where("role = ?", "admin").Updates(User{Name: "hello", Age: 18})
// UPDATE users SET name='hello', age=18 WHERE role = 'admin';

// map
db.Table("users").Where("id IN ?", []int{10, 11}).Updates(map[string]interface{}{"name": "hello", "age": 18})
// UPDATE users SET name='hello', age=18 WHERE id IN (10, 11);
```


### 更新选定字段
```go
// Map
db.Model(&user).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET name='hello' WHERE id=111; (User's ID is `111`:)

db.Model(&user).Omit("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "active": false})
// UPDATE users SET age=18, active=false, updated_at='2013-11-17 21:34:10' WHERE id=111;


// struct
db.Model(&user).Select("Name", "Age").Updates(User{Name: "new_name", Age: 0})
// UPDATE users SET name='new_name', age=0 WHERE id=111;
```

### 保存所有字段
`Save` 会保存所有的字段，即使字段是零值；`Save` 是一个组合函数。如果保存值不包含主键，则执行 `Create`，否则执行 `Update`（包含所有字段）
```go
// 没有值 新建
db.Save(&User{Name: "jinzhu" , Age: 100 }) 
// INSERT INTO `users` (`name`,`age`,`birthday`,`update_at`) VALUES ("jinzhu",100," 0000-00-00 00:00:00","0000-00-00 00:00:00")

// 有值 更新
db.Save(&User{ID: 1 , Name: "jinzhu" , Age: 100 }) 
// UPDATE `users` SET `name`="jinzhu",`age`=100,`birthday`="0000-00 -00 00:00:00",`update_at`="0000-00-00 00:00:00" 其中 `id` = 1
```

> Save 不能跟 Model 同用


## 链式方法
三种类型的方法： 
- 链式方法
  链式方法是将 Clauses 修改或添加到当前 Statement 的方法，例如：`Where, Select, Omit, Joins, Scopes, Preload, Raw...`

- 终结方法
  终结（方法） 是会立即执行注册回调的方法，然后生成并执行 SQL，比如这些方法：`Create, First, Find, Take, Save, Update, Delete, Scan, Row, Rows...`

- 新建会话方法
  定义了 `Session`、`WithContext`、`Debug` 方法做为 新建会话方法; 使用 新建会话方法 来标记 `*gorm.DB` 为可共享


在 链式方法, 终结方法之后, GORM 返回一个初始化的 `*gorm.DB` 实例，实例不能安全地重复使用，并且新生成的 SQL 可能会被先前的条件污染
```go
tx := DB.Where("name = ?", "jinzhu")

tx.Where("age > ?", 10).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age > 10

tx.Where("age > ?", 20).First(&user2)
// SELECT * FROM users WHERE name = "jinzhu" AND age > 10 AND age > 20
```

使用 新建会话方法 创建一个可共享的 `*gorm.DB`, 实现重新使用初始化的 `*gorm.DB` 实例
```go
// 利用新建会话方法创建一个可共享的 `*gorm.DB`
tx := DB.Where("name = ?", "jinzhu").Session(&gorm.Session{})

tx.Where("age > ?", 10).First(&user)
// SELECT * FROM users WHERE name = "jinzhu" AND age > 10

tx.Where("age > ?", 20).First(&user2)
// SELECT * FROM users WHERE name = "jinzhu" AND age > 20
```
或者

```go
// db是一个新初始化的' *gorm.DB '，可以安全重用
db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})

db.Where("name = ?", "jinzhu").Where("age = ?", 18).Find(&users)  // SELECT * FROM users WHERE name = 'jinzhu' AND age = 18;

db.Where("name = ?", "jinzhu2").Where("age = ?", 20).Find(&users) // SELECT * FROM users WHERE name = 'jinzhu2' AND age = 20;

db.Find(&users) // SELECT * FROM users;
```


## 事务
为了确保数据一致性，GORM 会在事务里执行写入操作（创建、更新、删除）。如果没有这方面的要求，可以在初始化时禁用它，将获得性能提升

```go
// 全局禁用
db, err := gorm.Open(sqlite.Open("gorm.db"), &gorm.Config{
  SkipDefaultTransaction: true,
})

// 持续会话模式
tx := db.Session(&Session{SkipDefaultTransaction: true})
```


## 自定义数据类型
自定义的数据类型必须实现 `Scanner` 和 `Valuer` 接口，以便让 GORM 知道如何将该类型接收、保存到数据库， 接口要求如下：
```go
type Scanner interface {
	// Scan assigns a value from a database driver.
	//
	// The src value will be of one of the following types:
	//
	//    int64
	//    float64
	//    bool
	//    []byte
	//    string
	//    time.Time
	//    nil - for NULL values
	//
	// An error should be returned if the value cannot be stored
	// without loss of information.
	//
	// Reference types such as []byte are only valid until the next call to Scan
	// and should not be retained. Their underlying memory is owned by the driver.
	// If retention is necessary, copy their values before the next call to Scan.
	Scan(src any) error
}

type Valuer interface {
	// Value returns a driver Value.
	// Value must not panic.
	Value() (Value, error)
}
```

举例：

```go
type JSON json.RawMessage

// 实现 sql.Scanner 接口，Scan 将 value 扫描至 Jsonb
func (j *JSON) Scan(value interface{}) error {
  bytes, ok := value.([]byte)
  if !ok {
    return errors.New(fmt.Sprint("Failed to unmarshal JSONB value:", value))
  }

  result := json.RawMessage{}
  err := json.Unmarshal(bytes, &result)
  *j = JSON(result)
  return err
}

// 实现 driver.Valuer 接口，Value 返回 json value
func (j JSON) Value() (driver.Value, error) {
  if len(j) == 0 {
    return nil, nil
  }
  return json.RawMessage(j).MarshalJSON()
}
```

## 作用域 [Scope](https://gorm.io/zh_CN/docs/scopes.html)
复用的通用逻辑，这种共享逻辑需要定义为类型 `func(*gorm.DB) *gorm.DB`

### 查询示例：
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

// 查找所有金额大于 1000 的信用卡订单
db.Scopes(AmountGreaterThan1000, PaidWithCreditCard).Find(&orders)

// 查找所有金额大于 1000 的 COD 订单
db.Scopes(AmountGreaterThan1000, PaidWithCod).Find(&orders)
```

### 动态表
```go
func TableOfYear(user *User, year int) func(db *gorm.DB) *gorm.DB {
  return func(db *gorm.DB) *gorm.DB {
        tableName := user.TableName() + strconv.Itoa(year)
        return db.Table(tableName)
  }
}

DB.Scopes(TableOfYear(user, 2019)).Find(&users) // SELECT * FROM users_2019;

DB.Scopes(TableOfYear(user, 2020)).Find(&users) // SELECT * FROM users_2020;
```


# 异常汇总：
## 结构体标签使用错误：
### demo 代码如下：

```go 	
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/schema"
)

type PolicyInfo struct {
	Id           uint32 `gorm:"column:id;primaryKey" mapstructure:"id"`            // 规则id
	RuleName     string `gorm:"column:rule_name" mapstructure:"rule_name"`         // 规则名称
	Action       uint32 `gorm:"column:action" mapstructure:"action"`               // 处置动作类型
	RedirectIp   string `gorm:"column:redirect_ip" mapstructure:"redirect_ip"`     // 重定向ip
	RulePriority uint32 `gorm:"column:rule_priority" mapstructure:"rule_priority"` // 规则优先级
	Status       uint32 `gorm:"column:status" mapstructure:"status"`               // 规则状态

	Eid       string `gorm:"column:eid" mapstructure:"eid"`               // 企业id
	IsDeleted bool   `gorm:"column:is_deleted" mapstructure:"is_deleted"` // 是否删除

	FormJson string `gorm:"column:form_json" mapstructure:"form_json"` // 表单json

	CreateTime    string `gorm:"column:create_time" mapstructure:"create_time"`         // 创建时间
	UpdateTime    string `gorm:"column:update_time" mapstructure:"update_time"`         // 更新时间
	OpenCloseTime string `gorm:"column:open_close_time" mapstructure:"open_close_time"` // 开关时间

	Severity    []uint32    `gorm:"column:severity" json:"severity,omitempty"`
	Blacklist   []string    `gorm:"column:blacklist" json:"blacklist,omitempty"`
	DefaultType uint32      `gorm:"column:default_type" json:"default_type,omitempty"`
	DomainType  []uint32    `gorm:"column:domain_type" json:"domain_type,omitempty"`
	EffectArea  *EffectArea `json:"effect_area,omitempty"` // 生效范围
	EffectCount uint32      `json:"effect_count,omitempty"`
}

type RequestRuleList struct {
	Action       uint32      `protobuf:"varint,1,opt,name=action,proto3" json:"action,omitempty"`                                  // 处置动作类型
	DomainType   []uint32    `protobuf:"varint,2,rep,packed,name=domain_type,json=domainType,proto3" json:"domain_type,omitempty"` // 域名类型
	Blacklist    []string    `protobuf:"bytes,3,rep,name=blacklist,proto3" json:"blacklist,omitempty"`                             // 黑名单
	EffectArea   *EffectArea `protobuf:"bytes,4,opt,name=effect_area,json=effectArea,proto3" json:"effect_area,omitempty"`         // 生效范围
	RedirectIp   string      `protobuf:"bytes,5,opt,name=redirect_ip,json=redirectIp,proto3" json:"redirect_ip,omitempty"`         // 重定向ip
	RuleName     string      `protobuf:"bytes,6,opt,name=rule_name,json=ruleName,proto3" json:"rule_name,omitempty"`               // 规则名称
	RulePriority uint32      `protobuf:"varint,7,opt,name=rule_priority,json=rulePriority,proto3" json:"rule_priority,omitempty"`  // 规则优先级
	Severity     []uint32    `protobuf:"varint,8,rep,packed,name=severity,proto3" json:"severity,omitempty"`                       // 威胁等级
}

type FormJson struct {
	DefaultType    uint32   `json:"default_type,omitempty"`     // 默认类型
	DomainType     []uint32 `json:"domain_type,omitempty"`      // 域名类型
	Blacklist      []string `json:"blacklist,omitempty"`        // 黑名单
	Severity       []uint32 `json:"severity,omitempty"`         // 威胁等级
	DataSourceUids []string `json:"data_source_uids,omitempty"` // 情报
	GroupCheckAll  uint32   `json:"group_check_all,omitempty"`  // 安全域
	Version        string   `json:"version,omitempty"`
}

type EffectArea struct {
	GroupCheckAll  uint32   `json:"group_check_all"`
	DataSourceUids []string `json:"data_source_uids"`
}

// 切片去重
func removeDuplicates(s []string) []string {
	seen := make(map[string]bool)
	result := []string{}
	for _, v := range s {
		if _, ok := seen[v]; !ok {
			seen[v] = true
			result = append(result, v)
		}
	}
	return result
}

const (
	postgre_data_source = "host=10.229.3.59 user=light password=HAbjC6RByLUP5qpSUhtAFWtPfmw0BPvzqJNCYgvu65bJVSVomX2gJiRlvxspTQpZ dbname=light search_path=light port=31387 sslmode=disable TimeZone=Asia/Shanghai"
	MaxIdeConn          = 10
	MaxOpenConn         = 100
	ConnMaxLife         = 60
)

func GetDB() (*gorm.DB, error) {
	db, err := gorm.Open(postgres.New(postgres.Config{
		DSN:                  postgre_data_source,
		PreferSimpleProtocol: true, // disables implicit prepared statement usage
	}), &gorm.Config{
		PrepareStmt: true,
		NamingStrategy: schema.NamingStrategy{
			SingularTable: true,
		},
	})

	if err != nil {
		return nil, fmt.Errorf("Unable to connection to database! dsn:%v err:%s", postgre_data_source, err)
	}

	sqlDb, err := db.DB()
	if err != nil {
		fmt.Errorf("get db is err:%s", err)
		return nil, err
	}

	// defer sqlDb.Close()

	// 构建连接池
	sqlDb.SetMaxIdleConns(int(MaxIdeConn))
	sqlDb.SetMaxOpenConns(int(MaxOpenConn))
	sqlDb.SetConnMaxLifetime(ConnMaxLife)

	if err = sqlDb.Ping(); err != nil {
		fmt.Errorf("models.Ping err: %v", err)
		return nil, err
	}
	return db, nil
}

func main() {

	ctx, cancle := context.WithCancel(context.Background())
	defer cancle()

	eid := "C6ADCD26-E63B-11EC-BC53-B62BBA637A35"

	effectArea := &EffectArea{
		GroupCheckAll:  0,
		DataSourceUids: []string{"sdsd", "test0914"},
	}

	in := RequestRuleList{
		RuleName:     "test",
		Action:       222,
		Blacklist:    []string{"afdasfsa", "test0914"},
		DomainType:   []uint32{10100, 10017, 10021, 10025, 10035, 10037, 10041, 10042, 10043, 10011, 10012, 10013, 10014, 10015},
		EffectArea:   effectArea,
		RedirectIp:   "1.1.1.1",
		RulePriority: 126,
		Severity:     []uint32{1, 3, 5},
	}
	// 构造数据
	black_list := removeDuplicates(in.Blacklist)

	form_json := FormJson{
		Blacklist:      black_list,
		DomainType:     in.DomainType,
		Severity:       in.Severity,
		GroupCheckAll:  in.EffectArea.GroupCheckAll,
		DataSourceUids: in.EffectArea.DataSourceUids,
		Version:        "1.0.0",
	}

	jsonBytes, err := json.Marshal(form_json)
	if err != nil {
		fmt.Println(err)
		return
	}
	jsonStr := string(jsonBytes)
	fmt.Println(jsonStr)

	now := time.Now().Format("2006-01-02 15:04:05")

	rule_info := PolicyInfo{
		Action:       in.Action,
		Status:       1,
		RuleName:     in.RuleName,
		RulePriority: in.RulePriority,

		IsDeleted:     false,
		RedirectIp:    in.RedirectIp,
		CreateTime:    now,
		UpdateTime:    now,
		OpenCloseTime: now,
		Eid:           eid,

		FormJson: jsonStr,
	}

	db, err := GetDB()
	if err != nil {
		fmt.Errorf("get db is err:%s", err)
		return
	}

	result := db.Table("s_rule").WithContext(ctx).Create(&rule_info)
	if result.Error != nil && result.Error != gorm.ErrRecordNotFound {
		fmt.Println(result.Error)
		return
	}
    
	fmt.Printf("result.Id: %d\tinsert rule row: %d\n", rule_info.Id, result.RowsAffected)
}
```

运行异常：
```bash
# [14.697ms] [rows:0] INSERT INTO "s_rule" ("rule_name","action","redirect_ip","rule_priority","status","eid","is_deleted","form_json","create_time","update_time","open_close_time","severity","blacklist","default_type","domain_type","effect_count") VALUES ('test',222,'1.1.1.1',126,1,'C6ADCD26-E63B-11EC-BC53-B62BBA637A35',false,'{"domain_type":[10100,10017,10021,10025,10035,10037,10041,10042,10043,10011,10012,10013,10014,10015],"blacklist":["afdasfsa","test0914"],"severity":[1,3,5],"data_source_uids":["sdsd","test0914"],"version":"1.0.0"}','2023-06-13 12:10:14','2023-06-13 12:10:14','2023-06-13 12:10:14',(NULL),(NULL),0,(NULL),0) RETURNING "id"
ERROR: column "severity" of relation "s_rule" does not exist (SQLSTATE 42703)
```

是因为将 FormJson 内容进行了展开，并追加到了 sql 语句中，导致了异常。

### 修复方法1:
问题出现的原因是 gorm 标签的使用不正确。在结构体 PolicyInfo 中，FormJson 字段的 gorm 标签声明在了错误的位置，导致其后面的字段都被认为是 FormJson 的子字段。

要解决这个问题，需要将 gorm 标签放置在结构体字段定义的后面，并且确保标签的格式正确。以下是修复后的代码示例：
```go
type PolicyInfo struct {
	Id           uint32   `gorm:"column:id;primaryKey" mapstructure:"id"` // 规则id
	RuleName     string   `gorm:"column:rule_name" mapstructure:"rule_name"` // 规则名称
	Action       uint32   `gorm:"column:action" mapstructure:"action"` // 处置动作类型
	RedirectIp   string   `gorm:"column:redirect_ip" mapstructure:"redirect_ip"` // 重定向ip
	RulePriority uint32   `gorm:"column:rule_priority" mapstructure:"rule_priority"` // 规则优先级
	Status       uint32   `gorm:"column:status" mapstructure:"status"` // 规则状态

	Eid          string   `gorm:"column:eid" mapstructure:"eid"` // 企业id
	IsDeleted    bool     `gorm:"column:is_deleted" mapstructure:"is_deleted"` // 是否删除
	FormJson     string   `gorm:"column:form_json" mapstructure:"form_json"` // 表单json
	CreateTime   string   `gorm:"column:create_time" mapstructure:"create_time"` // 创建时间
	UpdateTime   string   `gorm:"column:update_time" mapstructure:"update_time"` // 更新时间
	OpenCloseTime string   `gorm:"column:open_close_time" mapstructure:"open_close_time"` // 开关时间
	Severity     []uint32 `gorm:"-" json:"severity,omitempty"` // 临时排除
	Blacklist    []string `gorm:"-" json:"blacklist,omitempty"` // 临时排除
	DefaultType  uint32   `gorm:"-" json:"default_type,omitempty"` // 临时排除
	DomainType   []uint32 `gorm:"-" json:"domain_type,omitempty"` // 临时排除
	EffectArea   *EffectArea `json:"effect_area,omitempty"` // 生效范围
	EffectCount  uint32   `gorm:"-" json:"effect_count,omitempty"` // 临时排除
}
```



### 修复方法2:
在创建记录时，GORM 会自动将结构体中的所有字段都写入 SQL 中。要达到你想要的效果（只插入选定的字段），你可以使用 Select 方法指定要插入的字段
通过使用 Select 方法，在 SQL 中插入的字段将只包括所指定的字段，而不会在 open_close_time 后面追加其他字段。
```go
	result := p.db.Table(biz.PolicyTableName).
		WithContext(ctx).
		Select("RuleName", "Action", "RedirectIp", "RulePriority", "Status", "Eid", "IsDeleted", "FormJson", "CreateTime", "UpdateTime", "OpenCloseTime").
		Create(&rule_info)
```

### 使用gen 生成代码框架，然后操作中使用生成的数据结构


## 更新零值问题：

```go
type SAssetIP struct {
	ID          int64          `gorm:"column:id;primaryKey;autoIncrement:true" json:"id"`
	Eid         string         `gorm:"column:eid;not null;comment:企业sn" json:"eid"`                           // 企业sn
	ProxyID     string         `gorm:"column:proxy_id;not null;comment:proxy sn" json:"proxy_id"`             // proxy sn
	AssetIP     string         `gorm:"column:asset_ip;not null;comment:资产ip，即表示ip地址(ip或ip段)" json:"asset_ip"` // 资产ip，即表示ip地址(ip或ip段)
	AssetType   string         `gorm:"column:asset_type;not null;comment:资产类别" json:"asset_type"`             // 资产类别
	Place       string         `gorm:"column:place;comment:位置" json:"place"`                                  // 位置
	DiscoverWay int16          `gorm:"column:discover_way;not null;comment:来源，1手动添加，2发现" json:"discover_way"` // 来源，1手动添加，2发现
	Contacts    string         `gorm:"column:contacts;comment:联系人" json:"contacts"`                           // 联系人
	Phone       string         `gorm:"column:phone;comment:联系方式" json:"phone"`                                // 联系方式
	Status      int16          `gorm:"column:status;not null;comment:当前状态 1 未接入 2 接入成功" json:"status"`        // 当前状态 1 未接入 2 接入成功
	IsMajor     bool           `gorm:"column:is_major;not null;comment:是否重点资产" json:"is_major"`               // 是否重点资产
	CreatedAt   time.Time      `gorm:"column:created_at;not null;comment:创建时间" json:"created_at"`             // 创建时间
	UpdatedAt   time.Time      `gorm:"column:updated_at;comment:修改时间" json:"updated_at"`                      // 修改时间
	AssetName   string         `gorm:"column:asset_name;not null;comment:资产名称" json:"asset_name"`             // 资产名称
	DeletedAt   gorm.DeletedAt `gorm:"column:deleted_at" json:"deleted_at"`
}


func UpdateAssetInfoById(ctx context.Context, info *models.SAssetIP) error {
	if err := ds.db.Table(models.TableNameSAssetIP).WithContext(ctx).Where("id = ?", info.ID).Updates(info); err.Error != nil {
		return err.Error
	}
	return nil
}

info := &models.SAssetIP{
	ID: 19712170,
	Eid: '8211419A-5BDD-11EC-A457-F60242D1C7A8',
	ProxyId:'360bjyt',
	IsMajor:false,
	AssetIp:'10.209.2.153',
	AssetType: 'sdf1fa',
	Contacts:'xxxx',
	UpdatedAt:'2023-08-31 17:43:40.442',
	AssetName:'11sdfsf'

}
```

数据值为 info, 但是更新sql 缺少了 IsMajor 维度，执行结果如下：
```sql
  UPDATE "s_asset_ip" SET "eid"='8211419A-5BDD-11EC-A457-F60242D1C7A8',"proxy_id"='360bjyt',"asset_ip"='10.209.2.153',"asset_type"='sdf1fa',"contacts"='xxxx',"updated_at"='2023-08-31 17:43:40.442',"asset_name"='11sdfsf' WHERE id = 19712170 AND "s_asset_ip"."deleted_at" IS NULL AND "id" = 19712170
```

gorm在执行Updates时,如果结构体字段值为默认值,即零值,则会跳过该字段不更新。这里info中的IsMajor字段为false,是bool类型的零值,因此Updates执行时被跳过了

要解决这个问题,我们可以有两种方法:
1. 使用Select仅选择要更新的字段:
```go
db.Table("s_asset_ip").
  Select("eid", "proxy_id", "is_major").
  Where("id = ?", id).
  Updates(info)
```

2. 构建更新的map,避免零值字段:

```go
updateMap := map[string]interface{}{
  "eid": info.Eid,
  "proxy_id": info.ProxyID,
}

if info.IsMajor {
  updateMap["is_major"] = info.IsMajor 
}

db.Model(&SAssetIP{}).Where("id = ?", id).Updates(updateMap)
```

> GORM 的 `Create` 方法,它会插入模型的**全部字段,不会忽略零值**


## gorm json类型字段
### 查询匹配
```sql
CREATE TABLE light.s_proxy_node (
    id int8 NOT NULL DEFAULT nextval('s_data_source_nodes_id_seq'::regclass),
    node_id varchar(32) NOT NULL DEFAULT ''::character varying,
    ext jsonb NULL,
    created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP(0),
    updated_at timestamp NULL,
    deleted_at timestamp NULL
);
```
其中 ext 是jsonb类型
```json
{
    "os":"windows",
    "type":7,
    "channel":"sdns",
    "ip_list":{
        "10.18.191.83":"84:2a:fd:3b:6d:45",
        "192.168.137.1":"9e:bf:c0:0f:79:9d"
    },
    "os_info":"Microsoft Windows 10 Pro",
    "run_mod":4,
    "version":"2.6.0",
    "cpu_arch":"amd64",
    "hostname":"5CG026B3ZM",
    "master_ip":{
        "10.18.191.83":"84:2a:fd:3b:6d:45"
    },
    "uid_source":null
}
```


### 过滤检索
想要对 ext 进行过滤匹配：(借助 `->>` 指定成员)
```go
    tx = tx.Where("ext->>'hostname' like ? or ext->>'ip_list' like ?", likeKey, likeKey)
```


```go
func ListRecordNodeRaw(ctx context.Context, eid, uid, key string, lType int16, page, size int) ([]*models.SProxyNode, int64, error) {
    var ret = make([]*models.SProxyNode, 0)
    tx := ds.db.WithContext(ctx).Table(models.TableNameSProxyNode).Where("eid", eid).Where("uid", uid).Where("type", lType)
    if len(key) != 0 {
        likeKey := fmt.Sprintf("%%%s%%", key)
        tx = tx.Where("ext->>'hostname' like ? or ext->>'ip_list' like ?", likeKey, likeKey)
    }
    var count int64
    tx = tx.Count(&count)
    if page <= 0 {
        page = 1
    }
    if size <= 0 {
        size = 10
    }
    tx = tx.Limit(size).Offset((page - 1) * size)
    err := tx.Order("last_alive_time desc").Find(&ret).Error
    if err != nil {
        return nil, count, err
    }
    return ret, count, nil
}
```


### 输出指定成员
查询成员作为输出：(借助 `->>` 指定成员)

```go
    Select("ext->>'ip_list'")
```


```go
func GetIpListByEidAndType(ctx context.Context, eid string, typeId int) (map[string]string, error) {
    var ipList []string


    tx := ds.db.Table(models.TableNameSProxyNode).
        WithContext(ctx).
        Select("ext->>'ip_list'").
        Where("eid = ?", eid).
        Where("type = ?", typeId).
        Find(&ipList)


    if tx.Error != nil {
        if tx.Error == gorm.ErrRecordNotFound {
            return nil, nil
        }
        return nil, tx.Error
    }
    // IPv4:IPv6
    var dns map[string]string
    for _, item := range ipList {
        if err := json.Unmarshal([]byte(item), &dns); err != nil {
            return nil, err
        }
    }
    return dns, nil
}
```



## select 追加
```go
func GetMapGeo(ctx context.Context, tx *gorm.DB,dimensionality string, opts []biz.BaseOption) ([]*pb.GeoAttributionInfo, error) {
    db := tx.Table(models.TableNameQueryGeographicalStatus).WithContext(ctx)
    for _, opt := range opts {
        db = opt(db)
    }


    var (
        result = make([]*pb.GeoAttributionInfo, 0,10)
    )


    // 创建动态赋值
    switch dimensionality {
    case "city":
        db = db.Select("cl_city as client_geo, rd_city as rdata_geo")
    case "province":
        db = db.Select("cl_province as client_geo, rd_province as rdata_geo")
    case "country":
        db = db.Select("cl_country as client_geo, rd_country as rdata_geo")
    default:
        return nil, fmt.Errorf("GetMapGeo dimensionality 参数异常：%s\n", dimensionality)
    }


    // 构建sql
    err := db.Select(
        "max(cl_latitude) AS client_latitude, max(cl_longitude) AS client_longitudesum(pv) AS count_pv",
        ).Group("client_geo, rdata_geo").Order("client_geo desc").Offset(0).Limit(10).Find(&result)
    if err != nil && gorm.ErrRecordNotFound != err.Error {
        return nil, err.Error
    }


    return result, nil
}
```


上述利用gorm 框架查询数据库，为什么两次db.Select后，后者会将前者覆盖，如果想追加该如何修改。期望查询语句为：
```SQL
select cl_city as client_geo, rd_city as rdata_geo, max(cl_latitude) AS client_latitude, ...
```


Gorm 的 Select 方法接收一个 interface{} 参数，如果你传入一个 string 类型，它会将这个字符串直接作为查询的参数；但是如果你传入一个 []string 类型的切片，它会将切片的每个元素作为一个独立的查询参数。我们可以利用这个特性，将所有需要查询的字段放在一个切片中，然后一次性传入 Select 方法，达到字段选择追加的效果。
```go
func GetMapGeo(ctx context.Context, tx *gorm.DB, dimensionality string, opts []biz.BaseOption) ([]*pb.GeoAttributionInfo, error) {
    db := tx.Table(models.TableNameQueryGeographicalStatus).WithContext(ctx)
    for _, opt := range opts {
        db = opt(db)
    }


    var result = make([]*pb.GeoAttributionInfo, 0, 10)


    var selectFields []string


    // 根据 dimensionality 参数的值选择要查询的字段
    switch dimensionality {
    case "city":
        selectFields = append(selectFields, "cl_city as client_geo", "rd_city as rdata_geo")
    case "province":
        selectFields = append(selectFields, "cl_province as client_geo", "rd_province as rdata_geo")
    case "country":
        selectFields = append(selectFields, "cl_country as client_geo", "rd_country as rdata_geo")
    default:
        return nil, fmt.Errorf("GetMapGeo dimensionality 参数异常：%s\n", dimensionality)
    }


    // 添加其他需要查询的字段
    selectFields = append(selectFields, "max(cl_latitude) AS client_latitude", "max(cl_longitude) AS client_longitude","sum(pv) AS count_pv" )


    err := db.Select(selectFields).Group("client_geo, rdata_geo").Order("client_geo desc").Offset(0).Limit(10).Find(&result)
    if err != nil && gorm.ErrRecordNotFound != err.Error {
        return nil, err.Error
    }


    return result, nil
}
```


或者
```go
var selectExpr string


switch dimensionality {
case "city":
    selectExpr = "cl_city as client_geo, rd_city as rdata_geo"
case "province":
    selectExpr = "cl_province as client_geo, rd_province as rdata_geo"
case "country":
    selectExpr = "cl_country as client_geo, rd_country as rdata_geo"


}


var result = make([]*pb.GeoAttributionInfo, 0,10)
err := db.Select(
       selectExpr,
       "max(cl_latitude) AS client_latitude, max(cl_longitude) AS client_longitude, max(rd_latitude) AS rdata_latitude,max(rd_longitude) AS rdata_longitude, sum(countpv) AS count_pv",
       ).Group("client_geo, rdata_geo").Order("client_geo desc").Offset(0).Limit(10).Find(&result)
if err.Error != nil && gorm.ErrRecordNotFound != err.Error {
       return nil, err.Error
}
```


## 预编译的逻辑


## 参数化查询




## sql 注入
针对于 `db.Where("name = ?", userName).Find(&users)`， 其中的 userName 值为： `if(1==1, user(),"other")`，这种情况如何防止sql注入


## 查询时间带时区问题
```go
func GetEventDetail(ctx context.Context,tx *gorm.DB, opts ...biz.BaseOption) ([]*pb.MetricEventDetail, error) {
  rs := make([]*pb.MetricEventDetail, 0, 5)


  db := tx.Table(biz.TableIoc).WithContext(ctx)
  for _, opt := range opts {
    db = opt(db)
  }


  var selectFields []string
  selectFields = append(
    selectFields,
    "tnow",
    "localAddress as LocalAddress",
    "firstQueryName as FirstQueryName",
    "dictGet('s_data_uid', 'name', tuple(uid)) as name",
    "dictGet('s_threat_tag_mapping_join_parent', 'cat1', tuple(toInt64(mappingid))) as type",
  )


  result := db.Select(selectFields).Limit(5).Find(&rs)
  if result.Error != nil {
    logx.Errorf("GetEventDetail err:%v", result.Error)
    return nil, status.Errorf(code.ErrDatabaseOpt, code.String(code.ErrDatabaseOpt))
  }
  return rs, nil
}
```
返回的tnow 时间格式是:'2023-11-12T12:08:38+08:00', 期望格式: '2023-11-12T12:08:38'，应该如何实现时间输出格式自定义：


```go
  selectFields = append(
    selectFields,
    "formatDateTime(tnow, '%Y-%m-%d %R:%S') as Tnow", // 使用 FORMAT 函数控制时间格式
    "localAddress as LocalAddress",
    "firstQueryName as FirstQueryName",
    "dictGet('s_data_uid', 'name', tuple(uid)) as name",
    "dictGet('s_threat_tag_mapping_join_parent', 'cat1', tuple(toInt64(mappingid))) as type",
  )
```


## 利用interface实现通用查询接口查询


- 利用 BaseQueryRequest 传入整体查询条件
- BaseQuery 利用 interface{} 接收响应结果返回


```go
type BaseQueryRequest struct {
    tx *gorm.DB
    table string
    selectFields []string
    opts []biz.BaseOption
    group string
    order string
    offset int
    limit int
}


func GetSecurityPv (ctx context.Context,tx *gorm.DB, opts ...biz.BaseOption) ([]*pb.SeverityLevel, error) {
    rs := make([]*pb.SeverityLevel, 0, 4)
    request := &BaseQueryRequest{
        tx:           tx,
        table:        biz.TableCdns,
        selectFields: []string{
            "domainSecurityType as key",
            "uniqExact(uid,localAddress) as value",
        },
        opts: opts,
        group: "key with cube",
    }
    
    // 注意 rs 用指针
    err := ds.BaseQuery(ctx, request,&rs, opts)
    if err != nil {
        return nil, err
    }
    return rs, nil
}


func BaseQuery (ctx context.Context, base *BaseQueryRequest, rs interface{}, opts []biz.BaseOption) (err error) {
    if base == nil {
        return status.Errorf(code.ErrParamParse, code.String(code.ErrParamParse))
    }
    db := base.tx.Table(base.table).WithContext(ctx)
    for _, opt := range opts {
        db = opt(db)
    }


    db = db.Select(base.selectFields)
    if len(base.group) != 0 {
        db = db.Group(base.group)
    }


    if len(base.order) != 0 {
        db = db.Order(base.order)
    }


    if base.offset != 0 {
        db = db.Offset(base.offset)
    }


    if base.limit != 0 {
        db = db.Limit(base.limit)
    }


    result := db.Find(rs)
    if result.Error != nil {
        logx.Errorf("BaseQuery err:%v", result.Error)
        return  status.Errorf(code.ErrDatabaseOpt, code.String(code.ErrDatabaseOpt))
    }
    return nil
}
```

最终请求：
```sql
SELECT domainSecurityType as key,uniqExact(uid,localAddress) as value FROM `cdns_log` WHERE (toYYYYMMDD(tnow) BETWEEN 20231112 AND 20231113) AND source = 'C6ADCD26-E63B-11EC-BC53-B62BBA637A35' AND (tnow BETWEEN '2023-11-12 12:08:07' AND '2023-11-13 12:08:07') AND domainSecurityType != 0 GROUP BY key with cube
```

## go 空指针

```go
    result := make([]*pb.GeoSort, len(rs))
    for i, action := range rs{
        ruleName, err := bz.repo.GetRuleEnumList(ctx, RuleWithID(int32(action.Key)))
        if err != nil {
            result[i].Name = strconv.Itoa(int(action.Key))
        } else {
            result[i].Name = ruleName[0].Zh
        }
        result[i].Value = action.Value
    }
```
赋值时 result[i] 为空指针，因为使用 make 创建了一个切片，切片的元素是结构体类型，那么这个切片中的元素是零值，如果直接通过 result[i] 访问某个位置的元素，而这个位置还没有被初始化，就会导致空指针错误。

在使用 make 创建切片后，应该使用 `result[i] = &pb.GeoSort{}` 等语句为切片中的每个元素分配内存
```go
result := make([]*pb.GeoSort, len(rs))
for i, action := range rs {
    result[i] = &pb.GeoSort{} // 为切片中的每个元素分配内存
    ruleName, err := bz.repo.GetRuleEnumList(ctx, RuleWithID(int32(action.Key)))
    if err != nil {
        result[i].Name = strconv.Itoa(int(action.Key))
    } else {
        result[i].Name = ruleName[0].Zh
    }
    result[i].Value = action.Value
}
```


## find 切片参数异常：
```go
type BaseOption func(*gorm.DB) *gorm.DB


func BaseWithEID(eid string) BaseOption {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("source = ?", eid)
    }
}




func BaseQuery (ctx context.Context, base *BaseQueryRequest, rs interface{},opts []biz.BaseOption) (err error) {
    db := base.tx.Table(base.table).WithContext(ctx)
    for _, opt := range opts {
        db = opt(db)
    }


    db = db.Select(base.selectFields)
    
    result := db.Find(rs)
    if result.Error != nil {
        return result.Error
    }
    return nil
}


func GetEventIds(ctx context.Context, tx *gorm.DB, opts []biz.BaseOption) ([]uint64, error) {
    var rs = make([]uint64, 0, 100)
    request := &BaseQueryRequest{
        tx:           tx,
        table:        "table_name",
        selectFields: []string{
            "groupUniqArray(event_id) AS ids",
        },
        opts: opts,
    }


    err := ds.BaseQuery(ctx, request,&rs, opts)
    if err != nil {
        return nil, err
    }
    return rs, nil
}




func main(){
    // database init
    var tx *gorm.DB
    tx = initDB()
    ...


    opts := make([]BaseOption, 0,10)
    opts = append(opts, BaseWithEID(eid))
    
    ids, err := GetEventIds(context.Background(), tx, opts)
    ...
}
```
将 `var rs = make([]uint64, 0, 100)` 传给 `BaseQuery` 的 rs 参数时，gorm 的 `find` 返回异常：
`sql: Scan error on column index 0, name "ids": converting driver.Value type []uint64 ("[23267 23466...`


解决办法：


利用结构体完成，因为 find 需要一个 interface{}，但是提供了一个切片，所以用结构体替代
```go
func GetEventIds(ctx context.Context, tx *gorm.DB, opts []biz.BaseOption) ([]uint64, error) {
    //var rs = make([]uint64, 0, 100)
    type EventIDResult struct {
        Item []uint64 `gorm:"column:ids"`
    }
    var rs EventIDResult
    request := &BaseQueryRequest{
        tx:           tx,
        table:        models.TableNameSEventOrder,
        selectFields: []string{
            "groupUniqArray(event_id) AS ids",
        },
        opts: opts,
    }


    err := ds.BaseQuery(ctx, request, &rs, opts)
    if err != nil {
        return nil, err
    }
    return rs.Item, nil
}
```

## `[error] unsupported data type: &[]`
```go
func GetEventIds(ctx context.Context, tx *gorm.DB, opts []biz.BaseOption) ([]uint64, error) {
    type EventIDResult struct {
        Item []uint64 `gorm:"column:ids"`
    }
    var rs EventIDResult
    request := &BaseQueryRequest{
        tx:           tx,
        table:        models.TableNameSEventOrder,
        selectFields: []string{
            "groupUniqArray(event_id) AS ids",
        },
        opts: opts,
    }


    err := ds.BaseQuery(ctx, request, &rs, opts)
    if err != nil {
        return nil, err
    }
    return rs.Item, nil
}


func BaseQuery (ctx context.Context, base *BaseQueryRequest, rs interface{},opts []biz.BaseOption) (err error) {
    db := base.tx.Table(base.table).WithContext(ctx)
    for _, opt := range opts {
        db = opt(db)
    }


    db = db.Select(base.selectFields)
    
    result := db.Find(rs)
    if result.Error != nil {
        return result.Error
    }
    return nil
}
```
利用gorm 执行 BaseQuery 的 find 是提示：`[error] unsupported data type: &[]`
