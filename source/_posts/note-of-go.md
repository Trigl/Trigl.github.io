---
layout:     post
title:      "Go 语法要点"
subtitle:   "个人笔记"
date:       2020-02-14
author:     "Ink Bai"
header-style: "text"
catalog:    true
tags:
    - Go
---

> 最近的工作需要写一个 Go 的项目，所以准备先过一下《Go 语言圣经》这本入门书籍，本篇内容就是一篇摘抄，记录下 Go 不同于其他语言的特性和容易忘记的语法，后面可以直接翻出来使用。

1、main 包下的 main 函数是程序入口，注意 package 必须是 main

```Go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

2、组成程序的函数、变量、常量、类型声明关键字是 func、var、const、type。

3、os.Args 的第一个元素，os.Args[0]，是命令本身的名字，其他的元素才是传给程序的参数，所以想要拿到参数是从 os.Args[1] 开始的。

4、符号 `:=` 是声明并且赋值操作符，但是注意**只能用于函数内部**，不能用于包变量。

5、`fmt.Printf` 常用转换：

![](/img/content/printf.png)

6、Go 语言里的 switch 还可以不带操作对象，switch 不带操作对象时默认用 true 值代替，然后将每个 case 的表达式和 true 值进行比较

```go
func Signum(x int) int {
    switch {
    case x > 0:
        return +1
    default:
        return 0
    case x < 0:
        return -1
    }
}
```

这种形式叫做无 tag switch(tagless switch)

7、Go 中指针是可见的内存地址，&操作符可以返回一个变量的内存地址，并且*操作符可以获取指针指向的变量内容
