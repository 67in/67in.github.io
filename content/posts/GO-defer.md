---
title: "GO Defer"
date: 2022-07-04T22:18:21+08:00
draft: false
---

## 什么是defer

Go语言的一种用于注册延迟调用的机制，使得函数或语句可以在当前函数执行完毕后执行。

## 为什么需要defer

Go提供的语法糖，减少资源泄露的发生。

## 如何使用defer

在创建资源语句的附近，使用defer语句释放资源。

## defer语句的执行顺序

defer语句不会马上执行，而是会进入一个栈。函数return前，最先定义的defer语句最后执行。

defer函数**定义**时，对外部变量有两种方式：

- 函数参数: 在defer定义时就把值传递给defer，并被cache起来；
- 闭包引用: 在defer函数真正调用时根据整个上下文确定参数**当前的值**

### 函数参数

```go
func main() {
	var whatever [3]struct{}

	for i := range whatever {
		defer func(i int) {
			fmt.Println(i)
		}(i)
	}
}
$ go run main.go
2
1
0
```

### 闭包引用

```go
func main(){
  var whatever [3]struct{}
  
  for i := range whatever{
    defer func(){
      fmt.Println(i)
    }()
  }
}

$ go run main.go
2
2
2
```

defer的执行顺序和定义的顺序相反。

```go
package main

import (
	"fmt"
)

type number int

func (n number) print() {
	fmt.Println(n)
}

func (n *number) pprint() {
	fmt.Println(*n)
}

func main() {
	var n number

	defer n.print()
	defer n.pprint()
	defer func() {
		n.print()
	}()
	defer func() {
		n.pprint()
	}()
	n = 3
}

$ go run main.go
3 //第四个defer语句是闭包，引用外部函数的n，最终结果是3
3 //第三个defer同上
3 //第二个defer语句中的n是引用，最终是3
0 //第一个 是对n直接求值，开始的时候n=0,所以最后是0

```

如果defer像前面介绍的那样简单，这个世界就完美了。但事情总是没这么简单，defer用得不好，会陷入泥潭。其他例子：

`defer` 关键字使用传值的方式传递参数时会进行预计算，导致不符合预期的结果；

```go
func main() {
	startedAt := time.Now()
	defer fmt.Println(time.Since(startedAt))
  
  time.Sleep(time.Second)
}

$go run main.go
168.518µs
```

调用 `defer` 关键字会立刻拷贝函数中引用的外部参数，所以 `time.Since(startedAt)` 的结果不是在 `main` 函数退出之前计算的，而是在 `defer` 关键字调用时计算的，最终导致上述代码输出 0s。

```go
func main() {
	startedAt := time.Now()
	defer func() {
		fmt.Println(time.Since(startedAt))
	}()

	time.Sleep(time.Second)
}

$ go run main.go
1.001001205s
```

### 拆解延迟语句

```go
package main

import (
	"fmt"
)

func f1() (r int) {
	t := 5
  fmt.Println(&t)
	fmt.Println(&r)
	defer func() {
		t = t + 5
	}()
	return t
}

func main() {
	fmt.Println(f1())
}

$ go run main.go
5

```

必须深刻理解`return t`,这句编译之后，实际上生成三条指令：

```shell
1. 返回值 r = t
2. 调用 defer函数，操作的是t
3. 空的return
```

1、3是return生成的指令，return并不是一条原子指令

2 实际上并没有操作返回值r，操作的是t，可以打印出t, r的地址是不一样的。

再看一个例子：

```go
package main

import (
	"fmt"
)

func f() (r int) {
	fmt.Println(&r)
	defer func(r int) {
		fmt.Println(&r)
		r = r + 5
	}(r)
	return 1
}

func main() {
	fmt.Println(f())
}

$ go run main.go
0xc0000160a8
0xc0000160c0
1
```

拆解：

```shell
1. 返回值 r =1
2. 闭包，传参，打印两个r的地址，不是一个r。传值进去的r，是形参的一个复制值，不会影响实参r。
3. 空的return
```

### 确定延迟语句的参数

```go
package main

import (
	"errors"
	"fmt"
)

func e1() {
	var err error

	defer fmt.Println(err)

	err = errors.New("defer1 error")
	return
}

func e2() {
	var err error

	defer func() {
		fmt.Println(err)
	}()

	err = errors.New("defer2 error")
	return
}

func e3() {
	var err error

	defer func() {
		fmt.Println(err)
	}()

	err = errors.New("defer3 error")
	return
}

func main() {
	e1()
	e2()
	e3()
}

//执行e1,e3时候，声明后，传入defer的是nil,然后被defer塞入栈中
//e2闭包使用的是指针，所以会打印defer2 error
$ go run main.go
<nil>
defer2 error
<nil>
```

## 延迟语句配合恢复语句

recover()函数只在defer的函数中直接调用才有效。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	defer fmt.Println("defer main")
	var user = ""

	go func() {
		defer func() {
			fmt.Println("defer caller")
			if err := recover(); err != nil {
				fmt.Println("recover success err:", err)
			}
		}()

		defer func() {
			defer func() {
				fmt.Println("defer here")
			}()

			if user == "" {
				panic("should set user env")
			}

			fmt.Println("after panic")
		}()
	}()

	time.Sleep(1000)
	fmt.Println("end of main function")
}

$ go run main.go
end of main function
defer main
defer here
defer caller
recover success err: should set user env
```

panic会停掉当前正在执行的程序，而不只是当前线程。在这之前，它会有序地执行完当前线程defer列表里的语句，其他协程里定义的defer语句不做保证。所以在defer里定义一个recover语句，防止程序直接挂掉，就可以起到try...catch的效果。

## 其他

### Go调度模型

三个实体：M(Machine)、P(Processor)、G(Goroutine)

G: Go运行时对goroutine的描述，G中存放并发执行的代码入口地址、上下文、运行环境(关联的P和M)、运行栈等执行相关的信息。

M: OS内核线程，是操作系统层面调度和执行的实体。直接关联一个内核级线程。

P: 代表M和G所需要的资源，是对资源的一种抽象管理，P不是一段代码实体，而是一个管理的数据结构，P主要是降低M对G的复杂性，增加一个间接的控制层数据结构。P控制GO代码的并行度，它不是实体。

### 闭包

闭包 = 函数 + 引用环境， 也称匿名函数，不能独立存在，但可以直接调用或赋值于某个变量。

不太恰当的例子： 可以把闭包看成是一个类，闭包函数的一个调用就是实例化一个类。闭包在运行时可以有多个实例，它会将同一个作用域的变量和常量捕获，无论闭包在什么地方调用，都可以使用这些变量和常量。闭包捕获的变量和常量都是引用传递，不是值传递。

```go
package main

import (
	"fmt"
)

func main() {
	var a = Accumulator()

	fmt.Printf("%d\n", a(1))
	fmt.Printf("%d\n", a(10))
	fmt.Printf("%d\n", a(100))

	fmt.Println("--------------------------------")

	var b = Accumulator()

	fmt.Printf("%d\n", b(1))
	fmt.Printf("%d\n", b(10))
	fmt.Printf("%d\n", b(100))
}

func Accumulator() func(int) int {
	var x int
	return func(delta int) int {
		fmt.Printf("(%+v, %+v) - ", &x, x)
		x += delta
		return x
	}
}
```

闭包已用了x变量，a,b可看作2个不同的实例，实例之间互不影响，实例内部，x变量是同一个地址，因此具有"累加效应"

