---
typora-root-url: pic
---



从代码入手看看TCP协议如何用

抓包分析分析TCP有啥神奇之处



Go写网络服务的优势在哪里？



大纲：

go tcp代码

抓包分析过程

go的网络io模型



从代码开始

代码分为服务端和客户端

## 代码

### 服务端

首先使用TCP协议监听8080端口，启动一个网络服务。

```go
listen, err := net.Listen("tcp", ":8080")
if err != nil {
	panic(err)
}
fmt.Println("server started")
```

然后等待客户端对服务端发起连接

```go
for {
  conn, err := listen.Accept()
	if err != nil {
			fmt.Println("accept failed,err:", err)
			continue
	}		
	fmt.Println("server accept new connection")
	// 处理客户端连接
	go handler(conn)		
}

```

- `listen.Accept()`会等待客户端的连接建立成功，得到的conn就是一个已经建立好的连接
- 接下来使用开启协程调用handler函数来处理这个连接里面发生的事情
- 最外层的for循环是为了能接受多个客户端的连接

handler的逻辑：

```go
func handler(conn net.Conn) {
	defer conn.Close()
	for {
		reader := bufio.NewReader(conn)
		var buf [128]byte
    // 读取客户端发送的数据
		n, err := reader.Read(buf[:]) 
		if err != nil {
			fmt.Println("read from client failed, err:", err)
			break
		}
		recvStr := string(buf[:n])
		fmt.Println("收到client发来的消息:", recvStr)
		// 向客户端发送数据
		conn.Write([]byte("your message is" + recvStr))
	}
}
```

- handler内有一个for循环
- 循环内会从conn读取数据，这个数据是从客户端发送过来的
- 然后会讲客户端发送来的数据加上`your message is `后再发送给客户端
  - `conn.Write`

服务端的完整代码

```go
package main

import (
	"bufio"
	"fmt"
	"net"
)

func main() {
	listen, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}
	fmt.Println("server started")
	for {
		conn, err := listen.Accept()
		if err != nil {
			fmt.Println("accept failed,err:", err)
			continue
		}
		fmt.Println("server accept new connection")
		// 处理客户端连接
		go handler(conn)
	}
}

func handler(conn net.Conn) {
	defer conn.Close()
	for {
		reader := bufio.NewReader(conn)
		var buf [128]byte
    // 读取数据
		n, err := reader.Read(buf[:]) 
		if err != nil {
			fmt.Println("read from client failed, err:", err)
			break
		}
		recvStr := string(buf[:n])
		fmt.Println("收到client发来的消息:", recvStr)
		// 发送数据
		conn.Write([]byte("your message is" + recvStr))
	}
}

```

### 客户端

客户端首先要与服务端建立连接

```go
conn, err := net.Dial("tcp", "xxx.xxx.xxx.xxx:8080")
	if err != nil {
		panic(err)
	}
fmt.Println("client connected")
```

然后就开始与服务端交互

```go
	inputReader := bufio.NewReader(os.Stdin)
	for {
		// 读取终端输入
		input, _ := inputReader.ReadString('\n')
		inputInfo := strings.Trim(input, "\r\n")
    // 如果输入q就退出
		if strings.ToUpper(inputInfo) == "Q" { 
			// 关闭连接
			conn.Close()
			return
		}
		// 向服务端发送数据
		_, err = conn.Write([]byte(input))
		if err != nil {
			return
		}
		buf := [512]byte{}
		// 读取服务端发来的数据
		n, err := conn.Read(buf[:])
		if err != nil {
			fmt.Println("recv failed, err:", err)
			return
		}
		fmt.Println("收到server发来的消息:" + string(buf[:n]))
	}
```

- 交互过程就是：
  - 用户从终端输入需要交互的内容
  - 客户端程序读到内容后将终端输入的内容发送给服务端
    - 当终端内容为`Q`时，客户端主动断开与服务度的连接
      - `conn.Close()`
  - 客户端也会从`conn`中读取服务端的数据，并打印在终端上

客户端的完整代码：

```GO
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strings"
)

func main() {
	conn, err := net.Dial("tcp", "xxx.xxx.xxx.xxx:8080")
	if err != nil {
		panic(err)
	}
	fmt.Println("client connected")

	inputReader := bufio.NewReader(os.Stdin)
	for {
		// 读取终端输入
		input, _ := inputReader.ReadString('\n')
		inputInfo := strings.Trim(input, "\r\n")
    // 如果输入q就退出
		if strings.ToUpper(inputInfo) == "Q" { 
			// 关闭连接
			conn.Close()
			return
		}
		// 发送数据
		_, err = conn.Write([]byte(input))
		if err != nil {
			return
		}
		buf := [512]byte{}
		// 读取服务端发来的数据
		n, err := conn.Read(buf[:])
		if err != nil {
			fmt.Println("recv failed, err:", err)
			return
		}
		fmt.Println("收到server发来的消息:" + string(buf[:n]))
	}
}

```



