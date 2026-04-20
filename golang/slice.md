---
slice相关所有内容
---

## 底层结构

```golang
src/runtime/slice.go

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

## 1. 内存

> 64位系统下占用24字节。

## 2. 扩容 (release-branch.go1.26)

- 扩充前容量小于256，翻倍
- 否则`newcap += (newcap + 3*threshold) >> 2`，近似1.25倍扩容
- 无论哪种扩容，最终会被**内存对齐**
```golang
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
		newcap += (newcap + 3*threshold) >> 2

		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

下面有四个测试代码：

```golang
代码1
s1 := make([]int32, 256)
s1 = append(s1, 1)

len = 257 cap = 512
```

1. nextslicecap计算：
oldCap=256（≥256 阈值），doublecap=512，257<512，进入 1.25 倍公式：
newcap = 256 + (256+768)>>2 = 256+256 = 512。
1. 内存对齐：
512×4（int32 字节）=2048 字节，刚好匹配 Go 内存规格，无需对齐。
1. 结果：len=257，cap=512。

```golang
代码2
s1 := make([]int32, 100)
s1 = append(s1, 1)

len = 101 cap = 224
```

1. nextslicecap计算：
oldCap=100（<256），直接返回 doublecap=200。
2. 内存对齐：
200×4=800 字节，Go 内存规格无 800 字节，向上对齐到 896 字节，896÷4=224。
3. 结果：len=101，cap=224。


```golang
代码3
s1 := make([]int32, 200)
s1 = append(s1, make([]int32, 300)...)

len = 500 cap = 512
```

1. nextslicecap计算：
doublecap=400，500>400，直接返回 newLen=500。
2. 内存对齐：
500×4=2000 字节，向上对齐到 2048 字节，2048÷4=512。
1. 结果：len=500，cap=512。

```golang
代码4
s1 := make([]int32, 300)
s1 = append(s1, make([]int32, 200)...)

len = 500 cap = 576
```

1. nextslicecap计算：
oldCap=300（≥256），doublecap=600，500<600，进入 1.25 倍公式：
newcap = 300 + (300+768)>>2 = 300+267 = 567；567<500 不成立，停止循环。
2. 内存对齐：
567×4=2268 字节，向上对齐到 2304 字节，2304÷4=576。
1. 结果：len=500，cap=576。

- 字节对齐表 `src/internal/runtime/gc/sizeclasses.go`

## 3. 赋值

记住 **切片的赋值只是结构的拷贝!**
- 赋值的结构体仅为结构体本身，底层数组指针不变，除非有扩容操作
- 函数传参是值传递，与赋值一样

下面有四个测试代码：

```golang
代码1
s1 := []int32{1, 2, 3, 4}
s2 := s1
s2[0] = 8
s2 = append(s2, 5)
s2[0] = 9

// s1 array = [8,2,3,4] len = 4 cap = 4
// s2 array = [9,2,3,4] len = 5 cap = 8
```

```golang
代码2
s1 := make([]int32, 0, 10)
s1 = append(s1, []int32{1, 2, 3, 4}...)
s2 := s1
s2[0] = 8
s2 = append(s2, 5)
s2[0] = 9

// s1 array = [9,2,3,4] len = 4 cap = 10
// s2 array = [9,2,3,4,5] len = 5 cap = 10
```

```golang
代码3
s1 := []int32{1, 2, 3, 4}
func(s []int32) {
    s[0] = 8
    s = append(s, 5)
    s[0] = 9

    // s array = [9,2,3,4,5] len = 5 cap = 8
}(s1)

// s1 array = [8,2,3,4] len = 4 cap = 4
```

```golang
代码4
s1 := make([]int32, 0, 10)
s1 = append(s1, []int32{1, 2, 3, 4}...)
func(s []int32) {
    s[0] = 8
    s = append(s, 5)
    s[0] = 9

    // s array = [9,2,3,4,5] len = 5 cap = 10
}(s1)

// s1 array = [9,2,3,4] len = 4 cap = 10
```

- 赋值或者传参后，如果发生append操作，原切片的实际值将不可预测
- 如果需要函数或者赋值后改变元切片，需要将改变后的切片重新赋值给原始切片

## 4. 拷贝（copy）

**拷贝不会扩容**
- copy把数据从一个切片拷贝到另一个切片，只拷贝len长度的元素，完全不关心cap
- `copy(dst, src)`，拷⻉⻓度为 `min(dst.len, src.len)`

下面有两个测试代码：

```golang
代码1
s1 := []int32{1, 2, 3, 4}
s2 := make([]int32, 0, 8)
s2 = append(s2, []int32{-1, -1, -1, -1, -1, -1, -1, -1}...)
copy(s2, s1)

// s1 array = [1,2,3,4] len = 4 cap = 4
// s2 array = [1,2,3,4,-1,-1,-1,-1] len = 8 cap = 8
```

```golang
代码2
s1 := []int32{1, 2, 3, 4, 5, 6, 7, 8}
s2 := make([]int32, 0, 4)
s2 = append(s2, []int32{-1, -1, -1, -1}...)
copy(s2, s1)

// s1 array = [1,2,3,4,5,6,7,8] len = 8 cap = 8
// s2 array = [1,2,3,4] len = 4 cap = 4
```

## 5. 遍历

5.1. 历史争议问题 `range中的v`

```golang
package main

import "fmt"

func main() {
	slice := []int{10, 20, 30}
	var res []*int

	for _, v := range slice {
		res = append(res, &v)
	}

	fmt.Println(*res[0], *res[1], *res[2])
}

// go1.21.0 结果 30 30 30
// go1.26.0 结果 10 20 30
```

- 该问题在`go1.22.0`中修复，https://go.dev/blog/loopvar-preview
- `go1.22.0+`之后，v不再公用一个变量，每次循环使用单独的变量

5.2. 循环`append`

```golang
s1 := []int32{1, 2, 3, 4}
for _, v := range s1 {
    s1 = append(s1, v)
}

// s1 [1,2,3,4,1,2,3,4]
```

- 在range初始化的时候就已经确定了len，不会取最新的len

5.3. `v`到底会不会改变

```golang
代码1
s1 := []int32{1, 2, 3, 4}
for _, v := range s1 {
	v += 1
}

// s1 [1,2,3,4]
```

```golang
代码2
type Data struct {
	Value int32
}

s1 := []Data{{1}, {2}, {3}, {4}}
for _, v := range s1 {
	v.Value += 1
}

// s1 [{1},{2},{3},{4}]
```

```golang
代码3
type Data struct {
	Value int32
}

s1 := []*Data{{1}, {2}, {3}, {4}}
for _, v := range s1 {
	v.Value += 1
}

// s1 [{2},{3},{4},{5}]
```

- 虽然v都是临时变量（从代码1，2的执行结果很容易看出）,但是代码3的不一样的地方时，临时变量指向的是指针，所以可以修改指针指向的值

## 6. nil与空切片

`var s1 []int32` 与 `s2 := []int32{}` 不是一回事！

- s1与s2的len，cap都是0
- s1的array是nil，s2的array指向空数组
- json序列化结果不同，s1序列化为`null`，s2序列化为`[]`