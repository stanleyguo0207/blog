---
struct相关所有内容
---

## 空结构体

> 空结构体`struct{}`大小为0，作为字段时不占⽤内存，常⽤于Channel信号传递、Map占位符

## 内存对齐

```golang
type Player struct {
	Level int32  // 4字节
	Hp    int64  // 8字节
	Name  string // 16字节（string结构体）
	IsVip bool   // 1字节
}

// 4 + 4(padding) + 8 + 16 + 1 + 7(padding) = 40
```

1. 对齐系数：取类型⾃⾝⼤⼩与系统对⻬系数（64位系统为8）的较⼩值
2. 结构体整体⼤⼩必须是最⼤对⻬系数的整数倍
3. 字段顺序会直接影响结构体内存占用
```golang
type Player struct {
	IsVip bool   // 1字节
	Level int32  // 4字节
	Hp    int64  // 8字节
	Name  string // 16字节（string结构体）
}

// 1 + 4 + 3(padding) + 8 + 16 = 32
```

## 拷贝

1. 结构体是值类型，赋值、传参时会完整拷贝所有字段
2. 大结构体传参优先使用指针

## 比较

> 仅当所有字段均为可⽐较类型时，结构体才可⽐较；包含切⽚、Map、函数等
不可⽐较字段时，结构体不可⽐较，⽆法作为Map的key

## 值接受者 vs 指针接受者

```golang
type Player struct {
    Hp int64
}
// 值接收者：拷⻉结构体副本，修改不影响原对象
func (p Player) AddHpValue(add int64) {
    p.Hp += add
}
// 指针接收者：操作原结构体指针，修改直接⽣效
func (p *Player) AddHpPointer(add int64) {
    p.Hp += add
}
```

1. 值接收者：⽅法内操作的是结构体的完整副本，修改仅对副本⽣效，不影响原对象；适⽤于⼩结构体、只读⽅法、不可变类型
2. 指针接收者：⽅法内操作的是原结构体的指针，修改直接作⽤于原对象；适⽤于⼤结构体、需要修改原对象的场景，也是游戏服务端开发的主流⽤法
3. 接⼝实现规则：类型T的⽅法集仅包含值接收者⽅法；类型*T的⽅法集包含值接收者+指针接收者的所有⽅法。

## 嵌套

```golang
type BaseAttr struct {
	Hp int64
}

type Player1 struct {
	BaseAttr
	Name string
}

type Player2 struct {
	*BaseAttr
	Name string
}
```
下面有四种用法：
```golang
p := Player1{Name: "张三"}
p.Hp = 100
p1 := p
p1.Hp = 200

// 100 200
```
这种要尤其小心，每次都是拷贝，独立的副本
```golang
p := &Player1{Name: "张三"}
p.Hp = 100
p1 := p
p1.Hp = 200

// 200 200
```
```golang
p := Player2{BaseAttr: &BaseAttr{}, Name: "李四"}
p.Hp = 100
p1 := p
p1.Hp = 200

// 200 200
```
```golang
p := &Player2{BaseAttr: &BaseAttr{}, Name: "李四"}
p.Hp = 100
p1 := p
p1.Hp = 200

// 200 200
```

### 同名字段

> 多层嵌套结构体的字段提升仅适⽤于⾮冲突场景，同名字段必须显式指定路径访问

### 方法

> 嵌套结构体的⽅法接收者仅绑定⾃⾝类型，⽆法⾃动绑定到外层结构体，外层结构体调⽤内层⽅法时，操作的是内层结构体实例

### 接口

> 接⼝实现中，外层结构体不会⾃动继承内层结构体的接⼝实现，需显式转发