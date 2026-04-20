---
channel相关所有内容
---

## 核心基础

> Channel是Goroutine之间的通信原语，Go的核⼼设计理念：**不要通过共享内存通信，要通过通信共享内存**。

## 底层结构

```golang
type hchan struct {
	qcount   uint
	dataqsiz uint
	buf      unsafe.Pointer
	elemsize uint16
	closed   uint32
	timer    *timer
	elemtype *_type
	sendx    uint
	recvx    uint
	recvq    waitq
	sendq    waitq
	bubble   *synctestBubble
	lock     mutex
}
```

## 类型

- 无缓冲Channel
    > 发送和接收必须配对，否则会阻塞，常⽤于Goroutine同步
- 有缓冲Channel
    > 缓冲区未满时，发送不阻塞；缓冲区⾮空时，接收不阻塞，常⽤于限流、消息队列
- 单向Channel（只读<-chan、只写chan<-）