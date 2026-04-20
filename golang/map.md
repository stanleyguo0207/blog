---
map相关所有内容
---

## 底层结构
```golang
src/internal/runtime/maps/map.go

type Map struct {
	used              uint64
	seed              uintptr
	dirPtr            unsafe.Pointer
	dirLen            int
	globalDepth       uint8
	globalShift       uint8
	writing           uint8
	tombstonePossible bool
	clearSeq          uint64
}
```

## 1. Map的并发安全问题

> Go 原生 Map 非并发安全，并发读写（至少一个写操作）会直接触发 fatal error，而非 panic，无法通过 recover 捕获

1. 读多写少
	> 使用`sync.Map`
2. 读写均衡/写多读少
	> 使用`sync.Mutex` + `map`
3. 单goroutine
	> 使用`map`

## 2. Map的Value

> Map的Value是不可寻址的

修改map的value必须先把value取出来，修改完后，再赋值回去

## 3. 遍历

> Map的遍历是显式随机的

## 4. 删除

> 删除元素不会立即释放内存，仅标记为墓碑，仅在扩容时才会清理墓碑槽位；调用 clear () 会清空所有元素，重置哈希种子，释放内存