---
string相关所有内容
---

## 底层结构

> Go的字符串是不可变的字节序列，底层是⼀个只读的字节数组

```golang
src/runtime/string.go

type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

## 1. 核心知识点

1. 字符串是值类型，64位系统下占⽤16字节，赋值仅拷⻉结构体，不拷⻉底层字节数组
2. 字符串不可变性：底层字节数组是只读的，禁⽌直接修改字符串的单个字符，修改操作会⽣成新的
字符串
3. 字符串的len()是O(1)操作，直接读取结构体的len字段，⽆需遍历
4. Go字符串默认采⽤UTF-8编码，中⽂字符占⽤3个字节，len()返回的是字节数，不是字符数，遍历需注意

## 2. 拼接

- `strings.Builder`
    > 最快
- `bytes.Buffer`
- `+`
- `fmt.Sprintf`
    > 性能最差

## 3. 遍历

- for索引遍历取的是**单个字节**
- for range遍历会**⾃动解码UTF-8字符**，遍历中⽂字符必须⽤for range

## 4. 空字符串

空字符串`""`的底层str指针不为nil，len为0，与nil不相等，判空必须⽤`len(s) == 0`，不要⽤`s == ""`以外的⽅式