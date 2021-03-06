---
name: 高级查询
sort: 4
---

# 高级查询

ORM 以 **QuerySeter** 来组织查询，每个返回 **QuerySeter** 的方法都会获得一个新的 **QuerySeter** 对象。

基本使用方法:

```go
o := orm.NewOrm()

// 获取 QuerySeter 对象，user 为表名
qs := o.QueryTable("user")

// 也可以直接使用 Model 结构体作为表名
qs = o.QueryTable(&User)

// 也可以直接使用对象作为表名
user := new(User)
qs = o.QueryTable(user) // 返回 QuerySeter
```
## expr

QuerySeter 中用于描述字段和 sql 操作符，使用简单的 expr 查询方法

字段组合的前后顺序依照表的关系，比如 User 表拥有 Profile 的外键，那么对 User 表查询对应的 Profile.Age 为条件，则使用 `Profile__Age` 注意，字段的分隔符号使用双下划线 `__`，除了描述字段， expr 的尾部可以增加操作符以执行对应的 sql 操作。比如 `Profile__Age__gt` 代表 Profile.Age > 18 的条件查询。

注释后面将描述对应的 sql 语句，仅仅是描述 expr 的类似结果，并不代表实际生成的语句。

```go
qs.Filter("id", 1) // WHERE id = 1
qs.Filter("profile__age", 18) // WHERE profile.age = 18
qs.Filter("Profile__Age", 18) // 使用字段名和 Field 名都是允许的
qs.Filter("profile__age__gt", 18) // WHERE profile.age > 18
qs.Filter("profile__age__gte", 18) // WHERE profile.age >= 18
qs.Filter("profile__age__in", 18, 20) // WHERE profile.age IN (18, 20)

qs.Filter("profile__age__in", 18, 20).Exclude("profile__lt", 1000)
// WHERE profile.age IN (18, 20) AND NOT profile_id < 1000
```
## Operators

当前支持的操作符号：

