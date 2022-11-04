---
typora-root-url: pic
---











tcp服务端

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
		// 处理函数
		go handler(conn)
	}
}

func handler(conn net.Conn) {
	defer conn.Close()
	for {
		reader := bufio.NewReader(conn)
		var buf [128]byte
		n, err := reader.Read(buf[:]) // 读取数据
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



tcp客户端 

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
		if strings.ToUpper(inputInfo) == "Q" { // 如果输入q就退出
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



![](/gotcp代码运行效果.png)

- 1：服务端监听端口
- 2：客户端发起对服务端的TCP连接
- 3：服务端与客户端成功建立连接
- 4：客户端向服务端发送数据
- 5：服务端向客户端发送数据
- 6：客户端向服务端断开连接



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

c462 1f90 360a1d03 00000000 8002 faf0 4cc0 0000 020405b40103030801010402



源端口号（16位） c462 

目标端口号（16位） 1f90 

序号（32位） 360a1d03 

确认序号（32位 00000000 

首部长度（4位） 80

02 

窗口大小（16位） faf0 

校验和（16位） 4cc0 

紧急指针（16位） 0000 

选项 020405b40103030801010402

数据



可以分为几个部分

- Port 端口
  - 源端口 （）
  - 目标端口
- 序号
  - 序号
  - 确认序号
- 状态位
- 窗口大小





序号是为了解决顺序问题，机器A向机器B依次发送了两个包，分别是包1、包2，那么机器A一定是希望机器B知道包1应该是在包2前的。网络环境无法保证先发出去的包就一定会被对方先收到。所以包得有序号。

确认序号是为了解决对方是否收到我发出去的包的问题。机器A向机器B发送了包1，机器B收到后会回复一个包2。但是机器A只是单纯的收到包2，是不知道包2究竟是对包1的确认，还是机器2完全不知道包1的事情，是想有另外一件事想告诉机器A。

这个一会在抓包的分析时更加直观



为什么说TCP是靠谱的协议，机器A在一定时间内没收到机器B对包1的回复，机器A就会觉得对方没收到，它就会重新发包。

所以TCP的可靠不是它下层的IP层保证的，而是自己通过不断尝试去保证的。



为什么一定要建立连接？

机器A想对机器B发起一次HTTP请求，HTTP是基于TCP的，那它们之间得建立一个TCP连接。为什么非要建立连接，直接发不行吗？

tcp是可靠的协议，但是这个可靠是tcp这一层通过序号与确认序号自己做到的。A对B的HTTP请求，

三次握手



























1.tcp的三次握手与四次挥手





2.操作系统层面的事情

3.go语言发出请求