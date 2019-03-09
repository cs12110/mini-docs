# Go basic

于是,go 在这里开始了.

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

### list

### map

---

## 常规应用

### 继承

```go

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

### 反射

### IO

### 多线程
