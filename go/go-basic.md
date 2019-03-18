# Go basic

于是,go 在这里开始了.

参考资料: [菜鸟教程](http://www.runoob.com/go/go-tutorial.html)

---

## 基础语法

### 判断

```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	for i := 0; i < 10; i++ {
		// rand.Intn(max):生成[0,max)之间的数
		num := rand.Intn(100)
		if num < 50 {
			fmt.Println(num, " less then ", 50)
		} else {
			fmt.Println(num, " greate then ", 50)
		}
	}
}
```

### 循环

```go
package main

import (
	"fmt"
)

func main() {
	sum := 0
	numArr := [6]int{1, 2, 3, 4}

	// 普通for
	for index := 0; index < 10; index++ {
		fmt.Println(index)
		sum += index
	}
	fmt.Println(sum)

	// 遍历数组
	for i, x := range numArr {
		fmt.Printf("%d:%d", i, x)
	}
}
```

---

## 数据结构

### 实体类

当然少不了,传说中的实体类了,面向对象,没有实体类,这宇宙将毫无意义.

```go
package main

import (
	"fmt"
)

type Book struct {
	Name    string
	Author  string
	Page    int
	Price   float32
	Summary string
}

func main() {
	book := new(Book)
	book.Author = "jk"
	book.Name = "harry potter"
	book.Page = 300
	book.Summary = "About magic"

	fmt.Println(info(book))

}

/**
 * `*Book`传递对象
 */
func info(book *Book) string {
	return book.Name + "->" + book.Author
}
```

### 数组

```go
package main

import "strconv"

type Book struct {
	Name   string
	Author string
}

func main() {
	var bookArr [5] Book
	for i := 0; i < 5; i++ {
		book := new(Book)
		// strconv.Itoa: 转换int 2 string
		book.Name = "index" + strconv.Itoa(i)
		book.Author = "index" + strconv.Itoa(i)

		bookArr[i] = *book
	}
	show(bookArr)
}

func show(arr [5] Book) {
	for b := range arr {
		each := arr[b]
		println(each.Name, ":", each.Author)
	}
}
```

### map

```go
package main

func main() {
	// 创建map结构,map[keyType]valueType
	var cityMap map[string]string
	//初始化cityMap
	cityMap = make(map[string]string)

	// 另一种初始化方式
	// cityMap := map[string]string{"France": "Paris", "Italy": "Rome"}

	cityMap["gz"] = "广州"
	cityMap["sz"] = "深圳"
	cityMap["mkd"] = "马孔多"

	// 获取广州
	println(cityMap["gz"])
	// 获取出来为空
	println(cityMap["other"])

	// 删除广州
	delete(cityMap, "gz")

	// 获取所有的key
	for key := range cityMap {
		println(key, "->", cityMap[key])
	}
}
```

### slice

Q: 在 go 里面有定长度的数组,那么有没有像 java 里面的 list 的可变长度东西呀?

A: 有的,大人,这边请.

```go
package main

func main() {
	// 和数组的区别就是,数组的声明携带了固定长度
	var arr []int

	// len: 获取长度
	// cap: 获取容量
	println(len(arr), "", cap(arr))

	arr = append(arr, 0)
	arr = append(arr, 1)
	arr = append(arr, 2)

	println(len(arr), "", cap(arr))

	println("--------------")

	for _, n := range arr {
		println(n)
	}

	println("--------------")
	// 取出第一个至第三个元素
	ints := arr[0:2]
	for _, n := range ints {
		println(n)
	}
}
```

---

## 常规应用

### 时间处理

```go
package main

import (
	"time"
)

func main() {
	year := time.Now().Year()
	month := time.Now().Month()
	day := time.Now().Day()
	hour := time.Now().Hour()
	minute := time.Now().Minute()
	second := time.Now().Second()
	nano := time.Now().Nanosecond()

	println(year, "/", month, "/", day, " ", hour, ":", minute, ":", second, ",", nano)

	// 获取当前时间,2006-01-02 15:04:05为固定时间格式字符串,听闻为go的birthday
	formatDate := time.Now().Format("2006-01-02 15:04:05")
	println(formatDate)
}
```

### 继承

```go
import "encoding/json"

type Animal struct {
	Name string
}

type Pig struct {
	// 类似java里面的组合方式
	Animal
	Age int
}

func main() {
	pig := new(Pig)
	pig.Name = "pig"
	pig.Age = 10

	println(toJson(pig))

}

func toJson(pig *Pig) string {
	value, _ := json.Marshal(pig)
	return string(value)
}
```

### 接口

```go
package main

import (
	"encoding/json"
	"fmt"
)

/**
 * 定义接口
 */
type Info interface {
	say() string
	details() string
}

/**
 * 定义实体类
 */
type Book struct {
	Name    string
	Author  string
	Page    int
	Price   float32
	Summary string
}

/**
 * book实现接口Info#say方法
 *
 */
func (book Book) say() string {
	return book.Author + "->" + book.Name + ":" + book.Summary
}

/**
 * book实现接口Info#details方法
 */
func (book Book) details() string {
	value, _ := json.Marshal(book)
	return string(value)
}

func main() {
	book := new(Book)
	book.Author = "jk"
	book.Name = "harry potter"
	book.Page = 300
	book.Summary = "About magic"

	fmt.Println(book.say())
	fmt.Println(book.details())
}
```

### JSON 处理

Q: 实体类转换成 JSON,要怎么提高重用性呀?而不是 student 一个转换方法,tree 一个转换方法?

A: `interface{}`,你值得拥有.

```go
package main

import (
	"encoding/json"
)

type Teacher struct {
	Name    string
	Age     string
	Subject string
}

type Tree struct {
	Name   string
	Height int
	Color  string
}

func main() {
	// teacher
	var tea = Teacher{"haiyan", "20", "english"}
	// tree
	var tree = Tree{"Lemon", 170, "green"}

	println(parse2JSON(tea))
	println(parse2JSON(tree))
}

func parse2JSON(i interface{}) string {
	value, _ := json.Marshal(i)
	return string(value)
}
```



### 反射

```go
package main

import (
	"fmt"
	"reflect"
)

type Book struct {
	Name    string
	Author  string
	Page    int
	Summary string
}

// 实现DisplayInfo方法
func (b Book) DisplayInfo() {
	fmt.Println("this is display info method")
}

func main() {
	book := Book{"Harry potter", "JK.罗琳", 300, "All about magic"}
	MirrorOfEntity(book)
}

func MirrorOfEntity(obj interface{}) {

	// 获取类型
	getType := reflect.TypeOf(obj)
	fmt.Println("Get name of type:", getType.Name())

	// 获取值
	value := reflect.ValueOf(obj)
	fmt.Println("Get name of type:", value)

	// 获取实体类里面的每一个字段以及字段的值
	fieldSize := getType.NumField()
	for i := 0; i < fieldSize; i++ {
		field := getType.Field(i)
		fieldValue := value.Field(i).Interface()

		fmt.Println(field.Name, "[", field.Type, "]:", fieldValue)
	}

	// 反射获取反射
	for i := 0; i < getType.NumMethod(); i++ {
		method := getType.Method(i)
		fmt.Println(method.Name, ":", method.Type)
	}
}
```

执行结果

```go
Get name of type: Book
Get name of type: {Harry potter JK.罗琳 300 All about magic}
Name [ string ]: Harry potter
Author [ string ]: JK.罗琳
Page [ int ]: 300
Summary [ string ]: All about magic
DisplayInfo : func(main.Book)
```