### 执行效果

![](/gotcp代码运行效果.png)

- 第1步：服务端监听端口
  - `net.Liste()`

- 第2步：客户端发起对服务端的TCP连接
  - `net.Dail()`

- 第3步：服务端与客户端成功建立连接
  - `net.Accept()`

- 第4步：客户端向服务端发送数据
  - 客户端：`conn.Write()`

- 第5步：服务端向客户端发送数据
  - 服务端：`conn.Write()`

- 第6步：客户端向服务端发起断开连接
  - 客户端：`conn.Close()`




## 抓包看看TCP协议的特点

这一节会抓包分析执行效果中的6个步骤，网络底层发了那些网络包

### TCP协议

在这之前，先聊一聊TCP有什么特点。在网上搜到的TCP权威的定义：

`传输控制协议是一种面向连接的、可靠的、基于字节流的传输层通信协议，由IETF的RFC 793定义。`

那这里就有三个重点，也是问题：

- 面向连接？
- 为什么可靠？
- 什么是基于字节流？



从`可靠`开始

TCP处理网络分层的传输层，它会把包往下传，IP层、MAC层以及物理层，会将网络包发送到目的地。但是它们并不保证对方会收到，网络不好的情况下，丢包是很常见的。

TCP如果只是简单的依靠往下的网络层，那它就不能说是可靠的，都丢包了还可靠个鸡毛啊。而且网络环境中，还有种情况：

- A向B先后发送了两个包：包1、包2，如果两个包是有顺序的，那么A一定是希望B知道包1是在包2前面的。但是B可不一定先收到哪个包。

为了上诉的两个解决问题，TCP给包加了两个序号：序号（Seq）与确认序号（Ack）

- 序号可以解决顺序问题
- 确认序号用来解决丢包问题
  - A向B发送了包1，这是带着序号的，比如Seq=1，



序号是为了解决顺序问题，那么机器A一定是希望机器B知道包1应该是在包2前的。网络环境无法保证先发出去的包就一定会被对方先收到。所以包得有序号。

确认序号是为了解决对方是否收到我发出去的包的问题。机器A向机器B发送了包1，机器B收到后会回复一个包2。但是机器A只是单纯的收到包2，是不知道包2究竟是对包1的确认，还是机器2完全不知道包1的事情，是想有另外一件事想告诉机器A。

这个一会在抓包的分析时更加直观



为什么说TCP是靠谱的协议，机器A在一定时间内没收到机器B对包1的回复，机器A就会觉得对方没收到，它就会重新发包。

所以TCP的可靠不是它下层的IP层保证的，而是自己通过不断尝试去保证的。



为什么一定要建立连接？

机器A想对机器B发起一次HTTP请求，HTTP是基于TCP的，那它们之间得建立一个TCP连接。为什么非要建立连接，直接发不行吗？

tcp是可靠的协议，但是这个可靠是tcp这一层通过序号与确认序号自己做到的。A对B的HTTP请求，

三次握手





### TCP包头

### 建立连接：三次握手

### 断开连接：四次挥手



应用层往下的协议是操作系统提供的。

http库调用了系统调用来使用操作系统提供的网络功能



从代码开始，到抓包结束

Socket







TCP是一个靠谱的协议

tcp靠谱的协议

tcp的窗口控制

tcp的拥塞控制





tcp包头

 ![](/tcp包头截图.png)



 ![](/tcp包头格式.png)






源端口号（16位） cb cc

目标端口号（16位） 1f 90

序号（32位） ab ef d2 21

确认序号（32位）  2b 32 09 e5

首部长度（4位）  50     18

窗口大小（16位）04 05

校验和（16位） ce fc 

紧急指针（16位）  00 00 

数据  68 65 6c 6c 6f 0d 0a



可以分为几个部分

- Port 端口
  - 源端口 （）
  - 目标端口
- 序号
  - 序号
  - 确认序号
- 状态位
- 窗口大小











## Go的IO模型

又回到go

1.tcp的三次握手与四次挥手





2.操作系统层面的事情

3.go语言发出请求