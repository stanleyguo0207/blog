---
defer相关所有内容
---

## 核心特性

- 作⽤：注册延迟函数，在**当前函数return之前**、**panic退出之前**执⾏，常⽤于资源释放、锁释放、
recover捕获panic
- 执⾏顺序：后进先出LIFO，多个defer按注册顺序倒序执⾏
- 参数求值：defer注册函数时，**参数会⽴即求值**，不会等到执⾏时计算
- 闭包特性：defer注册的闭包函数，引⽤的外部变量是延迟绑定，执⾏时才取变量的当前值

## 执行顺序（LIFO）

```golang
func test() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
// 输出：3 2 1
```

## 传参

```golang
func test() {
    i := 0
    defer fmt.Println(i) // 注册时i=0，⽴即求值，输出0
    i++
    return
}
// 输出：0
```

## 闭包延迟绑定

```golang
func test() {
    i := 0
    defer func() { fmt.Println(i) }() // 闭包引⽤i，执⾏时i=1，输出1
    i++
    return
}
// 输出：1
```

## 命名返回值

```golang
func test() (res int) {
    defer func() { res++ }()
    return 0
}
// 输出：1
// 原理：return 0 先给res赋值0，执⾏defer后res变为1，最终返回res
```

## recover

### 生效条件

1. 必须在`defer`函数内部直接调用`recover()`
2. `defer`必须在`panic`发生之前注册
3. 当前`goroutine`正处于`panic`状态

### 多次recover

```golang
func main() {
	defer func() {
		fmt.Println(recover()) // error
		fmt.Println(recover()) // nil
	}()
	panic("error")
}
```

### 子goroutine

只能捕获当前goroutine的panic，这种情况无效
```golang
func main() {
	defer func() { recover() }()
	go func() {
		panic("error")
	}()
	time.Sleep(time.Second)
}
```

### 嵌套panic

后注册的defer先执行，新panic覆盖旧panic
```golang
func main() {
	defer func() {
		fmt.Println("1:", recover())
	}()

	defer func() {
		panic("panic2")
	}()

	panic("panic1")
}

// 1:panic2
```

### 无法恢复

> Go 的 recover() **只能捕获用户态 panic() 触发**、运行时普通 runtime panic；所有 runtime 底层致命致命错误（runtime.throw/fatal）、协程强制退出、系统信号、内存级致命异常、跨协程 panic、用法错误场景，全部无法恢复，直接进程终止或协程直接消亡

1. 普通 panic（可 recover）
	由代码 panic(x)、语言层面运行时错误触发（空指针、数组越界、关闭已关 channel 等）。
	
	流程：触发 panic → 逆序执行当前 goroutine 所有 defer → defer 内 recover() 可拦截，恢复协程执行。
2. 致命错误 fatal error（不可 recover）
	底层运行时 runtime.throw() / runtime.fatal() 直接触发，完全不走 panic 栈、不执行 defer、不触发 recover，直接打印崩溃信息并终止整个进程。

#### 常见场景

1. 运行时底层致命 fatal error

	1. map并发读写
	2. 栈溢出
	3. 内存非法访问 & unsafe 底层内存损坏
		- 野指针、非法内存地址解引用、unsafe 包越界内存操作
		- CGO 调用 C 代码导致的内存越界、段错误，底层直接触发系统级异常 + runtime fatal，recover 完全无效。

2. runtime 强制协程退出 runtime.Goexit()

3. 操作系统原生信号（非 Go 转换 panic）

	系统发送的硬中断信号，Go 运行时不会转为 Go panic，recover 无感知：
	- SIGSEGV 段错误（非法内存访问）
	- SIGABRT 进程异常终止
	- SIGKILL 系统强制杀死进程（无法捕获、无法处理）
	- SIGBUS 内存总线错误
	
	仅 SIGINT（Ctrl+C）这类信号，你可以用 signal 包注册回调手动触发 panic，才能被 recover 捕获；原生系统致命信号直接杀进程。

4. 进程级资源耗尽崩溃	

	- OOM 内存耗尽：系统内存耗尽，操作系统直接杀掉整个 Go 进程，Go 运行时本身已瘫痪，recover 无任何作用
	- 运行时调度器、GC 底层致命故障、内存分配器损坏，整个运行时环境失效，无 recover 执行上下文。

5. 跨 Goroutine 的 panic（协程隔离，天生无法捕获）

6. recover 自身用法错误（逻辑上无法恢复，非底层 bug）

7. 可以recover的常见panic

	- 手动代码 panic("自定义错误")
	- 空指针解引用 (*int)(nil)
	- 数组 / 切片下标越界
	- 关闭已关闭的 channel、向关闭 channel 发送数据
	- nil map 赋值 var m map[int]int; m[1]=1
	- 类型断言失败 x.(int) 类型不匹配
	- 整数除零错误