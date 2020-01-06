---
title: 基于Websocket的消息传输系统
author: dashjay
date: '2019-12-03'
slug: websocket
categories:
  - golang
  - Websokcet
tags: []
---


丑话说前面，本人就是LJ本科生，水平没多高，大多数代码源自其他项目，比如一个叫leaf的游戏框架。

[leafgithub.com](https://link.zhihu.com/?target=https%3A//github.com/name5566/leaf)

但是作为普通的穷苦人民，在上面催你的时候，很难静下新来，慢慢的打开vscode去一行一行的看，比如你现在可能就是在火急火燎的想在知乎上寻求一个解决方案。

先说一下我的使用场景。我的场景是这样：

用户会登陆软件后，需要一些认证方法，连接上socket，然后会进行高速传输，比如消息、群消息、交易拍卖、其他等等一系列消息，主体就是用户，客体是其他用户或者服务器。

## **架构**

我现对整个项目架构描述一下

![img](https://pic3.zhimg.com/80/v2-f064e3332e066c5ec3a41331ffe0e302_hd.jpg)

先简述一下Hub的设计，总的一个Hub包含Client这样一个map用来储存连接信息，然后每个连接有一个Reader和一个Write来从客户端读取和写入客户端。

按照图中标志，按实际执行顺序来讲

①. 客户端向服务端发起Socket连接，（服务器识别到这是Socket连接之后有个Upgrade操作），这里不太明白的可以去看看Socket连接的过程，并不复杂。

③. Reader收到的第一条消息，是注册消息，一般又客户端OnConnect的时候发送，一般会携带认证信息进行身份认证。

②. 认证成功之后服务器应该返回用户类似于JWT或者SessionCookie类的东西，或者没有也没关系，因为每次重新创建连接都要认证，我用JWT是为了做访问控制。

> 如果没有其他异常：就会一直处在①、②、①、②......这样的收发状态。直到.......

④. 注销，一旦客户端退出或者网络中断等异常，就会出现断开。

好了讲了整个东西架构，让我们来看看怎么实现的把，showcode时间到了。

## **实现**

首先，让我们来看看hub的结构吧，说结构还真的就是结构

```go
// 整个项目的交流系统，由clients和各种不同的chan组成，
type Hub struct {
	Clients      map[*Client]bool   // 储存保持连接的用户
	UserToClient map[string]*Client // 获得用户和其连接的对应关系
	Message      chan []byte        // 广播消息队列
	Register     chan *Client       // 发起注册请求的队列
	Unregister   chan *Client       // 断开连接的队列
}
```

注释讲的比较清楚了，我中间使用UserToClient这个map来方便找到我们的Client而已，

> 注意，Client我们存的是指针，因此在这里并没有冗余，我只是为了方便找用户，而且不用再遍历。

这里大家有没有注意到一个好玩的东西叫做`chan` ，这个东西在C/C++里叫做pipe，但是我个人觉得在go里爽的不止那么一点点，有这个东西，我的思路清晰多了。

> 我在这里插入一个操作系统中生产者消费者的例子，如果觉得没啥好玩的可以跳过。

```go
package main

import (
	"fmt"
	"time"
)

func produce(p chan<- int) {
	for i := 0; i < 1000; i++ {
		p <- i
		fmt.Println("send:", i)
	}
}
func consumer(c <-chan int) {
	for i := 0; i < 1000; i++ {
		v := <-c
		fmt.Println("receive:", v)
	}
}
func main() {
	ch := make(chan int, 5)
	go produce(ch)
	go consumer(ch)
	time.Sleep(10000 * time.Second)
}
```

就这么个东西，golang 给我们提供了一个类似于管道的东西，目的是进行进程间通信。我们先假装自己不知道goroutine这个东西，我们就可以认为 go produce(ch) 就相当于手动执行了一个程序，传入了我们事先 make 的一个长度（缓冲长度）为 5 的chan `ch :=make(chanint,5)` 这个管道有点像。

你还没放学，你妈妈给你做了5个烧饼，你回来就吃了一个 ，然后妈妈问你还要不要，你说要，然后管道里又多了一个生产者生产的烧饼，就是这样一个过程，说白了有点像个队列，具体我就没有看了。

============操作系统课结束=============

那么我们在本次项目当中也会使用这种 `chan` ，功能就是这样：

> 再补充一个关键词 阻塞（block） 就是当程序执行到 v <- chan 然后这个 chan 里没东西的时候，程序就会阻塞。那么好我们来看一下怎么用上面的那个Hub的结构。

忘了说，这个Hub是单例，也就是说整个代码里只运行一份。因此一个Hub要写完整个项目的逻辑，那么我就从hub的创建开始

```go
func NewHub() *Hub {
	return &Hub{
		Message:      make(chan []byte),        // 消息队列
		Register:     make(chan *Client),       // 等待创建连接的用户
		Unregister:   make(chan *Client),       // 等待退出断开连接的用户
		Clients:      make(map[*Client]bool),   // 客户数组
		UserToClient: make(map[string]*Client), //建立用户名和客户端唯的唯一通道
	}
}
```

main函数中运行hub并且run起来

```go
// 创建端系统的hub
	hub := starlight.NewHub()
	// 运行hub
	go hub.Run()
```

我的天，我给项目起了个名字，叫星光，太羞耻了。

当同时操控好几 chan 的时候golang官方推荐一种处理方式使用语句 select，以下案例来自官方。

> If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection. Otherwise, if there is a default case, that case is chosen. If there is no default case, the "select" statement blocks until at least one of the communications can proceed.

```go
//select基本用法
select {
case <- chan1:
// 如果chan1成功读到数据，则进行该case处理语句
case chan2 <- 1:
// 如果成功向chan2写入数据，则进行该case处理语句
default:
// 如果上面都没有成功，则进入default处理流程
```

这样就如何使用hub就很明朗了，你只需要将自己游戏或者软件里的逻辑每个业务需要使用 chan 的让他循环跑起来，像这样

```go
// Run 跑起来，完成和用户保持连接，注册，注销用户等功能，控制消息队列
func (h *Hub) Run() {
	for {
		select {

		case client := <-h.Register: // 用户创建新的连接
			{
				h.Clients[client] = true
			}

		case client := <-h.Unregister: // 用户断开连接
			{
				if _, ok := h.Clients[client]; ok {
					delete(h.Clients, client)
					close(client.Send)
				}
				delete(h.UserToClient, client.Info.UserID)
			}
		// 这个是全局消息 每个人都会收到
		case message := <-h.Message:
			{
				for client := range h.Clients {
					select {
					case client.Send <- message:
					default:
						close(client.Send)
						delete(h.Clients, client)
					}
				}
			}
		}
	}
}
```

第一个 `case client := <- h.Register `也就是从注册队列中取一个出来执行注册逻辑，下面是注销逻辑，记得用了哪些东西必须删除，然后下面那个Message是大喇叭，所有客户端都会收到消息，写好这个东西，你的加特林就已经转起来了，虽然还没有一发子弹。

那我们来看一下简单逻辑。

```go
// 监听消息
	http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
		wshandler.ServeWs(hub, w, r)
	})
```

这里我使用的是官方http库，如果要使用beego或者iris等框架，应该有对应的 FromStd 的handler，我记得 iris 的handler是一个只有 `iris.Context`的函数，但是应该有` iris.FronStd `这样的函数，总之就是一个 Handler 嘛 。然后我们来看一下ServeWs怎么写的。

```go
// serveWs handles websocket requests from the peer.
func ServeWs(hub *starlight.Hub, w http.ResponseWriter, r *http.Request) {
	//完成之后将http请求升级成socket连接
	conn, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}
	// 初始化该用户
	c := &starlight.Client{Hub: hub, Conn: conn, Send: make(chan []byte, 256)}
	// 连接注册
	hub.Register <- c

	go c.ReadPump()
	go c.WritePump()
}
```

我用升级这个词可能不专业，但是官方库的函数名称就叫做 `Upgrade`，然后中间用到了一个upgrader这个是官方写法，应该算是一个中间件，写法如下。

```go
var upgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
	CheckOrigin:     func(r *http.Request) bool { return true },
}
```

现在幻想一下，http服务也跑起来了，hub也跑起来了，每一个用户只要请求，服务器的/ws，他发起的Socket连接就会被 upgrade 成一个WebSocket连接，并且被Client持有，他的指针会被推入注册的的队列中，你再会看一下之前的注册逻辑，其实就是把Client这个map的对应项设置为true。到现在为止客户端已经可以和用户保持连接了。但是还不能收发数据。根据更前方的框架图主要在`go c.ReadPump() go c.WritePump()`这两个函数上，他们在处理，和用户交互的逻辑。 我们一样的进来看看这两个函数怎么写的。

```go
func (c *Client) ReadPump() {
	// 函数返回后执行内容
	defer func() {
		c.Hub.Unregister <- c
		_ = c.Conn.Close()
	}()

	// 设置读限制
	c.Conn.SetReadLimit(maxMessageSize)
	//read超时时间
	_ = c.Conn.SetReadDeadline(time.Now().Add(pongWait))
	// 超时操作
	c.Conn.SetPongHandler(func(string) error {
		_ = c.Conn.SetReadDeadline(time.Now().Add(pongWait))
		return nil
	})

	// 第一个消息就是处理注册
	if err := c.OnLogin(); err != nil {
		log.Println(err.Error())
		return
	}
        for { ........
.............
```

无非就是一些配置，然后再循环外面执行OnConnect的操作，我们来看看 Login 咋写的我都快忘了

```go
func (c *Client) OnLogin() error {

	// 接收用户的消息头
	_, message, err := c.Conn.ReadMessage()
	if err != nil {
		return err
	}
        // 回复注册
	c.Send <- message
}
```

我这里没有做什么特殊处理，也就是将用户发过来的消息原封不动发回去就算是成功注册了，但是在业务逻辑当中你可能需要比如，认证用户，并且加入某种权限组之类的操作，一类操作。在这里我就不再详细说。

你们有没有注意到`ReadPump()`函数开始写了一段defer

```go
defer func() {
		c.Hub.Unregister <- c
		_ = c.Conn.Close()
	}()
```

这个东西作用就是一旦用户出现任何异常，立马`return`立马关闭他的连接，防止出现任何异常，你可以写上其他操作，不过一般 注销逻辑都放在hub下面的for select那写了。

ReadPump后面for循环内的逻辑就比较简单了，

```go
for {

		// 读取消息
		_, message, err := c.Conn.ReadMessage()

		if err != nil {
			if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
				log.Printf("error: %v", err)
			}
			break
		}
		mess := Message{}

		err = json.Unmarshal(message, &mess)
		if err != nil {
			fmt.Println(err)
			continue
		}
........
```

我还是不全部粘贴了，我的垃圾代码实在是太长了，主要我的业务逻辑比较扯淡，复杂。

你会发现这样一个问题，你从c是可以取到这个client的标识的，并且可与取到hub，那么爽的来了，当某c说要发消息给另一个c`的时候，我们就直接粗暴解决问题

```go
c.Hub.UserToClient[mess.ToUserID].Send <- res
```

简直爽到不能在爽，你的加特林一直在疯狂转（其实是阻塞的），你只要把消息用 <-这个操作符推进去，消息就会自动通过`WritePump`发出去。

那么这个项目算不算完成了，暂时没有，WritePump也看一眼，这个东西是一劳永逸的函数写完一次就不用动了。

```go
func (c *Client) WritePump() {
	ticker := time.NewTicker(pingPeriod)
	defer func() {
		ticker.Stop()
		_ = c.Conn.Close()
	}()

	// 发送消息的队列
	for {
		select {
		case message, ok := <-c.Send:
			_ = c.Conn.SetWriteDeadline(time.Now().Add(writeWait))
			if !ok {
				// The hub closed the channel.
				_ = c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
				return
			}
			// TODO:这里是一个发送内容需要指定的Writer
			w, err := c.Conn.NextWriter(websocket.TextMessage)
			if err != nil {
				return
			}
			// 发送取出的第一条消息
			_, _ = w.Write(message)

			// 将队列中的消息全都发送出来
			n := len(c.Send)
			for i := 0; i < n; i++ {

				_, _ = w.Write(<-c.Send)
			}

			// 差错
			if err := w.Close(); err != nil {
				return
			}
		case <-ticker.C:
			_ = c.Conn.SetWriteDeadline(time.Now().Add(writeWait))
			if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
				return
			}
		}
	}
}
```

我都写了注释了，这个里面这个ticker我也没看是什么东西，但是从别人那儿把代码Copy过来的。到此为止这个项目就结束了。

再整理一遍，

1、 初始化一个Hub，并且运行起来

2、接收用户请求，upgrade成socket连接并且交给hub持有。

3、运行`Readpump()`，`WritePump()`来从客户端读取消息，并且写入消息。

4、每个队列各司其职，完美解决消息传输业务。

有人想看我代码么？有人看在留个言，我再整理整理，主要我的代码里面框架都被业务逻辑淹没了，比较垃圾。

就酱，最近人在山东，太冷了，冷影响写代码了。

加油，各位代码狗子们，我已经写了一年半的代码了，进展很快。现在上手什么语言过渡都很快，很适合写垃圾业务逻辑代码，啥时候能给我个清净的研究的时间啊，哎。