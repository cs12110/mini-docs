# Go 的多线程

如果没有多线程,这个世界的雨,是一滴,一滴下的.

---

## 1. Hello world

万事皆可 hello world.

```go
package main

import (
	"runtime"
	"time"
)

func main() {

	runtime.GOMAXPROCS(3)

	go run("a")
	go run("b")
	go run("c")

	// 但这里为什么tmd要休眠呀???
	time.Sleep(10 * time.Second)
}

func run(something string) {
	for i := 0; i < 2; i++ {
		println(something)
	}
}
```

执行结果

```go
a
a
c
c
b
b
```

Q: 但这里为什么 tmd 要休眠呀???

A: 用来阻止主线程关闭的,如果没有 sleep 一会的话,会发现程序瞬间就结束了,而且什么都没有输出.这是因为主线程关闭之后,所有开启的 goroutine 都会强制关闭,还没有来得及输出就结束了(一如我那 18 岁的初恋???).

Q: 那该怎么变得优雅一点呀?

A: 用 channel,用 channel.

---

## 2. channel

`channel`:goroutine 之间通过 channel 来通讯,可以认为 channel 是一个管道或者先进先出的队列.可以从一个 goroutine 中向 channel 发送数据,在另一个 goroutine 中取出这个值(类似于 java 里面的`CountdownLatch`).

创建方式为: `ch := make(chan int)`

```go
package main

import (
	"runtime"
)

func main() {

	runtime.GOMAXPROCS(3)

	flg1 := make(chan int)

	go run("a", flg1)
	go run("b", flg1)
	go run("c", flg1)

	// 注意channel类似队列
	println(<-flg1)
	println(<-flg1)
	println(<-flg1)
}

func run(something string, f chan int) {
	for i := 0; i < 2; i++ {
		println(something)
	}
	f <- 1
}
```

执行结果

```go
b
b
1
c
c
a
a
1
1
```
