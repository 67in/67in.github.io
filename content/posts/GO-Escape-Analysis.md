---
title: "GO Escape Analysis"
date: 2022-06-24T07:09:27+08:00
draft: false
---

## 逃逸分析
>“Quality is never an accident; it is always the result of intelligent effort.” — John Ruskin.

### 逃逸分析是什么
Go编译器的一个阶段，对代码进行逃逸分析，确定其中变量的内存分配位置(栈、堆)，把变量合理地分配到它该去的地方。

逃逸是什么？当一个对象的指针被多个方法或线程引用时，那么这个指针就会发生逃逸。如果一个函数返回对一个变量的引用，那么这个变量就会发生逃逸。

### 逃逸分析作用

- 栈：分配速度快，只需通过PUSH指令，函数return后可自动释放
- 堆：需要找到合适大小的内存块，易形成内存碎片，通过垃圾回收才能释放

通过逃逸分析，可以把一些不需要分配到堆上的变量直接分配到栈上，减轻堆内存分配的开销。GC会定期停止并收集未使用的对象，这也减轻了GC的压力，提高程序的运行速度。

## 如何确定是否发生逃逸

例子如下：

```go
package main

import (
	"fmt"
)

func foo() *int {
	t := 3
	return &t
}

func main() {
	x := foo()
	fmt.Println(*x)
}
```

使用`go build -gcflags '-m -l' xx.go`查看Go编译过程中变量是否发生逃逸。-gcflags参数用于弃用编译器支持的额外标志。如，-m用于输出编译器的优化细节(包括使用逃逸分析这种优化)，相反可以使用-N来关闭编译器优化；而-l则用于禁用foo函数的内联优化，防止逃逸被编译器通过内联彻底地抹除。

```shell
$  go build -gcflags '-m -l' main.go
./escapeAnalysis.go:8:2: moved to heap: t
./escapeAnalysis.go:14:13: ... argument does not escape
./escapeAnalysis.go:14:14: *x escapes to heap
```

可以看到t被移动到堆上，发生了逃逸，*x也发生了逃逸，因为Println()函数的参数类型为interface{}，编译期间很难确定其参数的具体类型，也会发生逃逸。

使用反汇编命令也可以看出变量是否发生逃逸。执行命令：`go tool compile -S main.go`

```shell
"".foo STEXT size=79 args=0x8 locals=0x18 funcid=0x0
        0x0000 00000 (escapeAnalysis.go:7)      TEXT    "".foo(SB), ABIInternal, $24-8
        0x0000 00000 (escapeAnalysis.go:7)      MOVQ    (TLS), CX
        0x0009 00009 (escapeAnalysis.go:7)      CMPQ    SP, 16(CX)
        0x000d 00013 (escapeAnalysis.go:7)      PCDATA  $0, $-2
        0x000d 00013 (escapeAnalysis.go:7)      JLS     72
        0x000f 00015 (escapeAnalysis.go:7)      PCDATA  $0, $-1
        0x000f 00015 (escapeAnalysis.go:7)      SUBQ    $24, SP
        0x0013 00019 (escapeAnalysis.go:7)      MOVQ    BP, 16(SP)
        0x0018 00024 (escapeAnalysis.go:7)      LEAQ    16(SP), BP
        0x001d 00029 (escapeAnalysis.go:7)      FUNCDATA        $0, gclocals·2a5305abe05176240e61b8620e19a815(SB)
        0x001d 00029 (escapeAnalysis.go:7)      FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
        0x001d 00029 (escapeAnalysis.go:8)      LEAQ    type.int(SB), AX
        0x0024 00036 (escapeAnalysis.go:8)      MOVQ    AX, (SP)
        0x0028 00040 (escapeAnalysis.go:8)      PCDATA  $1, $0
        0x0028 00040 (escapeAnalysis.go:8)      CALL    runtime.newobject(SB)
        0x002d 00045 (escapeAnalysis.go:8)      MOVQ    8(SP), AX
        0x0032 00050 (escapeAnalysis.go:8)      MOVQ    $3, (AX)
        0x0039 00057 (escapeAnalysis.go:9)      MOVQ    AX, "".~r0+32(SP)
        0x003e 00062 (escapeAnalysis.go:9)      MOVQ    16(SP), BP
        0x0043 00067 (escapeAnalysis.go:9)      ADDQ    $24, SP
        0x0047 00071 (escapeAnalysis.go:9)      RET
        0x0048 00072 (escapeAnalysis.go:9)      NOP
        0x0048 00072 (escapeAnalysis.go:7)      PCDATA  $1, $-1
        0x0048 00072 (escapeAnalysis.go:7)      PCDATA  $0, $-2
        0x0048 00072 (escapeAnalysis.go:7)      CALL    runtime.morestack_noctxt(SB)
        0x004d 00077 (escapeAnalysis.go:7)      PCDATA  $0, $-1
        0x004d 00077 (escapeAnalysis.go:7)      JMP     0

```

这是部分结果，其中有使用runtime.newobject()函数，它的作用是在堆上分配一块内存，从而说明变量发生了逃逸。

## Go与C/C++中的堆栈概念的区别

C/C++中的堆栈是操作系统级别的概念，它通过编译器所在的环境来决定。

1. 栈：指的是程序运行时自动获得的一小块内存，函数调用消耗的栈的大小，会在编译期间由编译器决定。这块内存用于保存局部变量或者保存函数调用栈。（1MB）
2. 堆：每当程序通过系统调用向操作系统申请内存时，会将所需的空间从维护的堆内存地址空间中分配出去，而在归还是将归还的内存合并到所维护的地址空间中。（1GB）

Go语言中的堆栈与C/C++中的有较大区别。Go语言中的堆栈都指的是Go运行时向操作系统申请的堆内存，被全部用于Go的运行时，维护运行时各个组件的协调。调度器、垃圾回收、系统调用等。所以从理论上来说，相较于只有1MB的C/C++中的栈而言，Go可以拥有1GB的栈内存。

Go指针不能算术运算
Go语言运行时为了防止内存碎片化，会适当对整个栈进行深拷贝，将其整个复制到另一块内存（这个过程对用户态的代码是不可见的），所以在运行过程中无法确定运算前后指针所指向的地址内容是否被运行时移动。也正是这个原因，指针的算术运算不再能生效。

 

