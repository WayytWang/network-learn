用go的http库能对`http://www.baidu.com`成功的发起请求

```GO
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	resp, err := http.Get("http://www.baidu.com")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()

	c, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return
	}

	fmt.Println(string(c))
}

```

众所周知，HTTP是基于TCP的应用层协议。

应用层往下的协议是操作系统提供的。

http库调用了系统调用来使用操作系统提供的网络功能



tcp靠谱的协议

tcp的窗口控制

tcp的拥塞控制



1.tcp的三次握手与四次挥手





2.操作系统层面的事情

3.go语言发出请求