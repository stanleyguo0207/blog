---
reflect相关所有内容
---

## 核心原理

> Go的反射是在运⾏时动态获取变量的类型信息、值信息，以及调⽤其⽅法的能⼒，核⼼依赖reflect包，底层基于接⼝的eface/iface结构体实现。

## 三大法则

1. 反射可以将接⼝类型变量转换为反射对象（reflect.Type / reflect.Value）
2. 反射可以将反射对象还原为接⼝类型变量
3. 要修改反射对象的值，该值必须是可寻址的（可设置的）

## 修改反射值

> 要通过反射修改变量的值，必须传⼊变量的指针，否则Value对象是不可设置的，调⽤Set⽅法会直接panic。 CanSet() ⽅法可判断是否可设置。

```golang
func main() {
    num := 10
    v := reflect.ValueOf(&num) // 必须传⼊指针，才能修改原值
    v.Elem().SetInt(20) // 通过Elem()解引⽤，设置值
    fmt.Println(num) // 输出：20
}
```

## 限制

1. 反射有极⼤的性能开销，⽐直接调⽤慢1-2个数量级
2. 反射⽆法访问结构体的私有字段（⼩写开头），访问会直接panic
3. 反射调⽤⽅法时，参数必须以[]reflect.Value的形式传⼊，返回值也是[]reflect.Value，类型不匹配会panic