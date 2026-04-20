---
iota相关所有内容
---

## 基本定义

> `iota` 是 Go 语言预定义的常量计数器，只能用在常量块 const 里面。

- 默认从 0 开始
- 每新增一行常量定义，数值 自动 +1
- 只在同一个 const 块内生效，不同 const 块互不干扰

## 基础规则

- iota 初始值 = 0
- const 内部每换行一次，iota 自增 1
- 同一行多个常量，共享同一个 iota 值
- 常量表达式只写一次，后面行会继承表达式格式，只替换 iota

## 几种典型情况

1. 同一行多个常量，共享同一个 iota 值

```golang
const (
    a, b = iota, iota+1 // a=0, b=1
    c, d                // c=1, d=2
)
```

2. 继承表达式

```golang
const (
    a = 1 << iota   // 1 << 0 = 1
    b               // 1 << 1 = 2
    c               // 1 << 2 = 4
    d               // 1 << 3 = 8
)
```

3. 只要换行就加

```golang
const (
    a = 100  // iota=0
    b        // iota=1，值继承100
    c = iota // 这里才用iota，值=2
)
```

```golang
const (
    a = iota   // 0
    b = 5      // 固定值5，iota此时=1，但没用上
    c          // 继承上一行=5，iota=2
    d = iota   // 3
)
```

```golang
const (
    i = iota
    j
    k = iota * 2
    l
)
// i=0 j=1 k=4 l=6
```