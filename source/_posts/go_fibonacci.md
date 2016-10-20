---
title: 初涉Go实现斐波纳契数
tags:
  - Golang
categories:
  - Golang
id: 204
date: 2016-05-30 16:31:05
---

初涉Go语言，感到有些别扭，毕竟Java写多了，连定义函数的类型都写在后面，简直别扭的不要不要的，更没想都Go的函数这么灵活，可以当做返回值，参数和类型，当然也让我充满了好奇。

### 递归实现

```go
package main

import "fmt"

func fibonacci(x int) int {
	if x < 2 {
		return 1
	}
	return fibonacci(x-1) + fibonacci(x-2)
}

func main() {
	for i := 0; i < 10; i++ {
		fmt.Println(fibonacci(i))
	}
}
```

### 闭包实现

```go
package main

import "fmt"

func fibonacci() func() int {
	a,b := 0,1;
	return func() int{
		a,b = b,a+b
		return a;
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```


