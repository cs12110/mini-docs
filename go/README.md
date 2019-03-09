# For GO

狗语言,你值得拥有.

Java拍了一下我的肩膀说: 再见,老朋友.

----

## 笔记

### private/public

`go`里面没有像java里面声明一个方法或者属性是公开(`public`)还是私有(`private`),但ta们更粗暴

- 一个方法的首字母为大写,则是public,否则就是private
- 一个属性的首字母为大写,则是public,否则就是private

眼泪不争气的流下来了.jpg

### import

`go`里面不允许存在`import`进来到没有应用的`package`,这个,我还是很喜欢的,解决方法就是把不需要的`package`去除.


Q: 那么该怎么引入mysql的东西呀?

A: follow me,在项目配置的`GOPATH`目录下面安装依赖,这个类似于maven的本地仓库位置的意思.在项目里面引入相关依赖即可.


```go
package main

import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "root:Root@3306@tcp(47.98.104.252:3306)/spring_rookie?charset=utf8")
	if err != nil {
		println("err", err.Error())
	}

	sql := "insert into rookie_t(name) value ('123')"
	result, err := db.Exec(sql)
	insertId, err := result.LastInsertId()
	println(insertId)
	println("This is db app")
}
```

### json

二进制又不会,只有用json才能做接口这样子.

```go
package main

import "encoding/json"

type Student struct {
	// 将该属性字段转换成小写
	No   string `json:"no"`
	Name string
	Age  int
}

/**
 * 对象转换成json字符串
 */
func obj2JsonStr(student Student) string {
	value, _ := json.Marshal(student)
	return string(value)
}

/**
 * 将json转换成对象
 */
func jsonStr2Obj(jsonStr string) Student {
	var stu Student
	err := json.Unmarshal([]byte(jsonStr), &stu)
	if err != nil {
		println("err", err.Error())
	}
	return stu
}

func main() {
	student := Student{"cs12110", "3306", 60}
	jsonStr := obj2JsonStr(student)
	println(jsonStr)

	stu := jsonStr2Obj(jsonStr)
	println(stu.No)
	println(stu.Name)
	println(stu.Age)
}
```

测试结果

```go
{"no":"cs12110","Name":"3306","Age":60}
cs12110
3306
60
```

---

## 参考资料

a. [菜鸟教程](http://www.runoob.com/go/go-tutorial.html)