* [exact](#exact) / [iexact](#iexact) 等于
* [contains](#contains) / [icontains](#icontains) 包含
* [gt / gte](#gt / gte) 大于 / 大于等于
* [lt / lte](#lt / lte) 小于 / 小于等于
* [startswith](#startswith) / [istartswith](#istartswith) 以...起始
* [endswith](#endswith) / [iendswith](#iendswith) 以...结束
* [in](#in)
* [isnull](#isnull)

后面以 `i` 开头的表示：大小写不敏感

### exact

Filter / Exclude / Condition expr 的默认值

```go
qs.Filter("name", "slene") // WHERE name = 'slene'
qs.Filter("name__exact", "slene") // WHERE name = 'slene'
// 使用 = 匹配，大小写是否敏感取决于数据表使用的 collation
qs.Filter("profile_id", nil) // WHERE profile_id IS NULL
```

### iexact

```go
qs.Filter("name__iexact", "slene")
// WHERE name LIKE 'slene'
// 大小写不敏感，匹配任意 'Slene' 'sLENE'
```

### contains

```go
qs.Filter("name__contains", "slene")
// WHERE name LIKE BINARY '%slene%'
// 大小写敏感, 匹配包含 slene 的字符
```

### icontains

```go
qs.Filter("name__icontains", "slene")
// WHERE name LIKE '%slene%'
// 大小写不敏感, 匹配任意 'im Slene', 'im sLENE'
```

### in

```go
qs.Filter("age__in", 17, 18, 19, 20)
// WHERE age IN (17, 18, 19, 20)


ids:=[]int{17,18,19,20}
qs.Filter("age__in", ids)
// WHERE age IN (17, 18, 19, 20)

// 同上效果
```

### gt / gte

```go
qs.Filter("profile__age__gt", 17)
// WHERE profile.age > 17

qs.Filter("profile__age__gte", 18)
// WHERE profile.age >= 18
```

### lt / lte

```go
qs.Filter("profile__age__lt", 17)
// WHERE profile.age < 17

qs.Filter("profile__age__lte", 18)
// WHERE profile.age <= 18
```

### startswith

```go
qs.Filter("name__startswith", "slene")
// WHERE name LIKE BINARY 'slene%'
// 大小写敏感, 匹配以 'slene' 起始的字符串
```

### istartswith

```go
qs.Filter("name__istartswith", "slene")
// WHERE name LIKE 'slene%'
// 大小写不敏感, 匹配任意以 'slene', 'Slene' 起始的字符串
```

### endswith

```go
qs.Filter("name__endswith", "slene")
// WHERE name LIKE BINARY '%slene'
// 大小写敏感, 匹配以 'slene' 结束的字符串
```

### iendswith

```go
qs.Filter("name__iendswithi", "slene")
// WHERE name LIKE '%slene'
// 大小写不敏感, 匹配任意以 'slene', 'Slene' 结束的字符串
```

### isnull

```go
qs.Filter("profile__isnull", true)
qs.Filter("profile_id__isnull", true)
// WHERE profile_id IS NULL

qs.Filter("profile__isnull", false)
// WHERE profile_id IS NOT NULL
```

## 高级查询接口使用

QuerySeter 是高级查询使用的接口，我们来熟悉下他的接口方法
```go
type QuerySeter interface {
// add condition expression to QuerySeter.
// for example:
//	filter by UserName == 'slene'
//	qs.Filter("UserName", "slene")
//	sql : left outer join profile on t0.id1==t1.id2 where t1.age == 28
//	Filter("profile__Age", 28)
// 	 // time compare
//	qs.Filter("created", time.Now())
Filter(string, ...interface{}) QuerySeter
// add raw sql to querySeter.
// for example:
// qs.FilterRaw("user_id IN (SELECT id FROM profile WHERE age>=18)")
// //sql-> WHERE user_id IN (SELECT id FROM profile WHERE age>=18)
FilterRaw(string, string) QuerySeter
// add NOT condition to querySeter.
// have the same usage as Filter
Exclude(string, ...interface{}) QuerySeter
// set condition to QuerySeter.
// sql's where condition
//	cond := orm.NewCondition()
//	cond1 := cond.And("profile__isnull", false).AndNot("status__in", 1).Or("profile__age__gt", 2000)
//	//sql-> WHERE T0.`profile_id` IS NOT NULL AND NOT T0.`Status` IN (?) OR T1.`age` >  2000
//	num, err := qs.SetCond(cond1).Count()
SetCond(*Condition) QuerySeter
// get condition from QuerySeter.
// sql's where condition
//  cond := orm.NewCondition()
//  cond = cond.And("profile__isnull", false).AndNot("status__in", 1)
//  qs = qs.SetCond(cond)
//  cond = qs.GetCond()
//  cond := cond.Or("profile__age__gt", 2000)
//  //sql-> WHERE T0.`profile_id` IS NOT NULL AND NOT T0.`Status` IN (?) OR T1.`age` >  2000
//  num, err := qs.SetCond(cond).Count()
GetCond() *Condition
// add LIMIT value.
// args[0] means offset, e.g. LIMIT num,offset.
// if Limit <= 0 then Limit will be set to default limit ,eg 1000
// if QuerySeter doesn't call Limit, the sql's Limit will be set to default limit, eg 1000
//  for example:
//	qs.Limit(10, 2)
//	// sql-> limit 10 offset 2
Limit(limit interface{}, args ...interface{}) QuerySeter
// add OFFSET value
// same as Limit function's args[0]
Offset(offset interface{}) QuerySeter
// add GROUP BY expression
// for example:
//	qs.GroupBy("id")
GroupBy(exprs ...string) QuerySeter
// add ORDER expression.
// "column" means ASC, "-column" means DESC.
// for example:
//	qs.OrderBy("-status")
OrderBy(exprs ...string) QuerySeter
// add FORCE INDEX expression.
// for example:
//	qs.ForceIndex(`idx_name1`,`idx_name2`)
// ForceIndex, UseIndex , IgnoreIndex are mutually exclusive
ForceIndex(indexes ...string) QuerySeter
// add USE INDEX expression.
// for example:
//	qs.UseIndex(`idx_name1`,`idx_name2`)
// ForceIndex, UseIndex , IgnoreIndex are mutually exclusive
UseIndex(indexes ...string) QuerySeter
// add IGNORE INDEX expression.
// for example:
//	qs.IgnoreIndex(`idx_name1`,`idx_name2`)
// ForceIndex, UseIndex , IgnoreIndex are mutually exclusive
IgnoreIndex(indexes ...string) QuerySeter
// set relation model to query together.
// it will query relation models and assign to parent model.
// for example:
//	// will load all related fields use left join .
// 	qs.RelatedSel().One(&user)
//	// will  load related field only profile
//	qs.RelatedSel("profile").One(&user)
//	user.Profile.Age = 32
RelatedSel(params ...interface{}) QuerySeter
// Set Distinct
// for example:
//  o.QueryTable("policy").Filter("Groups__Group__Users__User", user).
//    Distinct().
//    All(&permissions)
Distinct() QuerySeter
// set FOR UPDATE to query.
// for example:
//  o.QueryTable("user").Filter("uid", uid).ForUpdate().All(&users)
ForUpdate() QuerySeter
// return QuerySeter execution result number
// for example:
//	num, err = qs.Filter("profile__age__gt", 28).Count()
Count() (int64, error)
// check result empty or not after QuerySeter executed
// the same as QuerySeter.Count > 0
Exist() bool
// execute update with parameters
// for example:
//	num, err = qs.Filter("user_name", "slene").Update(Params{
//		"Nums": ColValue(Col_Minus, 50),
//	}) // user slene's Nums will minus 50
//	num, err = qs.Filter("UserName", "slene").Update(Params{
//		"user_name": "slene2"
//	}) // user slene's  name will change to slene2
Update(values Params) (int64, error)
// delete from table
// for example:
//	num ,err = qs.Filter("user_name__in", "testing1", "testing2").Delete()
// 	//delete two user  who's name is testing1 or testing2
Delete() (int64, error)
// return a insert queryer.
// it can be used in times.
// example:
// 	i,err := sq.PrepareInsert()
// 	num, err = i.Insert(&user1) // user table will add one record user1 at once
//	num, err = i.Insert(&user2) // user table will add one record user2 at once
//	err = i.Close() //don't forget call Close
PrepareInsert() (Inserter, error)
// query all data and map to containers.
// cols means the columns when querying.
// for example:
//	var users []*User
//	qs.All(&users) // users[0],users[1],users[2] ...
All(container interface{}, cols ...string) (int64, error)
// query one row data and map to containers.
// cols means the columns when querying.
// for example:
//	var user User
//	qs.One(&user) //user.UserName == "slene"
One(container interface{}, cols ...string) error
// query all data and map to []map[string]interface.
// expres means condition expression.
// it converts data to []map[column]value.
// for example:
//	var maps []Params
//	qs.Values(&maps) //maps[0]["UserName"]=="slene"
Values(results *[]Params, exprs ...string) (int64, error)
// query all data and map to [][]interface
// it converts data to [][column_index]value
// for example:
//	var list []ParamsList
//	qs.ValuesList(&list) // list[0][1] == "slene"
ValuesList(results *[]ParamsList, exprs ...string) (int64, error)
// query all data and map to []interface.
// it's designed for one column record set, auto change to []value, not [][column]value.
// for example:
//	var list ParamsList
//	qs.ValuesFlat(&list, "UserName") // list[0] == "slene"
ValuesFlat(result *ParamsList, expr string) (int64, error)
// query all rows into map[string]interface with specify key and value column name.
// keyCol = "name", valueCol = "value"
// table data
// name  | value
// total | 100
// found | 200
// to map[string]interface{}{
// 	"total": 100,
// 	"found": 200,
// }
RowsToMap(result *Params, keyCol, valueCol string) (int64, error)
// query all rows into struct with specify key and value column name.
// keyCol = "name", valueCol = "value"
// table data
// name  | value
// total | 100
// found | 200
// to struct {
// 	Total int
// 	Found int
// }
RowsToStruct(ptrStruct interface{}, keyCol, valueCol string) (int64, error)
}
```

* 每个返回 QuerySeter 的 api 调用时都会新建一个 QuerySeter，不影响之前创建的。

* 高级查询使用 Filter 和 Exclude 来做常用的条件查询。囊括两种清晰的过滤规则：包含， 排除

### Filter

用来过滤查询结果，起到 **包含条件** 的作用

多个 Filter 之间使用 `AND` 连接

```go
qs.Filter("profile__isnull", true).Filter("name", "slene")
// WHERE profile_id IS NULL AND name = 'slene'
```

### Exclude

用来过滤查询结果，起到 **排除条件** 的作用

使用 `NOT` 排除条件

多个 Exclude 之间使用 `AND` 连接

```go
qs.Exclude("profile__isnull", true).Filter("name", "slene")
// WHERE NOT profile_id IS NULL AND name = 'slene'
```

### SetCond

自定义条件表达式

```go
cond := orm.NewCondition()
cond1 := cond.And("profile__isnull", false).AndNot("status__in", 1).Or("profile__age__gt", 2000)

qs := orm.QueryTable("user")
qs = qs.SetCond(cond1)
// WHERE ... AND ... AND NOT ... OR ...

cond2 := cond.AndCond(cond1).OrCond(cond.And("name", "slene"))
qs = qs.SetCond(cond2).Count()
// WHERE (... AND ... AND NOT ... OR ...) OR ( ... )
```

### Limit

限制最大返回数据行数，第二个参数可以设置 `Offset`

```go
var DefaultRowsLimit = 1000 // ORM 默认的 limit 值为 1000

// 默认情况下 select 查询的最大行数为 1000
// LIMIT 1000

qs.Limit(10)
// LIMIT 10

qs.Limit(10, 20)
// LIMIT 10 OFFSET 20 注意跟 SQL 反过来的

qs.Limit(-1)
// no limit

qs.Limit(-1, 100)
// LIMIT 18446744073709551615 OFFSET 100
// 18446744073709551615 是 1<<64 - 1 用来指定无 limit 限制 但有 offset 偏移的情况
```

### Offset

设置 偏移行数

```go
qs.Offset(20)
// LIMIT 1000 OFFSET 20
```

### GroupBy

```go
qs.GroupBy("id", "age")
// GROUP BY id,age
```

### OrderBy

参数使用 **expr**

在 expr 前使用减号 `-` 表示 `DESC` 的排列

```go
qs.OrderBy("id", "-profile__age")
// ORDER BY id ASC, profile.age DESC

qs.OrderBy("-profile__age", "profile")
// ORDER BY profile.age DESC, profile_id ASC
```

### ForceIndex

强迫走索引。使用该选选项请确认数据库支持该特性。

```go
qs.ForceIndex(`idx_name1`,`idx_name2`)
```

### UseIndex

使用索引。使用该特性的时候需要确认数据库是否支持该特性，以及该特性的具体含义。例如，部分数据库对于该选项是当成一种建议来执行的。

即，即便用户使用了`UseIndex`方法，但是数据库在具体执行的时候，也可能不会使用设定的索引。

```go
qs.UseIndex(`idx_name1`,`idx_name2`)
```

### IgnoreIndex

忽略索引。请确认数据是否支持该选项。

```go
qs.IgnoreIndex(`idx_name1`,`idx_name2`)
```

### Distinct

对应 sql 的 `distinct` 语句, 返回指定字段不重复的值.

```go
qs.Distinct()
// SELECT DISTINCT
```

### RelatedSel

关系查询，参数使用 **expr**

```go
var DefaultRelsDepth = 5 // 默认情况下直接调用 RelatedSel 将进行最大 5 层的关系查询

qs := o.QueryTable("post")

qs.RelatedSel()
// INNER JOIN user ... LEFT OUTER JOIN profile ...

qs.RelatedSel("user")
// INNER JOIN user ...
// 设置 expr 只对设置的字段进行关系查询

// 对设置 null 属性的 Field 将使用 LEFT OUTER JOIN
```

### Count

依据当前的查询条件，返回结果行数

```go
cnt, err := o.QueryTable("user").Count() // SELECT COUNT(*) FROM USER
fmt.Printf("Count Num: %s, %s", cnt, err)
```

### Exist

```go
exist := o.QueryTable("user").Filter("UserName", "Name").Exist()
fmt.Printf("Is Exist: %s", exist)
```

### Update

依据当前查询条件，进行批量更新操作

```go
num, err := o.QueryTable("user").Filter("name", "slene").Update(orm.Params{
	"name": "astaxie",
})
fmt.Printf("Affected Num: %s, %s", num, err)
// SET name = "astaixe" WHERE name = "slene"
```

原子操作增加字段值

```go
// 假设 user struct 里有一个 nums int 字段
num, err := o.QueryTable("user").Update(orm.Params{
	"nums": orm.ColValue(orm.ColAdd, 100),
})
// SET nums = nums + 100
```

orm.ColValue 支持以下操作

```go
ColAdd      // 加
ColMinus    // 减
ColMultiply // 乘
ColExcept   // 除
```

### Delete

依据当前查询条件，进行批量删除操作

```go
num, err := o.QueryTable("user").Filter("name", "slene").Delete()
fmt.Printf("Affected Num: %s, %s", num, err)
// DELETE FROM user WHERE name = "slene"
```

### PrepareInsert

用于一次 prepare 多次 insert 插入，以提高批量插入的速度。

```go
var users []*User
...
qs := o.QueryTable("user")
i, _ := qs.PrepareInsert()
for _, user := range users {
	id, err := i.Insert(user)
	if err == nil {
		...
	}
}
// PREPARE INSERT INTO user (`name`, ...) VALUES (?, ...)
// EXECUTE INSERT INTO user (`name`, ...) VALUES ("slene", ...)
// EXECUTE ...
// ...
i.Close() // 别忘记关闭 statement
```

### All

返回对应的结果集对象

All 的参数支持 *[]Type 和 *[]*Type 两种形式的 slice

```go
var users []*User
num, err := o.QueryTable("user").Filter("name", "slene").All(&users)
fmt.Printf("Returned Rows Num: %s, %s", num, err)
```

All / Values / ValuesList / ValuesFlat 受到 [Limit](#limit) 的限制，默认最大行数为 1000

可以指定返回的字段：

```go
type Post struct {
	Id      int
	Title   string
	Content string
	Status  int
}

// 只返回 Id 和 Title
var posts []Post
o.QueryTable("post").Filter("Status", 1).All(&posts, "Id", "Title")
```

对象的其他字段值将会是对应类型的默认值

### One

尝试返回单条记录

```go
var user User
err := o.QueryTable("user").Filter("name", "slene").One(&user)
if err == orm.ErrMultiRows {
	// 多条的时候报错
	fmt.Printf("Returned Multi Rows Not One")
}
if err == orm.ErrNoRows {
	// 没有找到记录
	fmt.Printf("Not row found")
}
```

可以指定返回的字段：

```go
// 只返回 Id 和 Title
var post Post
o.QueryTable("post").Filter("Content__istartswith", "prefix string").One(&post, "Id", "Title")
```

对象的其他字段值将会是对应类型的默认值

### Values

返回结果集的 key => value 值

key 为Model里的Field name, value的值是interface{}类型,例如，如果你要将value赋值给struct中的某字段，需要根据结构体对应字段类型使用[断言](https://golang.org/ref/spec#Type_assertions)获取真实值。举例:`Name : m["Name"].(string)`

```go
var maps []orm.Params
num, err := o.QueryTable("user").Values(&maps)
if err == nil {
	fmt.Printf("Result Nums: %d\n", num)
	for _, m := range maps {
		fmt.Println(m["Id"], m["Name"])
	}
}
```

返回指定的 Field 数据

**TODO**: 暂不支持级联查询 **RelatedSel** 直接返回 Values

但可以直接指定 expr 级联返回需要的数据

```go
var maps []orm.Params
num, err := o.QueryTable("user").Values(&maps, "id", "name", "profile", "profile__age")
if err == nil {
	fmt.Printf("Result Nums: %d\n", num)
	for _, m := range maps {
		fmt.Println(m["Id"], m["Name"], m["Profile"], m["Profile__Age"])
		// map 中的数据都是展开的，没有复杂的嵌套
	}
}
```

### ValuesList

顾名思义，返回的结果集以slice存储

结果的排列与 Model 中定义的 Field 顺序一致

返回的每个元素值以 string 保存

```go
var lists []orm.ParamsList
num, err := o.QueryTable("user").ValuesList(&lists)
if err == nil {
	fmt.Printf("Result Nums: %d\n", num)
	for _, row := range lists {
		fmt.Println(row)
	}
}
```

当然也可以指定 expr 返回指定的 Field

```go
var lists []orm.ParamsList
num, err := o.QueryTable("user").ValuesList(&lists, "name", "profile__age")
if err == nil {
	fmt.Printf("Result Nums: %d\n", num)
	for _, row := range lists {
		fmt.Printf("Name: %s, Age: %s\m", row[0], row[1])
	}
}
```

### ValuesFlat

只返回特定的 Field 值，将结果集展开到单个 slice 里

```go
var list orm.ParamsList
num, err := o.QueryTable("user").ValuesFlat(&list, "name")
if err == nil {
	fmt.Printf("Result Nums: %d\n", num)
	fmt.Printf("All User Names: %s", strings.Join(list, ", "))
}
```

## 关系查询

以例子里的[模型定义](/docs.mvc_model_orm)来看下怎么进行关系查询

#### User 和 Profile 是 OneToOne 的关系

已经取得了 User 对象，查询 Profile：

```go
user := &User{Id: 1}
o.Read(user)
if user.Profile != nil {
	o.Read(user.Profile)
}
```

直接关联查询：

```go
user := &User{}
o.QueryTable("user").Filter("Id", 1).RelatedSel().One(user)
// 自动查询到 Profile
fmt.Println(user.Profile)
// 因为在 Profile 里定义了反向关系的 User，所以 Profile 里的 User 也是自动赋值过的，可以直接取用。
fmt.Println(user.Profile.User)

// [SELECT T0.`id`, T0.`name`, T0.`profile_id`, T1.`id`, T1.`age` FROM `user` T0 INNER JOIN `profile` T1 ON T1.`id` = T0.`profile_id` WHERE T0.`id` = ? LIMIT 1000] - `1`
```

通过 User 反向查询 Profile：

```go
var profile Profile
err := o.QueryTable("profile").Filter("User__Id", 1).One(&profile)
if err == nil {
	fmt.Println(profile)
}
```

#### Post 和 User 是 ManyToOne 关系，也就是 ForeignKey 为 User

```go
type Post struct {
	Id    int
	Title string
	User  *User  `orm:"rel(fk)"`
	Tags  []*Tag `orm:"rel(m2m)"`
}
```

```go
var posts []*Post
num, err := o.QueryTable("post").Filter("User", 1).RelatedSel().All(&posts)
if err == nil {
	fmt.Printf("%d posts read\n", num)
	for _, post := range posts {
		fmt.Printf("Id: %d, UserName: %d, Title: %s\n", post.Id, post.User.UserName, post.Title)
	}
}
// [SELECT T0.`id`, T0.`title`, T0.`user_id`, T1.`id`, T1.`name`, T1.`profile_id`, T2.`id`, T2.`age` FROM `post` T0 INNER JOIN `user` T1 ON T1.`id` = T0.`user_id` INNER JOIN `profile` T2 ON T2.`id` = T1.`profile_id` WHERE T0.`user_id` = ? LIMIT 1000] - `1`
```

根据 Post.Title 查询对应的 User：

RegisterModel 时，ORM 也会自动建立 User 中 Post 的反向关系，所以可以直接进行查询

```go
var user User
err := o.QueryTable("user").Filter("Post__Title", "The Title").Limit(1).One(&user)
if err == nil {
	fmt.Printf(user)
}
```

#### Post 和 Tag 是 ManyToMany 关系

设置 rel(m2m) 以后，ORM 会自动创建中间表

```go
type Post struct {
	Id    int
	Title string
	User  *User  `orm:"rel(fk)"`
	Tags  []*Tag `orm:"rel(m2m)"`
}
```

```go
type Tag struct {
	Id    int
	Name  string
	Posts []*Post `orm:"reverse(many)"`
}
```

一条 Post 纪录可能对应不同的 Tag 纪录,一条 Tag 纪录可能对应不同的 Post 纪录，所以 Post 和 Tag 属于多对多关系,通过 tag name 查询哪些 post 使用了这个 tag

```go
var posts []*Post
num, err := dORM.QueryTable("post").Filter("Tags__Tag__Name", "golang").All(&posts)
```

通过 post title 查询这个 post 有哪些 tag

```go
var tags []*Tag
num, err := dORM.QueryTable("tag").Filter("Posts__Post__Title", "Introduce Beego ORM").All(&tags)
```

## 载入关系字段

LoadRelated 用于载入模型的关系字段，包括所有的 rel/reverse - one/many 关系

ManyToMany 关系字段载入

```go
// 载入相应的 Tags
post := Post{Id: 1}
err := o.Read(&post)
num, err := o.LoadRelated(&post, "Tags")
```

```go
// 载入相应的 Posts
tag := Tag{Id: 1}
err := o.Read(&tag)
num, err := o.LoadRelated(&tag, "Posts")
```

User 是 Post 的 ForeignKey，对应的 ReverseMany 关系字段载入

```go
type User struct {
	Id    int
	Name  string
	Posts []*Post `orm:"reverse(many)"`
}

user := User{Id: 1}
err := dORM.Read(&user)
num, err := dORM.LoadRelated(&user, "Posts")
for _, post := range user.Posts {
	//...
}
```

## 多对多关系操作

* type QueryM2Mer interface {
	* [Add(...interface{}) (int64, error)](#querym2mer-add)
	* [Remove(...interface{}) (int64, error)](#querym2mer-remove)
	* [Exist(interface{}) bool](#querym2mer-exist)
	* [Clear() (int64, error)](#querym2mer-clear)
	* [Count() (int64, error)](#querym2mer-count)
* }

创建一个 QueryM2Mer 对象

```go
o := orm.NewOrm()
post := Post{Id: 1}
m2m := o.QueryM2M(&post, "Tags")
// 第一个参数的对象，主键必须有值
// 第二个参数为对象需要操作的 M2M 字段
// QueryM2Mer 的 api 将作用于 Id 为 1 的 Post
```

### QueryM2Mer Add

```go
tag := &Tag{Name: "golang"}
o.Insert(tag)

num, err := m2m.Add(tag)
if err == nil {
	fmt.Println("Added nums: ", num)
}
```

Add 支持多种类型 Tag *Tag []*Tag []Tag []interface{}

```go
var tags []*Tag
...
// 读取 tags 以后
...
num, err := m2m.Add(tags)
if err == nil {
	fmt.Println("Added nums: ", num)
}
// 也可以多个作为参数传入
// m2m.Add(tag1, tag2, tag3)
```

### QueryM2Mer Remove

从M2M关系中删除 tag

Remove 支持多种类型 Tag *Tag []*Tag []Tag []interface{}

```go
var tags []*Tag
...
// 读取 tags 以后
...
num, err := m2m.Remove(tags)
if err == nil {
	fmt.Println("Removed nums: ", num)
}
// 也可以多个作为参数传入
// m2m.Remove(tag1, tag2, tag3)
```

### QueryM2Mer Exist

判断 Tag 是否存在于 M2M 关系中

```go
if m2m.Exist(&Tag{Id: 2}) {
	fmt.Println("Tag Exist")
}
```

### QueryM2Mer Clear

清除所有 M2M 关系

```go
nums, err := m2m.Clear()
if err == nil {
	fmt.Println("Removed Tag Nums: ", nums)
}
```

### QueryM2Mer Count

计算 Tag 的数量

```go
nums, err := m2m.Count()
if err == nil {
	fmt.Println("Total Nums: ", nums)
}
```
