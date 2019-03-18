# Go mysql

数据: 超喜欢 db 的,里面的格式说话又好听.

---

## 1. 安装依赖

```sh
# 下载依赖
root@DESKTOP-77LJV0L MINGW64 /d/docs/mini-docs (dev)
$ go get -u github.com/go-sql-driver/mysql

# 安装依赖
root@DESKTOP-77LJV0L MINGW64 /d/docs/mini-docs (dev)
$ go install  github.com/go-sql-driver/mysql
```

安装完后可以在: `C:\Users\root\go\`找到相应的依赖,把这个路径设置添加为`GOPATH`即可.

---

## 2. 增删改查

数据表结构如下

```sql
DROP TABLE
IF EXISTS rookie_t;

CREATE TABLE rookie_t (
	`id` INT (11) PRIMARY KEY auto_increment COMMENT 'id',
	`code` VARCHAR (64) COMMENT '编码',
	`name` VARCHAR (64) COMMENT '名称',
	`mark` VARCHAR (128) COMMENT '备注'
) ENGINE = INNODB auto_increment = 1 charset = 'utf8' COMMENT 'go rookie go';
```

import 依赖如下

```go
import (
	"database/sql"
	"encoding/json"
	"fmt"
	_ "github.com/go-sql-driver/mysql"
	"strconv"
)
```

实体类如下

```go
type Rookie struct {
	// id
	Id int
	// Name
	Name string
	// Code
	Code string
	// Mark
	Mark string
}
```

### 2.1 通用方法

获取数据库连接

```go
/**
 * 获取数据库连接
 */
func OpenMysql() (db *sql.DB, success bool) {
	mysqlUrl := "root:Root@3306@tcp(47.98.104.252:3306)/spring_rookie?charset=utf8"
	db, err := sql.Open("mysql", mysqlUrl)
	success = true
	if err != nil {
		println("err", err.Error())
		success = false
	}
	return db, success
}
```

打印异常

```go
/**
 * 如果异常不为空,打印异常
 */
func DisplayIfGotAnError(err error) {
	if err != nil {
		fmt.Println(err)
	}
}
```

JSON 转换方法

```go
func ToJSON(universe interface{}) string {
	bytes, _ := json.Marshal(universe)
	return string(bytes)
}
```

### 2.2 新增

使用 PreparedStatement 来操作.

```go
func AddRookie(rookie Rookie) int64 {
	db, success := OpenMysql()
	if !success {
		fmt.Println("Can't connect to db")
		return -1
	}
	// 声明prepared statement
	prepare, err := db.Prepare("insert into rookie_t(`code`,`name`,`mark`) values(?,?,?)")
	DisplayIfGotAnError(err)

	result, err := prepare.Exec(rookie.Code, rookie.Name, rookie.Mark)
	DisplayIfGotAnError(err)

	// 获取执行结果里面的最后插入Id
	lastInsertId, err := result.LastInsertId()
	DisplayIfGotAnError(err)

	return lastInsertId
}
```

测试

```go
func main() {
	rookie := Rookie{0, "haiyan", "haiyan", "top1"}
	lastId := AddRookie(rookie)
	fmt.Println(lastId)
}
```

执行结果

```go
1
```

### 2.3 删除

DB 第一定律: 根据 Id 删除数据.

```go
func DeleteRookie(id int) int64 {
	db, _ := OpenMysql()

	prepare, e := db.Prepare("delete from rookie_t where id = ?")
	DisplayIfGotAnError(e)
	result, e := prepare.Exec(strconv.Itoa(id))
	DisplayIfGotAnError(e)
	affect, e := result.RowsAffected()
	DisplayIfGotAnError(e)

	return affect
}
```

测试

```go
func main() {
	num := DeleteRookie(1)
	fmt.Println(num)
}
```

执行结果

```go
1
```

### 2.4 更新

```go
func DeleteRookie(id int) int64 {
	db, _ := OpenMysql()

	prepare, e := db.Prepare("delete from rookie_t where id = ?")
	DisplayIfGotAnError(e)
	result, e := prepare.Exec(strconv.Itoa(id))
	DisplayIfGotAnError(e)
	affect, e := result.RowsAffected()
	DisplayIfGotAnError(e)

	return affect
}
```

测试

```go
func main() {
	rookie := Rookie{1, "3306", "3306", "top100"}
	num := UpdateRookie(rookie)
	fmt.Println(num)
}

```

执行结果

```go
1
```

### 2.5 查询

```go
func QueryRookie(rookie Rookie) [] Rookie {
	db, _ := OpenMysql()
	prepare, e := db.Prepare("select id,name,code,mark from rookie_t")
	DisplayIfGotAnError(e)

	rows, e := prepare.Query()
	DisplayIfGotAnError(e)

	var rookieList [] Rookie
	for rows.Next() {
		var id int
		var name string
		var code string
		var mark string

		e := rows.Scan(&id, &name, &code, &mark)
		DisplayIfGotAnError(e)

		entity := Rookie{id, name, code, mark}
		rookieList = append(rookieList, entity)
	}
	return rookieList
}
```

测试

```go
func main() {
	rookie := Rookie{1, "3306", "3306", "top100"}
	list := QueryRookie(rookie)
	fmt.Println(ToJSON(list))
}
```

执行结果

```go
[{"Id":1,"Name":"3306","Code":"3306","Mark":"top100"},{"Id":2,"Name":"haiyan","Code":"haiyan","Mark":"top1"}]
```

---

## 3. 参考资料

a. [go 与 mysql 的使用](https://www.jianshu.com/p/71a319c1ff85)
