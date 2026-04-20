---
interface相关所有内容
---

## 底层结构

```golang
src/runtime/runtime2.go

// 空接口
type eface struct {
	_type *_type
	data  unsafe.Pointer
}

// 非空接口
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

src/internal/abi/type.go

// 抽象类型
type Type struct {
	Size_           uintptr
	PtrBytes        uintptr 
	Hash            uint32  
	TFlag           TFlag   
	Align_          uint8   
	FieldAlign_     uint8   
	Kind_           Kind    
	Equal           func(unsafe.Pointer, unsafe.Pointer) bool
	GCData          *byte
	Str             NameOff
	PtrToThis       TypeOff
}

src/internal/abi/iface.go

// 动态派发关键
type ITab struct {
	Inter *InterfaceType
	Type  *Type
	Hash  uint32    
	Fun   [1]uintptr // 方法表
}

// 接口类型
type InterfaceType struct {
	Type
	PkgPath Name     
	Methods []Imethod
}
```

## 空接口

> 空接口`interface{}`可以接受任意类型

## 鸭子类型

> go的接口是隐式的，无须显示声明实现，只要类型实现了接⼝的所有⽅法，就⾃动实现了该接⼝，编译期⾃动校验

## 动态派发

> Go的接⼝⽅法调⽤是动态派发的，运⾏时根据动态类型找到对应的⽅法实现，这是接⼝的核⼼机制。
> 
- 动态派发完整流程：
  1. 编译期：校验具体类型是否实现了接⼝的所有⽅法，⽣成对应的类型信息、⽅法表
  2. 赋值时：编译器⽣成代码，构建itab结构体，将接⼝的tab指针指向该itab，data指针指向具的动态值
  3. ⽅法调⽤时：通过itab的fun⽅法表，找到对应⽅法的函数指针，传⼊data指针作为接收者，完成⽅法调⽤
  4. 优化：itab会被缓存，避免重复构建，提升接⼝调⽤性能

## 方法

> 接⼝的⽅法表仅存储接⼝声明的⽅法，不会存储类型的其他⽅法

## nil

```golang
type TestInterface interface {
    Do()
}
type TestStruct struct{}

func (t *TestStruct) Do() {}

func main() {
    var t *TestStruct = nil
    var inter TestInterface = t
    fmt.Println(inter == nil) // 输出：false
}
```

> 接⼝类型与nil相等的充要条件是：tab指针和data指针同时为nil。上述代码中，inter的data指针为nil，但tab指针已经绑定了*TestStruct的类型信息，因此inter != nil。