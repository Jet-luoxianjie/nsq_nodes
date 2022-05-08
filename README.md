---
title: rpc解析和手写rpc框架（芜湖起飞🚀）
renderNumberedHeading: true
grammar_cjkRuby: true
---


<!-- TOC -->

- [[1] 为什么要用rpc？](#为什么要用rpc)
- [[2] 什么是rpc？](#什么是rpc)
- [[3] 实现远程调用的一些思路？](#实现远程调用的一些思路)
- [[4] GRPC框架](#GRPC框架)
- - [[4.1] GRPC框架解析](#GRPC框架解析)
- - [[4.2] GRPC框架优势](#GRPC框架优势)
- - [[4.3] GRPC框架缺点](#GRPC框架缺点)
- - [[4.4] 一个简单gRPC服务的golang实现](#一个简单gRPC服务的golang实现)
- [[5] 从0到1实现简易RPC框架](#从0到1实现简易RPC框架)

<!-- /TOC -->

## 为什么要用rpc

 当项目越来越大时，**集中式**的服务(如:单一**game**服)缺点越来越明显。当然应对的方案是对项目的**拆分**(分为**logic,team,battle**等),分对多个独立的服务来开发，后随着网路请求越来越大，多个独立的服务也需要**独立部署**到**不同的物理机**上。
 
>**这里就出现了一个问题**
> ---------
> 如果team服要实现一个功能，数据只存在logic服,而logic服也有对应接口。team服怎么办？
> 
> #### 方法1：把logic的数据传输到team服，team服再写对应的函数。
> **优点**：
> 1.传输一次后，后续调用数据可以在进程内调用，快捷方便。
> **缺点**：
> 1.**logic**和**team**服都维护一样数据，容易出现数据不一致。
> **总结**：**该方法适用于数据不易变的情况。**
> 
> #### 方法2：team服发送一个消息到logic服，然后等待logic服返回消息在往下运行。
> **优点**：
> 1.调用方便。（框架实现的好类似调用本地函数一样）
> **缺点**：
> 1.每次都要调用都是一次请求，相对于本地调用会有额外网络开销会影响。
> **总结**：**适用于数据易变的情况。**
> #### 关于方法2，有一个RPC协议刚刚就可以解决这个问题。

## 什么是rpc

RPC 是 1984 年代由 Andrew D. Birrell & Bruce Jay Nelson 提出的，所以并不是最近的概念，在二位大神的论文 "Implementing Remote Procedure Calls"。 (实现远程过程调用)

> **论文连接** : https://pages.cs.wisc.edu/~sschang/OS-Qual/distOS/RPC.htm

**论文简单概括**：就是让分布式系统更加简单，让开发人员把精力放到业务上，并且提供高效安全的通信。


**再来看看比较常见的解释，了解下 RPC 是啥：**

> RPC(Remote Procedure Call) 远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。也就是说两台服务器 A，B，一个应用部署在 A 服务器上，想要调用 B 服务器上应用提供的方法，由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。

**大白话理解这段话就是说**：RPC 让你调用其他服务器的函数和调用本地函数一样。

## 实现远程调用的一些思路

   前面说了 RPC 远程过程调用就是让服务 A 像调用本地功能一样调用远端的服务 B 上的功能，不要把这个事情想的太悬乎。
  **想想我们本地调用时需要哪些东西:**
  **1.类或函数
  2.类或函数的参数
  3.类或函数的返回值。**

![](https://github.com/Jet-luoxianjie/rpc_node/blob/main/images/dy.png?raw=true)

远程调用肯定也不会缺少这三要素，唯一的区别在于这三要素是要被传输过去的，这其中就涉及协议编码和解码的过程。

**远程调用时**
这样服务 team需要通过网络传输来告诉服务 logic，它想要获取玩家信息，传入的两个参数为 1，返回的结果放在 **playerInfo** 里面就可以。
![](https://github.com/Jet-luoxianjie/rpc_node/blob/main/images/dy1.png?raw=true)

传输的报文里面按照约定的协议格式给出了**函数名**和**参数**和**返回值**即可。


> **看到网上会有人问 tcp,http,tcp的区别？（因为http也可以请求呀）**
>   rpc 应该是比 HTTP 更高层级的概念。完整的 RPC 实现包含有 传输协议 和 序列化协议，其中，传输协议既可以使用 HTTP，也可以使用 TCP 等，不同的选择可以适应不同的场景
>   rpc的框架是对 http ，tcp等协议进行的封装和优化，来实现一个**远程调用**,rpc**重点**是**远程调用**，而不是协议。

## GRPC框架

### GRPC框架解析

gRPC 是一个现代化的开源 RPC 框架，一开始由 google 开发，是一款语言中立、平台中立、的 RPC 系统，与许多 RPC 系统类似，gRPC 也是基于以下理念：定义一个 **服务**，指定能够被远程调用的 **方法**（包含参数和返回类型）。在服务端实现这个接口，并运行一个gRPC 服务器来处理客户端调用，在客户端拥有一个 stub 连接服务端上的方法
![enter description here](https://img2020.cnblogs.com/blog/496803/202003/496803-20200321153703720-1047433603.png)

### GRPC框架优势
**gRPC 的优势是它被越来越多人采用的关键所在，主要有以下几个方面：**

> 1 .提供高效的进程间通信。使用一个基于 protocol buffers 的二进制协议而不是文本格式与客户端通信，同时在 HTTP2 上实现，拥有更好的性能。

> 2.  具有简单且定义良好的服务接口。契约优先，必须首先定义服务接口，然后才能去处理细节，简单一致，可扩展。

> 3. 强类型。服务契约清晰地定义了应用程序间通信所使用的类型，分布式应用程序的开发更加稳定。

> 4. 支持多语言。基于 protocol buffers 的服务定义是语言中立的，可以选择任意一种语言具体实现。

> 5 .支持双工流。与传统的 REST 相比，gRPC 能够同时构建传统的请求-响应风格的消息以及客户端流和服务端流。

> 6 .具备内置的商业化特性。如认证、加密、弹性时间、元数据交换、压缩、负载均衡以及服务发现等。

> 7. 与云原生生态进行了集成。gRPC 是 CNCF（云原生计算基金会）的一部分，大多数现代框架和技术都对 gRPC 提供了原生支持。

> 8. 业界成熟。通过在谷歌进行的大量实战测试，gRPC 已经发展成熟，被许多公司采用。


### GRPC框架缺点

**gRPC 也存在一定劣势，选择它用来构建应用程序时，需要注意以下三点：**

> 1.gRPC 不太适合面向外部的服务。gRPC 具有契约驱动、强类型等特点，这会限制向外部暴露服务的灵活性，对客户端有诸多限制，所以更适合用在内部服务器之间通信。

> 2.避免巨大的服务定义变更。如果出现巨大的服务定义变更，通常需要重新生成客户端代码和服务端代码，会让整个开发生命周期变得复杂，需要小心引入破坏性的变更。

> 3.与REST等协议对比生态系统相对较小。


### 一个简单gRPC服务的golang实现
**下载 protoc 编译器（如:protoc-3.20.1-win32.zip）：[protobuf](https://github.com/protocolbuffers/protobuf/releases)，选择合适的平台，解压后将可执行文件加入环境变量。**
**go get google.golang.org/protobuf/cmd/protoc-gen-go**
**go get google.golang.org/grpc/cmd/protoc-gen-go-grpc**
 
 
 创建代码目录 **grpc_hero**，实现一个简单的从**队伍服**获取存在**逻辑服**中玩家英雄数据 rpc 服务，在其中新建三个文件夹 proto、server、client 分别存放服务定义文件和生成的目标代码、服务端程序实现、客户端程序实现，然后执行 go mod init grpc_hero 初始化模块。

 
 **工程目录如下**
- grpc_hero
- - logic
- - team
- - proto

#### 服务器定义
开发 gRPC 应用程序时，要首先定义服务接口，然后生成服务端骨架和客户端 stub，客户端通过调用其中定义的方法来访问远程服务器上的方法，服务定义都以 protocol buffers 的形式记录，也就是 gRPC 所使用的服务定义语言
**在 proto 目录下新建服务定义文件 hero.proto**

``` protobuf
// 版本
syntax = "proto3";
// proto文件所属包名
package proto;
// 声明生成的go文件所属的包，路径末尾为包名，相对路径是相对于编译生成目标代码时的工作路径
option go_package = "./proto";

// 包含两个远程方法的 rpc 服务，远程方法只能有一个参数和一个返回值  (一个是请求 一个是返回)
service Hero {
  rpc GetHero(Request) returns (Response);
}

// 自定义消息类型，用这种方法传递多个参数，必须使用唯一数字标识每个字段
message Response {
  HeroInfo heroInfo = 1;
}

message Request {
  int32 playerId = 1;
  int32 heroId = 2;
}

message HeroInfo {
  int32 heroId = 1;
  int32 heroLevel = 2;
  string heroName = 3;
}

```
编译服务定义文件生成目标源代码，这一步之后在 **proto** 文件下生成了以下两个文件：
**hero.pb.go**，包含用于填充、序列化、检索请求和响应消息类型的所有 protocol buffers 代码
**hero_grpc.pb.go**，包含服务端需要继承实现和客户端进行调用的接口定义

> go_out 和 go-grpc-out 目录是相对于服务定义文件中 go_package 指定的目录
> protoc proto/hero.proto --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative


#### 服务端(logic被调用端)实现
编译生成服务端骨架的时候，已经得到了建立 gRPC 连接、相关消息类型和接口的基础代码，接下来就是实现得到的接口，在 logic文件夹中新建服务端主程序 logic.go：

``` go
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"net"

	pb "grpc_hero/proto"

	"google.golang.org/grpc"
)

const (
	port = ":50051"
)

// 对服务器的抽象，用来实现服务方法
type server struct {
	pb.UnimplementedHeroServer
}

// 存放玩家的英雄数据
var playerHero map[int32]map[int32]*pb.HeroInfo

// GetHero 实现 GetHero 方法
func (s *server) GetHero(ctx context.Context, re *pb.Request) (*pb.Response, error) {
	if re == nil {
		return nil, errors.New("request nil")
	}
	heroMap, ok := playerHero[re.GetPlayerId()]
	if !ok {
		return nil, errors.New("no  heroMap")
	}
	heroInfo, ok := heroMap[re.GetHeroId()]
	if !ok {
		return nil, errors.New("no  heroInfo")
	}
	return &pb.Response{HeroInfo: heroInfo}, nil
}

//填充测试数据
func initTestData() {
	playerHero = make(map[int32]map[int32]*pb.HeroInfo)
	for i := int32(0); i < 10; i++ {
		playerHero[i] = make(map[int32]*pb.HeroInfo)
		playerHero[i][i] = &pb.HeroInfo{
			HeroId:    i,
			HeroLevel: i,
			HeroName:  fmt.Sprintf("英雄:[%d]", i),
		}
	}

}

func main() {
	//填充测试数据
	initTestData()
	// 创建一个 tcp 监听器
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	// 创建一个 gRPC 服务器实例
	s := grpc.NewServer()
	// 将服务注册到 gRPC 服务器上
	pb.RegisterHeroServer(s, &server{})
	// 绑定 gRPC 服务器到指定 tcp
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

#### 客户端(team调用端)实现
接下来创建客户端程序来与服务器对话，之前编译服务定义文件生成的目标源代码已经包含了访问细节的实现，我们只需要创建客户端实例就可以直接调用远程方法。在 team文件夹中创建客户端主程序 team.go：

``` go
package main

import (
	"context"
	pb "grpc_hero/proto"
	"log"

	"google.golang.org/grpc"
)

const (
	// 服务端地址
	address = "localhost:50051"
)

func main() {
	// 创建 gRPC 连接
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	// 创建客户端 stub，利用它调用远程方法
	c := pb.NewHeroClient(conn)
	// 调用远程方法
	r, err := c.GetHero(context.Background(), &pb.Request{
		PlayerId: 1,
		HeroId:   1,
	})
	if err != nil {
		log.Fatalf("getHero err : %v", err)
	}
	log.Printf("Response [%+v]", r)
}
```
#### 构建运行
分别构建运行服务端和客户端程序，go build 或者直接 go run
**启动logic服务端：go run ./logic/logic.go**
**启动team客户端：go run ./team/team.go**


## 从0到1实现简易RPC框架

**当前简易RPC 的目的是以最少的代码，实现 RPC 框架中最为重要的部分，帮助大家理解 RPC 框架在设计时需要考虑什么。代码简洁是第一位的，功能是第二位的。**

//todo:




