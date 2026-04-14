---
map相关所有内容
---

---
## 2026-04-14

### 底层结构
```golang
src/internal/runtime/maps/map.go

type Map struct {
	used uint64
	seed uintptr
	dirPtr unsafe.Pointer
	dirLen int
	globalDepth uint8
	globalShift uint8
	writing uint8
	tombstonePossible bool
	clearSeq uint64
}
```

#### 1. Map的并发安全问题