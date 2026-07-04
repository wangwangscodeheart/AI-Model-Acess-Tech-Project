# 1. 项目整体认识

## 1.1 项目目标

RPC（Remote Procedure Call）即远程过程调用，它希望屏蔽网络通信细节，使开发者能够像调用本地函数一样调用远程服务。

本项目基于 **Muduo 网络库** 实现了一套轻量级 **Json-RPC 框架**。整个项目不仅完成了 RPC 请求与响应，还逐步实现了：

-   基于 Muduo 的网络通信；
-   自定义 Length-Value 通信协议；
-   Json 消息序列化与反序列化；
-   Dispatcher 消息分发机制；
-   Requestor 请求生命周期管理；
-   同步、Future、Callback 三种调用方式；
-   服务注册与发现；
-   发布订阅模型。

整个框架围绕 **"一次完整的 RPC 调用"** 展开，各模块按照职责进行划分，共同完成一次远程调用。

----------

## 项目最终实现的功能

根据项目实现，整个框架主要包含以下几个部分：

模块**************************************功能

RPC 调用******支持客户端远程调用服务端接口

网络通信******基于 Muduo 实现 TCP 长连接通信

通信协议******自定义 Length-Value 协议解决 TCP 粘包问题

消息序列化******使用 Json 组织请求与响应数据

消息分发******Dispatcher 根据消息类型分发到对应模块

请求管理******Requestor 管理客户端请求生命周期

服务注册******Provider 向注册中心注册服务

服务发现******Caller 从注册中心获取可用服务

发布订阅******支持 Topic 创建、订阅及消息广播

这些模块共同组成了一套完整的 RPC 通信框架。

----------

## 项目整体调用流程

整个项目可以概括为下面这条调用链：

```
Caller    
│    
▼
Requestor    
│
▼
Protocol（Json + Length-Value）    
│    
▼
Muduo Network    
│    
▼
Dispatcher    
│
▼
Provider    
│    
▼
RpcService    
│    
▼
Response
```

客户端发起 RPC 调用后，Requestor 负责组织请求并发送数据。

服务端收到完整消息后，由 Dispatcher 根据消息类型进行分发，再交给对应业务模块处理，最后将处理结果返回客户端。

整个过程中，各模块之间通过消息进行协作，而不是直接依赖具体业务逻辑。

----------

## 为什么要划分这么多模块

整个项目没有把所有逻辑集中到 Client 或 Server 中，而是按照不同职责进行拆分。

主要可以划分为以下几层：

```
网络通信层        
│
协议解析层        
│
消息分发层        
│
请求管理层        
│
业务处理层
```

这样划分主要解决了几个问题：

-   网络通信与业务逻辑解耦；
-   协议解析与具体业务解耦；
-   不同类型消息统一交给 Dispatcher 管理；
-   客户端请求统一交给 Requestor 管理；
-   后续增加新的业务模块时，不需要修改已有网络代码。

整个框架也因此具有较好的扩展性和可维护性。

----------

## 本章涉及的核心知识点

-   RPC 基本原理
-   TCP 网络通信
-   Muduo Reactor 模型
-   自定义应用层协议
-   Json 序列化与反序列化
-   请求响应模型（Request-Response）
-   同步与异步编程
-   Future / Promise
-   回调机制（Callback）
-   服务注册与发现
-   发布订阅模型
-   模块解耦与分层设计

----------
# 1.2 为什么需要 RPC

## RPC 要解决什么问题

RPC（Remote Procedure Call）的目标可以概括为一句话：

> **屏蔽底层网络通信细节，使远程服务调用方式尽可能接近本地函数调用。**

例如，本地函数调用：

```
UserLogin(name, password);
```

开发者只需要关心：

-   调用哪个函数；
-   传递哪些参数；
-   返回什么结果。

整个调用过程发生在同一个进程中，不需要考虑网络通信。

而当服务部署到另一台机器后，事情就开始变得复杂。

客户端不仅需要知道：

-   服务运行在哪台机器；
-   使用哪个端口；
-   如何建立 TCP 连接；
-   如何发送数据；
-   如何等待响应；
-   如何解析返回结果。

这些本来与业务无关的问题，都需要业务代码自己完成。

RPC 的目的就是把这些细节全部隐藏起来。

业务层仍然按照"调用函数"的方式使用远程服务，而底层通信过程全部由 RPC 框架完成。

----------

## 如果没有 RPC，需要做什么

假设客户端希望调用服务端登录接口。

如果直接使用 Socket 编程，大致需要完成下面几个步骤：

```
创建 Socket↓建立 TCP 连接↓组织请求数据↓发送请求↓等待服务端响应↓接收数据↓解析数据↓关闭连接
```

这些代码与真正的业务逻辑没有任何关系。

但是每调用一次远程服务，都必须重复完成这一套流程。

如果项目中存在几十甚至上百个远程接口，那么客户端将充满大量重复的网络通信代码。

因此，需要把这些公共逻辑统一封装。

----------

## RPC 框架主要解决哪些问题

结合整个项目，可以把 RPC 解决的问题划分为四部分。

### ① 网络通信

负责客户端与服务端之间的数据传输。

本项目使用 **Muduo** 完成 TCP 网络通信，负责连接建立、数据发送以及数据接收。

这一层只负责传输数据，不关心数据具体表示什么。

----------

### ② 消息组织

网络上传输的是字节流，因此需要规定一套统一的数据格式。

项目中使用 Json 保存请求与响应数据，并在外层增加自定义 Length-Value 协议，用于解决 TCP 粘包与拆包问题。

这样客户端与服务端才能正确解析收到的数据。

----------

### ③ 请求管理

一次 RPC 调用不仅需要发送请求，还需要知道：

-   哪个响应对应哪个请求；
-   同步调用如何返回结果；
-   异步调用如何通知调用者。

因此项目中增加了 **Requestor** 模块，对客户端所有请求进行统一管理。

这一部分也是整个项目最核心的模块之一。

----------

### ④ 服务管理

客户端真正关心的是"调用哪个服务"，而不是"连接哪台服务器"。

因此项目又增加了：

-   服务注册；
-   服务发现；
-   服务上下线通知；

使服务能够动态管理，而不是把服务器地址写死在客户端。

----------

## 本项目中 RPC 调用的大致流程

整个 RPC 调用过程可以概括为下面几个阶段：

```
业务层发起调用↓Requestor 组织请求↓Protocol 封装消息↓Muduo 发送数据↓服务端接收消息↓Dispatcher 分发消息↓执行对应业务↓返回响应↓Requestor 接收响应
```

虽然整个流程包含多个模块，但每个模块职责都比较单一。

这种分层设计也是整个框架后续容易扩展的重要原因。

----------

## 本章涉及知识点

-   RPC 基本原理
-   Socket 网络通信
-   Request / Response 模型
-   Json 消息组织
-   自定义应用层协议
-   请求生命周期管理
-   服务注册与发现


这一章重点不是记住 RPC 的定义，而是明确 **RPC 框架究竟解决了哪些问题**。

整个项目实际上围绕四件事情展开：

-   网络通信；
-   消息组织；
-   请求管理；
-   服务管理。

后续所有模块，都可以归属于这四个方向。

----------
# 2. 网络通信层（Muduo + Reactor）

## 2.1 为什么选择 Muduo

整个 RPC 框架的基础是网络通信，因此首先需要解决客户端与服务端之间的数据传输问题。

项目最终选择 Muduo 作为网络通信框架，而没有直接使用 Socket API，主要原因有以下几点：

-   Muduo 已经封装好了 TCP Server、TCP Client 等网络组件，不需要重复编写底层网络代码；
-   基于 Reactor 模型实现事件驱动，能够同时管理多个连接；
-   网络通信与业务逻辑天然分离，更适合作为 RPC 框架的网络层。

因此，在整个项目中，Muduo 主要负责完成两件事情：

-   建立客户端与服务端连接；
-   完成数据的发送与接收。

除此之外，RPC 相关逻辑全部由框架自身实现。

----------

## 2.2 Muduo 在项目中的职责

整个项目中，Muduo 只负责网络通信。

收到数据以后，并不会直接处理 RPC 请求，而是把完整数据继续交给后续模块处理。

整个流程可以表示为：

```
Muduo 接收数据        
│        
▼
Protocol 解析协议        
│        
▼
Dispatcher 消息分发        
│        
▼
对应业务模块
```

也就是说：

Muduo 并不知道收到的是：

-   RPC 请求；
-   服务注册消息；
-   服务发现消息；
-   Topic 消息。

对于 Muduo 来说，它只是负责收发字节流。

真正的业务处理发生在网络层之后。

----------

## 2.3 为什么网络层不直接处理业务

刚开始实现项目时，很容易想到一种写法：

收到数据以后：

```
onMessage(...)
{    
	解析Json;    
	判断消息类型;    
	执行业务逻辑;
}
```

这种方式虽然能够完成所有功能，但是随着消息类型不断增加：

-   RPC
-   Registry
-   Discovery
-   Publish
-   Subscribe

网络层会越来越复杂。

最终网络通信和业务逻辑完全耦合在一起。

因此项目没有采用这种实现方式。

网络层收到完整消息以后，只负责继续向下传递，由 Dispatcher 根据消息类型完成后续处理。

这样网络层始终保持简单，只负责：

-   接收数据；
-   发送数据。

而不负责任何业务逻辑。

----------

## 2.4 Reactor 在项目中的作用

Muduo 底层采用 Reactor 模型。

对于整个 RPC 项目来说，并不需要深入研究 Muduo 的源码实现，而需要理解 Reactor 给项目带来了什么。

整个 Reactor 可以理解为：

```
事件发生
↓
EventLoop
↓
对应事件回调
↓
业务处理
```

例如：

客户端发送数据以后，

服务端并不会主动轮询是否收到请求，而是等待读事件发生。

当连接上有数据到达时，Muduo 自动回调对应消息处理函数。

RPC 框架也是从这里开始进入自己的处理流程。

因此，Reactor 提供的是一种事件驱动模型，而 RPC 框架则建立在这一模型之上。

----------

## 2.5 网络层与 RPC 框架的关系

整个项目可以理解为两层：

```
RPC Framework
────────────────────
Muduo Network
```

Muduo 负责：

-   网络连接；
-   数据收发；

RPC Framework 负责：

-   协议解析；
-   请求管理；
-   消息分发；
-   服务调用；
-   服务注册；
-   发布订阅。

二者职责完全不同。

整个项目实际上是在 Muduo 提供的网络能力之上，实现了一套完整的 RPC 通信框架。

----------

## 本章涉及知识点

-   Reactor 模型
-   EventLoop
-   TCP Server / TCP Client
-   事件驱动
-   网络层与业务层解耦
-   Muduo 网络库

Muduo 并不是 RPC 框架的一部分，而是整个 RPC 框架的网络基础。

整个项目真正需要关注的是：

> **Muduo 负责"传输"，RPC 框架负责"处理"。**

二者之间通过完整消息进行衔接，而不是直接耦合业务逻辑。

# 3. 自定义协议与粘包处理

## 3.1 为什么需要自定义协议

整个 RPC 框架建立在 TCP 连接之上，而 TCP 本身是一种**面向字节流**的传输协议。

也就是说，TCP 负责保证数据能够可靠到达，但并不会保留应用层消息之间的边界。

例如客户端连续发送两条 RPC 请求：

```
Request1
Request2
```

服务端收到的数据可能变成：

```
Request1Request2
```

也可能是：

```
Reque
st1Requ
est2
```

对于 TCP 来说，这两种情况都是正常的。

因此，仅仅依赖 `recv()` 或 Muduo 的消息回调，并不能知道：

-   一条消息从哪里开始；
-   一条消息在哪里结束；
-   当前收到的数据是否已经完整。

所以，在应用层必须设计一套自己的通信协议。

----------

## 3.2 TCP 为什么会发生粘包

需要明确一点：

**粘包并不是 TCP 的 Bug，而是 TCP 的正常工作方式。**

TCP 只保证：

-   数据可靠传输；
-   数据顺序一致。

它并不会关心：

> "这一段数据是不是一条完整的 RPC 请求。"

因此：

客户端发送两次：

```
send(A)send(B)
```

服务端可能收到：

```
AB
```

也可能收到：

```
A

B
```

甚至：

```
A的一半A剩余部分+B
```

因此，应用层必须能够自己判断：

> **什么时候才算收到了一条完整消息。**

----------

## 3.3 常见解决方案

应用层协议通常有三种设计方式。

### ① 固定长度

每条消息长度固定。

例如：

```
1024 Byte
```

优点：

-   实现简单；
-   不需要解析长度。

缺点：

-   浪费空间；
-   消息长度不灵活。

并不适合作为 RPC 请求格式。

----------

### ② 分隔符

每条消息增加结束标记。

例如：

```
message\n
```

读取到指定分隔符后认为消息结束。

优点：

实现简单。

缺点：

如果正文中也出现相同分隔符，需要额外进行转义处理。

对于 Json 数据来说并不方便。

----------

### ③ Length-Value（LV）

在消息最前面增加长度字段：

```
Length

Body
```

先读取 Length。

再根据 Length 读取后续数据。

这是整个项目最终采用的方案。

----------

## 3.4 为什么选择 Length-Value

整个项目最终采用 Length-Value 协议，主要有几个原因。

第一，消息长度不固定。

RPC 请求可能包含不同数量的参数。

Json 长度也完全不同。

固定长度并不适用。

第二，Json 本身可能包含各种字符。

如果使用分隔符方案，需要考虑分隔符冲突问题。

第三，Length-Value 实现简单。

收到数据以后：

先读取长度。

再判断当前 Buffer 中是否已经收到完整消息。

如果没有收到完整数据，则继续等待下一次网络数据到达。

整个解析过程比较清晰，也更适合 RPC 通信。

----------

## 3.5 项目协议格式

根据项目设计，应用层消息由两部分组成：

```
+----------------------+
|      Length          |
+----------------------+
|        Body          |
+----------------------+
```

其中：

-   **Length**：表示后续消息体长度；
-   **Body**：保存真正的 Json 消息内容。

网络层首先读取 Length，再根据 Length 从 Buffer 中取出完整消息，随后交给 Json 解析模块继续处理。

整个协议保持了较高的通用性。

无论后续发送：

-   RPC 请求；
-   RPC 响应；
-   服务注册消息；
-   服务发现消息；
-   Topic 消息；

都可以复用同一套协议。

----------

## 3.6 协议解析流程

协议解析过程可以概括为：

```
收到网络数据        
	 │        
	 ▼
读取 Length        
	 │        
	 ▼
判断数据是否完整        
	 │   
┌────┴────┐   
│         │ 
不完整     完整   
│         │
继续等待    解析 Body              
		  │              
		  ▼        
		  Json 反序列化              
		  │              
		  ▼         
		  Dispatcher 分发
```

整个协议层只负责完成：

-   消息完整性判断；
-   消息编解码。

不会参与任何业务处理。

解析完成以后，消息立即进入 Dispatcher，由后续模块继续处理。

----------

## 本章涉及知识点

-   TCP 字节流
-   粘包与拆包
-   应用层协议设计
-   Length-Value 协议
-   Buffer 数据解析
-   Json 编解码

整个项目能够正确完成 RPC 通信，前提就是能够正确解析每一条消息。

因此，自定义协议真正解决的问题不是"传输数据"，而是：

> **如何从连续的 TCP 字节流中准确恢复出一条完整的应用层消息。**

Length-Value 协议正是整个 RPC 框架通信的基础，也是后续所有模块能够正常工作的前提。

# 4. Dispatcher 消息分发设计

## 4.1 为什么需要 Dispatcher

协议层完成消息解析以后，已经得到了一条完整的 `BaseMessage`。

但是此时框架仍然面临一个问题：

> **收到的消息应该交给谁处理？**

整个项目不仅包含 RPC 请求，还包含：

-   服务注册
-   服务发现
-   发布订阅
-   RPC 响应

不同消息对应不同业务模块。

如果收到消息以后直接在网络层不断判断：

```
if(...)
else if(...)
else if(...)
```

随着消息类型不断增加，网络层会承担越来越多业务逻辑，后续维护成本也会越来越高。

因此项目增加了 Dispatcher 模块，专门负责消息分发。

Dispatcher 不负责处理业务，只负责找到对应的业务处理函数。

----------

## 4.2 Dispatcher 的核心思想

Dispatcher 本质上维护了一张：

```
消息类型
↓
处理函数
```

之间的映射关系。

源码中对应的数据结构：

```
std::unordered_map<MType, Callback::ptr> _handlers;
```

其中：

-   `MType` 表示消息类型；
-   `Callback` 表示对应消息处理函数。

收到消息以后，只需要根据消息类型查找对应 Handler 即可。

整个 Dispatcher 不需要知道：

-   Handler 做了什么；
-   Handler 属于哪个模块；

它只负责：

> **找到并调用。**

----------

## 4.3 Handler 是如何注册的

Dispatcher 并没有把所有 Handler 写死。

而是提供了统一注册接口：

```
template<typename T>void registerHandler(...)
```

项目启动时，各业务模块主动调用 `registerHandler()`，将自己的消息处理函数注册到 Dispatcher。

Dispatcher 负责保存：

```
MType
↓
Callback
```

之间的对应关系。

后续收到消息以后，就可以直接查表完成分发。

这种设计避免了大量 `if-else` 判断，也使新增一种消息类型时，不需要修改 Dispatcher 内部逻辑。

----------

## 4.4 为什么设计 Callback 抽象类

源码中首先定义了：

```
class Callback
```

它只有一个接口：

```
virtual void onMessage(...)
```

所有消息处理函数最终都会继承这个统一接口。

这样 Dispatcher 就不需要关心具体 Handler 类型。

无论处理的是：

-   RPC
-   Registry
-   Topic

最终调用方式都是：

```
callback->onMessage(...)
```

整个调用过程保持一致。

----------

## 4.5 为什么又实现 CallbackT<T>

不同消息最终对应不同 Message 类型。

例如：

```
RpcMessage
RegistryMessage
TopicMessage
```

如果全部使用 BaseMessage，就需要大量类型转换。

因此项目又增加了：

```
template<typename T>class CallbackT
```

Dispatcher 保存的是：

```
Callback
```

真正执行时：

```
dynamic_pointer_cast<T>
```

恢复成具体消息类型。

随后调用对应业务 Handler。

这种设计兼顾了：

-   Dispatcher 的统一管理；
-   各业务模块的类型安全。

----------

## 4.6 Dispatcher 的消息分发流程

整个 Dispatcher 工作流程如下：

```
收到 BaseMessage        
	 │        
	 ▼
读取 mtype()        
	 │        
	 ▼
查询 _handlers        
	 │
┌────┴────┐   
│         │ 
找到      未找到   
│         │
执行回调   输出日志   
│         │
业务处理   关闭连接
```

源码中：

```
auto it = _handlers.find(msg->mtype());
```

找到对应 Handler 后：

```
it->second->onMessage(conn, msg);
```

否则：

```
conn->shutdown();
```

结束连接。

----------

## 4.7 本章涉及知识点

-   回调（Callback）
-   模板（Template）
-   多态（Virtual）
-   `std::function`
-   `unordered_map`
-   `dynamic_pointer_cast`
-   消息分发（Dispatcher Pattern）

！！！Dispatcher 并不是业务模块。

它真正完成的是：

-   保存消息类型与处理函数之间的映射；
-   根据 `mtype()` 查找对应 Handler；
-   调用统一的 `onMessage()` 接口完成消息分发。

整个设计最大的特点不是 `if-else` 改成 `map`，而是利用：

> **模板 + 多态 + 回调注册**

实现了统一的消息分发机制。

# 5. Requestor 请求管理设计

## 5.1 为什么需要 Requestor

刚开始实现 RPC 时，我认为客户端只需要完成两件事情：

```
发送请求
↓
等待响应
```

后来随着项目逐渐完善，我发现这种方式只能支持最简单的同步调用。

如果客户端连续发送多个 RPC 请求，就会遇到新的问题：

-   当前收到的响应属于哪一个请求？
-   多个请求同时发送，响应顺序是否一定一致？
-   Future 如何知道什么时候返回？
-   Callback 又应该什么时候执行？

因此，仅仅负责发送请求已经无法满足整个框架的需求。

项目最终增加了 **Requestor** 模块，对客户端所有请求进行统一管理。

整个请求从创建、发送、等待响应直到请求结束，都由 Requestor 负责。

----------

## 5.2 Requestor 在整个项目中的位置

整个客户端调用流程如下：

```
Caller    
│
▼
Requestor    
│    
▼
Protocol    
│    
▼
Muduo
```

其中：

Caller 只关心调用哪个 RPC 接口。

Protocol 只负责消息封装。

Muduo 负责网络发送。

而 Requestor 位于中间，负责管理整个请求生命周期。

因此，Requestor 更像客户端的一层"调度中心"，而不是简单的发送模块。

----------

## 5.3 Requestor 管理什么

阅读源码后可以发现，Requestor 最重要的职责不是发送请求，而是维护所有**未完成请求**的信息。

项目中，每发送一个请求，都会创建对应的请求描述对象，并保存到 Requestor 中。

请求完成之前，这些信息都会一直保存在 Requestor 内部。

收到响应以后，再根据对应信息继续完成后续处理。

因此，Requestor 实际管理的是：

```
请求创建
↓
保存请求信息
↓
发送请求
↓
等待响应
↓
收到响应
↓
结束请求
```

整个生命周期。

----------

## 5.4 为什么需要 rid

项目中，每一个请求都会拥有唯一的 **rid**。

rid 的作用只有一个：

> **唯一标识一次 RPC 请求。**

客户端可能同时发送多个请求。

服务端返回响应时，并不会告诉客户端：

> "这是第几个发送出去的请求。"

因此只能依赖请求编号进行对应。

整个流程可以理解为：

```
Request1
rid = 1
Request2
rid = 2
Request3
rid = 3
```

收到响应：

```
Response
↓
rid = 2
↓
找到对应请求
```

因此：

rid 连接了请求与响应，也是 Requestor 能够管理多个请求的基础。

----------

## 5.5 RequestDescribe 的作用

源码中，请求并不仅仅保存一个 rid。

而是通过 **RequestDescribe** 保存一次请求相关的全部信息。

可以理解为：

```
一次RPC请求
↓
RequestDescribe
↓
保存请求状态
```

RequestDescribe 可以看作整个请求生命周期的载体。

无论是同步调用还是异步调用，最终都会围绕 RequestDescribe 展开。

因此，它也是 Requestor 最重要的数据结构之一。

----------

## 5.6 Requestor 与同步、异步调用的关系

阅读源码以后可以发现：

同步调用、Future 调用以及 Callback 调用，并不是三套完全独立的流程。

它们最终都会经过 Requestor。

区别仅仅在于：

收到响应以后采用不同方式通知调用者。

因此可以理解为：

```
Requestor        
│        
├──────── 同步调用        
│        
├──────── Future        
│        
└──────── Callback
```

整个请求发送流程保持一致。

真正发生变化的是：

> **响应返回之后如何继续执行。**

这也是后面同步、Future、Callback 能够统一实现的重要原因。

----------
## 5.7 RequestDescribe 设计（Requestor 核心数据结构）

### 为什么需要 RequestDescribe

Requestor 并不是简单地把请求发送出去。

对于每一次 RPC 调用，在收到响应之前，都需要保存一些与该请求相关的信息，例如：

-   当前发送的是哪一个请求；
-   请求采用哪种调用方式；
-   收到响应后应该如何通知调用者。

如果这些信息分散保存在多个地方，请求管理会变得非常混乱。

因此，项目定义了 `RequestDescribe`，统一保存一次 RPC 请求的所有上下文信息。

可以理解为：

```
一次 RPC 请求        
│        
▼
RequestDescribe        
│        
├── request        
├── rtype        
├── response        
└── callback
```

整个请求在生命周期内，都围绕这个对象进行管理。

----------

## RequestDescribe 成员分析

源码中定义如下：

```
struct RequestDescribe 
{    
	BaseMessage::ptr request;    
	RType rtype;    
	std::promise<BaseMessage::ptr> response;    
	RequestCallback callback;
};
```

每个成员都有明确的职责。

### ① request

```
BaseMessage::ptr request;
```

保存当前请求对象。

这样 Requestor 能够知道当前请求对应的是哪一条 RPC 消息，也可以通过 `request->rid()` 作为唯一标识，将请求保存到 `_request_desc` 中。

----------

### ② rtype

```
RType rtype;
```

表示当前请求采用的调用方式。

源码中主要有两种：

-   `REQ_ASYNC`
-   `REQ_CALLBACK`

收到响应以后，Requestor 首先判断 `rtype`，再决定采用哪种方式处理响应。

因此，`rtype` 相当于一次请求的处理策略。

----------

### ③ response

```
std::promise<BaseMessage::ptr> response;
```

这个成员只用于异步（Future）调用。

创建请求时：

```
rdp->response.get_future();
```

返回一个 `future` 给调用者。

收到响应后：

```
rdp->response.set_value(msg);
```

将结果写入 `promise`。

之前等待 `future.get()` 的线程会立即得到响应结果。

因此，`promise` 与 `future` 共同完成了异步结果的传递。

----------

### ④ callback

```
RequestCallback callback;
```

当请求采用回调方式发送时，Requestor 会把用户传入的回调函数保存在这里。

收到响应以后：

```
rdp->callback(msg);
```

直接执行回调。

整个过程中，不需要 `future`，也不会阻塞等待。

----------

### RequestDescribe 的创建过程

每发送一次请求，Requestor 都会调用：

```
newDescribe(...)
```

创建对应的 `RequestDescribe`。

源码中主要完成了几件事情：

```
RequestDescribe::ptr rd = std::make_shared<RequestDescribe>();rd->request = req;rd->rtype = rtype;if (rtype == RType::REQ_CALLBACK && cb) {    rd->callback = cb;}
```

随后：

```
_request_desc.insert(    std::make_pair(req->rid(), rd));
```

将请求保存到 `_request_desc` 中。

这里使用 **请求的 `rid`** 作为 Key，保证后续收到响应时能够快速找到对应请求。

----------

### _request_desc 的作用

源码中真正管理所有请求的是：

```
std::unordered_map<std::string, RequestDescribe::ptr> _request_desc;
```

它保存了：

```
rid        │        ▼RequestDescribe
```

之间的映射关系。

因此，可以认为：

> `_request_desc` 保存了当前客户端所有**尚未完成**的 RPC 请求。

请求发送成功后加入容器。

收到响应后，再调用：

```
delDescribe(rid);
```

将该请求移除。

整个容器始终只保存**未完成请求**。

----------

### RequestDescribe 在整个流程中的作用

一次完整请求的生命周期如下：

```
发送请求    
│    
▼
newDescribe()    
│    
▼
创建 RequestDescribe    
│    
▼
保存到 _request_desc    
│    
▼
发送网络请求    
│
▼
收到响应    
│    
▼
getDescribe(rid)    
│    
▼
根据 rtype 处理响应    
│    
▼
delDescribe(rid)
```

可以看到，`RequestDescribe` 并不是一个简单的数据结构，而是贯穿了一次 RPC 请求的整个生命周期。

`RequestDescribe` 是 Requestor 最核心的数据结构。

它将一次 RPC 请求需要的所有上下文信息封装到一个对象中，再通过 `_request_desc` 统一管理。

这种设计使 Requestor 不需要维护多套数据结构，而是以 **"一个请求对应一个 RequestDescribe"** 的方式管理整个请求生命周期，也为 Future 和 Callback 两种调用方式提供了统一的数据基础。

## 5.8 Requestor 如何处理响应

### 响应是如何找到对应请求的

客户端收到服务端返回的数据以后，并不会立即返回给业务层。

首先需要解决一个问题：

> **这条响应属于哪一个请求？**

项目中，每个请求发送之前都会分配唯一的 `rid`。

服务端处理完成以后，会把同一个 `rid` 放回响应消息中。

因此，当 Requestor 收到响应时，首先执行的并不是业务处理，而是根据 `rid` 查找之前保存的请求描述对象。

整个过程可以表示为：

```
收到 Response        
│        
▼
读取 rid        
│        
▼
_request_desc.find(rid)        
│        
▼
得到 RequestDescribe
```

这样，无论客户端同时发送多少个请求，都能够准确找到对应的请求信息。

----------

### getDescribe 的作用

源码中提供了：

```
getDescribe(const std::string& rid)
```

它的职责非常简单：

> 根据 `rid` 从 `_request_desc` 中查找对应的 `RequestDescribe`。

如果能够找到：

说明该请求仍然存在，可以继续处理响应。

如果没有找到：

说明该请求可能已经结束，或者 `rid` 非法，此时不会继续执行后续逻辑。

因此，整个 Requestor 都围绕：

```
rid
↓
RequestDescribe
```

这一映射关系工作。

----------

### 根据调用方式处理响应

找到 `RequestDescribe` 后，Requestor 并不会立即把结果返回。

而是首先判断：

```
rtype
```

因为不同调用方式，响应处理流程完全不同。

源码中主要分为两种情况。

----------

#### Future 调用

如果当前请求采用 Future 方式。

收到响应以后：

```
rd->response.set_value(msg);
```

将响应对象写入 `promise`。

此前调用：

```
future.get();
```

等待结果的线程，此时会立即被唤醒。

整个过程中：

Requestor 并不会主动调用业务代码。

它只负责：

> **把结果放入 promise。**

后续由 Future 自己返回结果。

----------

#### Callback 调用

如果当前请求采用 Callback 方式。

收到响应以后：

```
rd->callback(msg);
```

直接执行之前保存的回调函数。

整个调用过程无需阻塞，也不需要 Future。

Requestor 只负责找到回调并执行。

至于回调函数内部如何处理响应，则由业务层决定。

----------

### 请求什么时候结束

无论 Future 还是 Callback。

请求处理完成以后，都需要将对应请求从 `_request_desc` 中移除。

源码中调用：

```
delDescribe(rid);
```

完成清理。

整个过程如下：

```
发送请求        
│        
▼
_request_desc 保存        
│        
▼
等待响应        
│        
▼
收到响应        
│        
▼
处理响应        
│        
▼
删除 RequestDescribe
```

因此，`_request_desc` 中始终保存的是：

> **当前所有尚未完成的请求。**

已经完成的请求不会继续保留。

这样既保证了请求能够正确匹配，也避免容器不断增长。

----------

### 为什么要立即删除

如果请求处理完成以后仍然保留在 `_request_desc` 中。

随着 RPC 调用不断增加：

```
_request_desc
↓
越来越大
```

不仅会浪费内存。

还可能导致：

-   重复处理同一个响应；
-   查找效率下降；
-   已完成请求长期占用资源。

因此，项目采用：

> **请求完成立即释放。**

整个 Requestor 生命周期管理也因此形成一个完整闭环。

----------

Requestor 收到响应以后，并不会直接返回给调用者。

而是按照：

```
Response    
│    
▼
rid    
│
▼
RequestDescribe    
│    
▼
rtype    
│    
├── Future    
└── Callback
```

完成统一处理。

整个模块真正的核心思想是：

> **所有请求统一管理，所有响应统一分发，根据调用方式决定最终如何通知业务层。**

## 5.9 Requestor 对外提供的调用接口

### 为什么需要统一调用接口

Requestor 内部已经完成了请求管理、请求匹配以及响应处理。

对于业务层来说，并不需要关心这些实现细节。

业务层真正关心的是：

-   我想同步调用；
-   我想异步获取结果；
-   我想收到结果后执行回调。

因此，Requestor 对外提供了统一的调用接口，将请求管理逻辑全部封装在内部。

业务层只需要选择不同的调用方式即可。

----------

### 同步调用

同步调用是最直接的一种方式。

调用流程如下：

```
业务层    
│    
▼
Requestor    
│    
▼
发送请求    
│    
▼
等待响应    
│    
▼
返回 Response
```

整个调用期间，当前线程会一直等待，直到收到对应响应后才继续向下执行。

这种方式实现简单，调用方式也最接近普通函数调用。

但是，同步等待期间线程无法继续执行其它任务，因此适用于对实时返回结果要求较高的场景。

----------

### Future 调用

Future 调用本质上仍然会发送 RPC 请求。

不同的是，发送完成以后并不会阻塞等待响应。

Requestor 创建 `RequestDescribe` 时，会返回一个：

```
std::future<BaseMessage::ptr>
```

业务层可以继续执行其它逻辑。

当真正需要结果时，再调用：

```
future.get();
```

等待返回值。

整个流程如下：

```
发送请求      
│      
▼
返回 Future      
│      
▼
继续执行其它代码      
│      
▼
收到响应      
│      
▼
promise.set_value()      
│      
▼
future.get() 返回
```

整个过程中，请求生命周期仍然由 Requestor 管理。

Future 只是提供了一种延迟获取结果的方式。

----------

### Callback 调用

Callback 调用也是异步调用。

不同的是，它不会返回 Future。

发送请求时，业务层直接传入一个回调函数。

```
发送请求      
│      
▼
保存 Callback      
│      
▼
收到响应      
│      
▼
执行 Callback
```

收到响应以后，Requestor 根据 `rtype` 找到之前保存的回调函数，并立即执行。

整个过程中，业务层无需主动等待，也无需主动获取结果。

这种方式适用于收到响应后立即执行后续业务逻辑的场景。

----------

### 三种调用方式的共同点

虽然三种接口使用方式不同，但阅读源码可以发现，它们最终都会经过相同的请求管理流程。

统一流程如下：

```
创建 RequestDescribe        
│        
▼
保存到 _request_desc        
│        
▼
发送请求        
│        
▼
等待服务端响应        
│        
▼
根据 rid 找到 RequestDescribe        
│        
▼
根据 rtype 处理响应
```

区别仅存在于**响应返回后的处理方式**：

-   同步调用：等待结果后直接返回；
-   Future：通过 `promise/future` 传递结果；
-   Callback：执行用户注册的回调函数。

因此，Requestor 并没有针对三种调用方式分别实现三套请求管理逻辑，而是在统一生命周期管理的基础上，根据 `rtype` 选择不同的响应处理策略。

----------

### 设计上的优点

这种设计最大的优点在于：

-   请求发送逻辑统一；
-   请求管理逻辑统一；
-   响应匹配逻辑统一。

真正变化的只有响应通知方式。

因此，无论以后新增新的调用模式，还是修改请求管理流程，都只需要修改 Requestor，而不需要影响网络层和业务层。

整个模块保持了较好的扩展性。

----------
# 6. Json 消息设计

## 6.1 为什么又抽象了一层 JsonMessage

阅读源码后，我发现整个消息体系并不是：

```
BaseMessage    
│    
├── RpcRequest    
├── RpcResponse    
├── ...
```

而是又增加了一层：

```
BaseMessage      
	 │      
	 ▼
JsonMessage      
	 │ 
┌────┴───────────────┐ 
│                    │
JsonRequest      JsonResponse 
│                    │ 
├──RpcRequest    	 ├──RpcResponse 
├──TopicRequest  	 ├──TopicResponse 
└──ServiceRequest 	 └──ServiceResponse
```

这层 `JsonMessage` 的作用非常明确：

> **统一完成 Json 的序列化与反序列化。**

源码中：

```
virtual std::string serialize() override 
{    
	std::string body;    
	bool ret = JSON::serialize(_body, body);    
	...
}
virtual bool unserialize(const std::string &msg) override 
{    
	return JSON::unserialize(msg, _body);
}
```

可以看到，它内部维护了一个：

```
Json::Value _body;
```

所有消息最终都把自己的字段保存到 `_body` 中。

因此：

-   RpcRequest 不需要自己实现 Json 序列化；
-   ServiceRequest 不需要重复写 Json 解析；
-   TopicRequest 也不用重复实现。

所有 Json 编解码全部集中到了 `JsonMessage`。

这样避免了大量重复代码。

----------

## 6.2 为什么又分成 JsonRequest 与 JsonResponse

继续往下看源码，又发现：

```
class JsonRequest : public JsonMessage
```

和

```
class JsonResponse : public JsonMessage
```

这里最开始我以为只是为了分类。

实际上不是。

真正的原因是：

**所有 Response 都具有共同的"响应码"。**

源码中：

```
virtual bool check() override
{    
	if (_body[KEY_RCODE].isNull())        
	...
}
```

以及：

```
RCode rcode();
void setRCode(RCode rcode);
```

这些接口全部放在 `JsonResponse` 中。

因此：

RpcResponse

TopicResponse

ServiceResponse

天然拥有：

```
RCodecheck()
setRCode()
```

而 Request 根本不需要这些成员。

所以这里再次抽象了一层。

这是一个典型的：

> **提取公共行为。**

而不是为了继承而继承。

----------

## 6.3 RpcRequest 的设计

RpcRequest 是整个 RPC 调用最核心的消息。

源码中它并没有保存成员变量。

所有数据全部保存在：

```
_body
```

里面。

它真正提供的是访问接口。

例如：

```
method()
setMethod()
```

对应：

```
_body[KEY_METHOD]
```

而：

```
params()
setParams()
```

对应：

```
_body[KEY_PARAMS]
```

也就是说。

RpcRequest 本身并没有：

```
std::string method;
Json::Value params;
```

这些成员。

而是统一放到 Json 对象中管理。

这样序列化时，不需要再做一次对象到 Json 的转换。

因为消息本身一直就是 Json。

----------

## 6.4 check() 为什么存在

源码中每一种 Message 都实现了：

```
check()
```

例如 RpcRequest：

它检查：

```
KEY_METHOD
```

是否存在。

并且必须是：

```
String
```

随后检查：

```
KEY_PARAMS
```

必须存在。

并且必须是：

```
Object
```

如果任何字段不满足要求：

直接返回 false。

ServiceRequest、

TopicRequest、

RpcResponse

也是同样设计。

也就是说：

**每一种消息自己负责校验自己的合法性。**

而不是 Dispatcher 去判断。

也不是 Protocol 去判断。

这符合单一职责原则。

----------

## 6.5 Message 为什么全部操作 Json::Value

阅读源码以后，我觉得这里是一个设计亮点。

项目没有写：

```
class RpcRequest
{    
	std::string method;    
	Json::Value params;
};
```

然后再写：

```
toJson()
```

而是直接：

```
_body[KEY_METHOD]_body[KEY_PARAMS]
```

进行操作。

这样带来几个好处。

第一。

不会维护两份数据。

否则：

```
成员变量
↓
Json对象
```

需要不断同步。

第二。

序列化更简单。

直接：

```
serialize(_body)
```

即可。

第三。

新增字段成本非常低。

例如以后增加：

```
timeout
```

只需要：

```
_body[KEY_TIMEOUT]
```

即可。

不需要修改序列化代码。

整个 Message 更像：

> **Json 的业务封装层。**

而不是传统意义上的实体类。


## 6.8 为什么每个 Message 都实现 check()

刚开始看到源码时，我以为 `check()` 只是简单判断 Json 是否为空。

继续往下看发现并不是。

项目中**每一种 Message 都重写了 `check()`**，并且校验规则完全不同。

例如：

**RpcRequest** 校验：

```
KEY_METHOD    必须存在，并且是 String
KEY_PARAMS    必须存在，并且是 Object
```

只有两个字段全部合法，请求才能继续处理。

----------

而 **TopicRequest** 的校验更加复杂。

首先检查：

```
KEY_TOPIC_KEY
```

必须存在。

然后检查：

```
KEY_OPTYPE
```

必须存在。

最后还有一段非常值得学习的逻辑：

```
if(optype == TOPIC_PUBLISH)
{    
	检查 KEY_TOPIC_MSG
}
```

也就是说：

**只有发布消息的时候，才要求 TopicMsg 存在。**

如果只是：

-   创建 Topic
-   删除 Topic
-   订阅 Topic
-   取消订阅

根本不需要消息内容。

所以源码不是统一检查所有字段，而是**根据业务类型动态校验字段**。

这个设计非常合理。

----------

再看 **ServiceRequest**。

它更有意思。

源码首先检查：

```
KEY_METHODKEY_OPTYPE
```

随后：

```
if(optype != SERVICE_DISCOVERY)
{    
	检查 Host
}
```

什么意思？

说明：

**服务发现请求，不需要携带主机地址。**

因为：

Caller

↓

Registry

↓

我要查：

```
method
↓
有哪些 Provider
```

这时候：

Caller 自己根本没有 Host。

因此不用检查。

而：

服务注册

服务上线

服务下线

这些请求。

必须告诉注册中心：

```
IPPORT
```

否则注册中心不知道是哪一个服务节点。

所以源码这里只针对：

非 Discovery

检查：

```
KEY_HOST
↓
IP
↓
PORT
```

这是完全符合业务逻辑的。

----------

## 为什么不用 Dispatcher 检查

这里其实体现了一个设计思想。

Dispatcher 不知道：

```
RpcRequest
TopicRequest
ServiceRequest
```

里面到底应该有哪些字段。

如果所有检查都写 Dispatcher：

以后新增一种 Message。

Dispatcher 又得改。

现在：

每一个 Message 自己负责：

```
check()
```

Dispatcher 只负责：

```
msg->check();
```

成功继续。

失败返回。

这样：

新增 Message。

只需要：

实现自己的：

```
check()
```

Dispatcher 完全不用修改。

这就是典型的：

> **行为跟数据放在一起。**

----------

## 6.9 为什么 MessageFactory 存在

这一块我觉得也是源码里面一个比较漂亮的设计。

源码最后实现了：

```
class MessageFactory
```

最重要的接口：

```
create(MType)
```

内部：

```
switch(MType)
{    
	...
}
```

返回：

对应 Message。

例如：

```
REQ_RPC
↓
RpcRequest
```

```
RSP_RPC
↓
RpcResponse
```

```
REQ_SERVICE
↓
ServiceRequest
```

……

----------

为什么要这样？

因为 Protocol 收到网络数据以后。

知道的只有：

```
MType
```

例如：

```
REQ_RPC
```

但是：

Protocol 根本不知道：

```
new RpcRequest
```

还是：

```
new TopicRequest
```

所以：

Protocol 只需要：

```
MessageFactory::create(mtype)
```

得到：

```
BaseMessage::ptr
```

随后：

```
unserialize()
```

即可。

整个 Protocol 根本不用依赖：

```
RpcRequestTopicRequestServiceRequest
```

这些具体类。

----------

### 为什么 Factory 返回 BaseMessage

源码没有：

```
RpcRequest*
```

而是：

```
BaseMessage::ptr
```

原因也很简单。

因为：

Protocol

Dispatcher

Requestor

全部工作在：

```
BaseMessage
```

这一层。

只有真正进入业务 Handler。

例如：

RPC Handler

才会：

```
dynamic_pointer_cast<RpcRequest>()
```

恢复真正类型。

整个框架因此始终依赖抽象。

而不是依赖具体实现。

----------

### MessageFactory 为什么还有模板 create()

源码还有一个模板版本：

```
template<typename T, typename... Args>
static std::shared_ptr<T> create(...)
```

它和：

```
create(MType)
```

用途不同。

前者：

用于运行过程中：

```
收到 MType
↓
动态创建消息
```

后者：

用于业务代码主动创建对象。

例如：

```
auto req =MessageFactory::create<RpcRequest>();
```

这样以后即使对象创建方式发生变化。

业务代码也不用修改。

仍然统一通过 Factory 创建。

----------
## 6.10 Message 为什么全部使用 Getter / Setter

阅读源码可以发现，所有消息类几乎都没有直接暴露 `_body`。

而是统一提供类似：

```
method()
setMethod()
params()
setParams()
host()
setHost()
```

这样的接口。

刚开始我觉得只是普通的封装。

继续往下分析后发现，它实际上有三个作用。

----------

### ① 屏蔽 Json 实现

业务层调用：

```
req->setMethod("Login");
```

而不是：

```
_body["method"] = "Login";
```

这样，业务层根本不知道内部使用的是 `Json::Value`。

以后如果底层改成：

-   Protobuf
-   FlatBuffers
-   MessagePack

业务代码完全不用修改。

修改的只有 Message 内部实现。

因此，**Getter / Setter 实际上屏蔽了底层数据存储方式。**

----------

#### ② 保证字段访问统一

整个项目所有地方读取 Method 都是：

```
req->method();
```

而不是：

```
_body["method"]
```

这样整个项目只有 Message 自己知道：

```
KEY_METHOD
```

到底叫什么。

如果以后协议字段名字发生变化，例如：

```
method
↓
rpc_method
```

修改 Message 即可。

其它模块完全不受影响。

----------

#### ③ 后续方便增加校验

目前 Setter 基本只是赋值。

例如：

```
setMethod(...)
```

以后完全可以增加：

```
长度检查
合法字符检查
空字符串检查
```

调用方仍然不用修改。

因此，字段访问全部通过接口，也是后续扩展性的体现。

----------

## 6.11 Message 的继承关系分析

整个 Message 系统虽然类很多，但继承关系其实非常清晰。

可以整理成下面这一张图。

```
                      BaseMessage                           
		                   │                     
                      JsonMessage                    
                      /            \             
                JsonRequest     JsonResponse             
              /      |   \      /      |    \      
         RpcReq  TopicReq ... RpcRsp TopicRsp ...
```

每一层都承担不同职责。

**BaseMessage**

负责统一消息抽象。

定义所有消息共同接口。

----------

**JsonMessage**

负责：

-   Json 序列化
-   Json 反序列化
-   保存 `_body`

它解决的是：

> **消息如何保存。**

----------

**JsonRequest**

抽象所有请求公共行为。

----------

**JsonResponse**

抽象所有响应公共行为。

例如：

-   `RCode`
-   `check()`

它解决的是：

> **响应有哪些公共属性。**

----------

**具体 Message**

例如：

```
RpcRequest
TopicRequest
ServiceRequest
```

只负责：

自己的业务字段。

例如：

Method、

Params、

Host、

Topic、

Message……

因此，每一层职责都非常明确，没有出现职责混乱的问题。

----------

## 6.12 Message 模块与其它模块的关系

整个项目中，Message 是所有模块共同依赖的数据模型。

可以表示为：

```
		              Message        
                ┌────────┼────────┐        
                │        │        │    
            Protocol Dispatcher Requestor        
                │        │        │        
                └────────┼────────┘                 
				         │              
				      Business
```

可以看到：

**Protocol**

负责：

Message ↔ 字节流。

----------

**Dispatcher**

负责：

Message → Handler。

----------

**Requestor**

负责：

Message 生命周期管理。

----------

**业务模块**

负责：

真正处理 Message。

因此，Message 本身并不负责：

-   网络通信；
-   请求管理；
-   消息分发。

它只是整个框架的数据载体。

# 7. Protocol 协议设计

----------

## 7.1 为什么需要 Protocol

刚开始学习 Socket 编程时，我认为：

```
send(sockfd, buf, len);
recv(sockfd, buf, len);
```

发送和接收数据就够了。

后来真正实现 RPC 才发现：

**TCP 传输的是字节流(Byte Stream)，而不是消息(Message)。**

例如客户端连续发送：

```
RpcRequest1
RpcRequest2
RpcRequest3
```

经过 TCP 后，服务端收到的可能变成：

```
RpcRequest1RpcReq
```

下一次：

```
uest2RpcRequest3
```

甚至：

```
RpcRequest1RpcRequest2RpcRequest3
```

全部粘在一起。

TCP 根本不知道：

> "这里是一条消息结束。"

所以：

**必须自己定义消息边界(Message Boundary)。**

Protocol 模块就是解决这个问题。

----------

## 7.2 Protocol 在整个框架中的位置

阅读源码以后，可以把整个调用链整理出来：

```
Message    
│
serialize()    
│
Json String    
│
Protocol    
│
TcpConnection    
│
Muduo
```

收到数据时：

```
Muduo
↓
Buffer
↓
Protocol
↓
Message
↓
Dispatcher
```

因此：

Protocol 位于：

> **网络层与消息层之间。**

它既不关心业务。

也不关心网络。

它只负责：

> **消息如何在 TCP 中正确传输。**

----------

# 7.3 为什么不用 Json 直接发送

很多人第一次做 RPC 都会想到：

```
socket.send(json.dump());
```

其实这样是不够的。

假设发送：

```
{    
	"method":"login"
}
```

服务端收到：

```
{    
	"method"
```

怎么办？

继续等？

还是已经结束？

根本不知道。

所以：

**Json 只负责数据格式。**

Protocol 负责：

> **告诉对方这一条消息什么时候结束。**

因此：

Json 与 Protocol 解决的是两个不同的问题。

----------

# 7.4 Length-Value 协议

阅读源码以后可以看到。

项目采用的是：

```
Length+Body
```

协议。

发送时：

```
+------------+----------------------+
| Length(4B) | Json String          |
+------------+----------------------+
```

例如：

Body：

```
{"method":"login"}
```

长度：

```
18
```

真正发送：

```
		18
		 ↓
{"method":"login"}
```

这样：

服务端首先读取：

```
Length
```

知道：

后面还有多少字节。

然后继续读取完整 Body。

这样：

无论发生：

-   粘包
-   半包

都能够恢复完整消息。

----------

## 7.5 为什么长度放前面

长度放在消息最前面。

这样 Protocol 收到 Buffer 后。

第一件事就是：

```
读取 Length
```

然后：

判断：

```
Buffer
↓
是否达到 Length
```

如果：

没有。

继续等待。

如果：

已经达到。

说明：

一条完整消息已经到达。

整个流程如下：

```
收到 Buffer
↓
读取 Length
↓
长度够吗？
↓
否继续接收
↓
是解析 Body
```

因此：

Protocol 不需要猜：

> 消息什么时候结束。

而是：

根据长度准确判断。

----------

## 7.6 Protocol 为什么不关心 Message

Protocol 并不知道：

```
RpcRequest
RpcResponse
TopicRequest
```

它只知道：

```
BaseMessage
```

发送时：

调用：

```
serialize()
```

得到：

```
Json String
```

接收时：

得到：

```
Json String
```

随后：

交给：

```
MessageFactory
```

恢复真正对象。

因此：

Protocol 从始至终都没有：

```
new RpcRequest
```

也没有：

```
dynamic_cast
```

它始终工作在：

```
字符串
```

这一层。

职责非常单一。

----------

# 7.7 Protocol 与 Message 的关系

整个数据流如下：

```
RpcRequest
↓
serialize()
↓
Json
↓
Protocol
↓
Length + Body
↓
TCP
↓
Length + Body
↓
Protocol
↓
Json
↓
MessageFactory
↓
RpcRequest
```

可以发现：

Protocol 根本不知道：

```
method
params
host
```

这些字段。

它唯一负责的是：

> **保证消息完整传输。**

而：

消息里面是什么内容。

完全交给 Message 模块。

## 7.8 Protocol 模块的设计优点

整个 Protocol 模块看下来，它最大的特点并不是采用了 Length-Value 协议，而是**将协议处理与业务处理彻底分离**。

整个数据流可以表示为：

```
业务对象    
│
▼
Message    
│    
▼
Protocol    
│    
▼
Muduo    
│    
▼
TCP
```

收到数据以后：

```
TCP    
│    
▼
Muduo Buffer    
│    
▼
Protocol    
│    
▼
Message    
│    
▼
Dispatcher
```

可以发现：

Protocol 始终位于 Message 与网络之间。

它不知道：

-   RpcRequest
-   RpcResponse
-   ServiceRequest
-   TopicRequest

里面到底有哪些字段。

它唯一关心的是：

> **如何把一段完整的数据发送出去，以及如何从 Buffer 中恢复一条完整消息。**

因此，Protocol 与业务模块之间保持了较低的耦合。

----------

## Protocol 为什么不直接调用业务代码

阅读整个项目可以发现。

Protocol 完成消息解析以后，并不会直接调用：

```
RpcService
TopicService
RegistryService
```

而是只恢复：

```
BaseMessage
```

随后交给 Dispatcher。

这样做有两个好处。

第一。

Protocol 不需要知道系统有哪些业务模块。

第二。

以后新增新的消息类型。

Protocol 不需要修改。

整个消息处理流程始终保持：

```
Protocol
↓
Dispatcher
↓
Handler
```

协议层只负责恢复消息对象，而消息如何处理，由 Dispatcher 决定。

----------

## Protocol 为什么可以独立替换

整个项目中。

Protocol 对外暴露的是统一接口。

发送时：

输入：

```
BaseMessage
```

输出：

```
Buffer
```

接收时：

输入：

```
Buffer
```

输出：

```
BaseMessage
```

因此：

如果以后需要把：

```
Length-Value
```

修改为：

```
HTTPWebSocket
自定义二进制协议
```

理论上：

Message、

Dispatcher、

Requestor

都无需修改。

只需要重新实现 Protocol 即可。

这也是项目采用分层设计带来的优势。

----------

### Protocol 与 Message 的职责边界

这一章最后，再总结一下 Message 与 Protocol 的关系。

**Message** 解决的是：

> 一条消息包含哪些数据。

例如：

-   Method
-   Params
-   Host
-   Topic

这些都属于 Message。

----------

**Protocol** 解决的是：

> 一条消息如何在 TCP 上传输。

例如：

-   消息长度；
-   数据封包；
-   数据拆包；
-   消息完整性判断。

因此，两者虽然都会参与数据发送，但关注点完全不同。

整个框架的数据流可以概括为：

```
业务对象    
│    
▼
Message（组织业务数据）    
│    
▼
Protocol（组织传输格式）    
│    
▼
TCP（负责传输）
```

每一层都只负责自己的工作，没有职责交叉。
--
# 8. 同步与异步 RPC 调用

## 8.1 为什么需要多种调用方式

阅读整个项目以后可以发现，`RpcCaller` 并没有只提供一种调用接口，而是支持：

-   同步调用（Sync）
-   Future 异步调用
-   Callback 异步调用

刚开始我认为只是为了让接口更丰富。

后来结合整个 Requestor 的实现才发现，这三种调用方式解决的是**不同的业务需求**。

例如：

同步调用：

```
发请求    
│
等待结果    
│
继续执行
```

整个线程会一直阻塞。

调用方式最简单，也最接近普通函数调用。

----------

Future 调用：

```
发请求    
│
返回 Future    
│
继续执行其它任务    
│
future.get()
```

发送请求以后，不需要立即等待结果。

真正需要结果时，再主动获取。

因此能够减少线程空闲等待时间。

----------

Callback 调用：

```
发请求    
│
继续执行    
│
收到响应    
│
自动执行 Callback
```

整个过程中，不需要主动等待，也不需要主动获取结果。

收到响应以后，由 Requestor 自动回调业务函数。

因此更加符合事件驱动的思想。

----------

## 8.2 三种调用方式的共同流程

虽然接口不同，但结合源码来看，三种调用方式实际上共用同一套请求处理流程。

统一流程如下：

```
业务层    
│    
▼
创建 RpcRequest    
│    
▼
Requestor::newDescribe()    
│    
▼
保存 RequestDescribe    
│    
▼
发送 Message    
│    
▼
等待服务端处理    
│    
▼
收到 Response    
│    
▼
Requestor 根据 rid 找到对应请求
```

直到这里，三种调用方式完全一致。

真正不同的是：

**收到响应以后如何通知业务层。**

----------

## 8.3 同步调用的实现思路

同步调用最大的特点是：

> **调用线程必须等待结果返回。**

结合项目来看，同步调用并没有重新实现一套发送流程，而是仍然通过 Requestor 管理请求。

区别在于：

发送请求以后，当前线程不会继续执行，而是等待对应响应到达。

收到响应以后：

Requestor 找到对应请求。

将结果返回给同步调用接口。

随后：

同步调用结束。

整个流程如下：

```
发送请求      
│      
▼
线程阻塞等待      
│      
▼
收到响应      
│      
▼
返回 RpcResponse
```

同步调用实现简单，代码逻辑清晰，但线程利用率较低。

----------

## 8.4 Future 异步调用的实现思路

Future 调用是在同步调用基础上的进一步封装。

发送请求时：

RequestDescribe 中保存：

```
promise
```

随后：

返回：

```
future
```

给业务层。

业务线程继续执行其它逻辑。

收到响应以后：

Requestor 找到对应 RequestDescribe。

调用：

```
promise.set_value(...)
```

之前返回出去的：

```
future
```

立即变为 Ready。

业务层调用：

```
future.get()
```

即可获得结果。

整个过程中：

发送流程没有变化。

变化的是：

**响应通知方式。**

----------

## 8.5 Callback 异步调用的实现思路

Callback 调用更加直接。

发送请求时：

RequestDescribe 保存：

```
callback
```

收到响应以后：

Requestor 找到对应请求。

直接执行：

```
callback(response)
```

整个过程中：

没有 Future。

没有阻塞。

没有主动获取结果。

真正体现的是：

> **事件发生以后通知业务层。**

因此 Callback 更适合：

收到结果立即处理，而不是等待结果的场景。

----------

## 8.6 RequestDescribe 为什么能够统一三种调用方式

阅读 Requestor 的源码以后，我认为这是整个客户端设计中最巧妙的一点。

虽然：

-   同步调用；
-   Future；
-   Callback；

接口完全不同。

但是：

Requestor 并没有为三种调用方式分别维护三套请求管理逻辑。

而是统一保存：

```
RequestDescribe
```

其中根据调用方式：

保存不同的信息。

例如：

同步调用：

保存同步等待所需的数据。

Future：

保存 `promise`。

Callback：

保存回调函数。

因此：

Requestor 始终只维护：

```
rid
↓
RequestDescribe
```

这一张请求表。

收到响应以后：

也是统一：

```
rid
↓
RequestDescribe
↓
根据调用方式通知业务层
```

整个请求生命周期保持一致，仅在最后一步采用不同的通知策略。

这也是整个客户端调用框架能够同时支持三种调用方式的重要原因。

## 8.7 三种调用方式的优缺点分析

整个项目同时支持同步、Future、Callback 三种调用方式，并不是因为一种方式不好，而是它们分别适用于不同的业务场景。

----------

### 同步调用

同步调用最大的优点就是：

**调用方式最简单。**

对于业务层来说：

```
调用 RPC
↓
等待返回
↓
继续执行
```

整个流程与普通函数调用几乎一致。

因此代码逻辑清晰，调试也比较方便。

但是，它最大的缺点也很明显。

请求发送以后：

当前线程会一直等待。

在等待期间：

线程不能继续处理其它任务。

如果：

RPC 响应较慢。

或者：

一次需要调用多个服务。

线程利用率就会比较低。

因此，同步调用更适合：

-   请求耗时较短；
-   调用链简单；
-   必须立即获得结果。

----------

### Future 调用

Future 的特点是：

> **把等待结果这件事情延后。**

发送请求以后：

立即返回 Future。

业务线程可以继续执行其它任务。

真正需要结果时：

再调用：

```
future.get();
```

这样：

网络等待时间与业务计算时间可以部分重叠。

例如：

```
发送 RPC      
│      
├───────────────┐      
│               │      
▼               ▼
继续计算      服务端处理请求      
│               │      
└──────┬────────┘             
	   ▼        
	future.get()
```

因此：

Future 能够提高线程利用率。

但是：

Future 本质上仍然可能阻塞。

因为：

```
future.get();
```

如果结果没有回来。

线程仍然需要等待。

所以：

Future 属于：

> **延迟等待。**

而不是：

完全无等待。

----------

### Callback 调用

Callback 的特点更加明显。

业务层发送请求以后。

整个线程立即返回。

收到响应以后：

由 Requestor 主动执行：

```
callback(response)
```

整个过程中：

业务层根本不需要：

```
future.get()等待轮询
```

因此：

线程利用率最高。

事件响应也最快。

但是：

Callback 最大的问题是：

业务逻辑容易分散。

例如：

```
发送请求
↓
业务代码 A
↓
Callback
↓
业务代码 B
```

业务流程会被拆成多个函数。

如果：

回调层数不断增加。

容易形成：

> Callback Hell（回调地狱）。

虽然本项目只有一层 Callback。

不存在这个问题。

但是这是 Callback 模式本身需要注意的一点。

----------

## 8.8 为什么项目同时保留三种接口

刚开始看到源码时。

我觉得：

既然有 Callback。

为什么还要 Future？

后来整理整个调用流程以后发现。

实际上：

三种接口解决的是三类完全不同的需求。

同步：

更接近普通函数。

适合：

简单业务。

----------

Future：

适合：

需要并发多个 RPC。

例如：

```
RPC ARPC BRPC C
```

全部发送。

最后统一：

```
future.get();
```

等待结果。

----------

Callback：

适合：

收到结果以后立即处理。

整个业务流程更加偏向事件驱动。

因此：

项目保留三种接口，并不是重复实现。

而是为了适配不同的业务场景。

----------

## 8.9 这一章最大的设计思想

如果只看接口。

很容易认为：

同步。

Future。

Callback。

分别实现了三套 RPC。

实际上并不是。

阅读 Requestor 源码以后可以发现。

真正变化的只有：

```
收到响应以后
↓
如何通知业务层
```

其它流程：

```
创建请求
↓
保存 RequestDescribe
↓
发送 Message
↓
收到 Response
↓
根据 rid 找到 RequestDescribe
```

全部一致。

因此：

整个项目真正做到的是：

> **统一请求生命周期，不同响应通知方式。**

这样：

既避免了重复代码。

也保证了三种调用方式拥有一致的行为。

如果以后增加新的调用模式。

例如：

```
Coroutine
```

理论上。

仍然可以复用：

整个 Requestor 请求管理流程。

只需要增加一种新的通知方式即可。

---
# 9. Dispatcher 消息分发机制

## 9.1 为什么需要 Dispatcher

整个框架中，不同类型的消息最终都会经过网络层到达本地。

例如：

-   RpcRequest
-   RpcResponse
-   ServiceRequest
-   ServiceResponse
-   TopicRequest
-   TopicResponse

如果没有 Dispatcher，那么收到一条消息以后，只能这样处理：

```
if (mtype == REQ_RPC)
{    
	...
}
else if (mtype == RSP_RPC)
{    
	...
}
else if (mtype == REQ_SERVICE)
{    
	...
}
...
```

随着消息类型不断增加，这段代码会越来越长。

而且每增加一种新的 Message，都需要修改这里。

整个模块越来越难维护。

因此项目引入 Dispatcher，将：

> **消息接收**

与

> **消息处理**

彻底分离。

收到消息以后。

Dispatcher 只负责：

```
根据 MessageType
↓
找到对应 Handler
↓
调用 Handler
```

至于 Handler 内部怎么处理。

Dispatcher 完全不关心。

----------

## 9.2 Dispatcher 的职责

结合整个项目来看。

Dispatcher 并不负责：

-   网络通信；
-   Json 解析；
-   请求生命周期管理。

它真正负责的是：

**根据消息类型，把消息分发给正确的处理函数。**

整个流程如下：

```
收到 BaseMessage        
│        
▼
读取 MessageType        
│        
▼
查找 Handler        
│        
▼
执行 Handler
```

整个过程中：

Dispatcher 始终面对的是：

```
BaseMessage
```

而不是具体业务对象。

真正恢复具体类型，是在 Handler 内部完成。

----------

## 9.3 为什么 Dispatcher 只依赖 BaseMessage

阅读整个框架以后，可以发现：

Protocol 返回：

```
BaseMessage::ptr
```

Requestor 保存：

```
BaseMessage::ptr
```

Dispatcher 接收：

```
BaseMessage::ptr
```

整个消息处理链始终依赖：

```
BaseMessage
```

这样做最大的好处就是：

Dispatcher 根本不用包含：

```
RpcRequest
RpcResponse
TopicRequest
ServiceRequest
```

这些具体类型。

Dispatcher 唯一关心的是：

```
这是什么类型的消息？
```

至于：

```
Message 里面有哪些字段？
```

交给真正的 Handler 去处理。

因此：

Dispatcher 成为了整个框架中耦合度最低的模块之一。

----------

## 9.4 Handler 为什么采用注册机制

项目没有把所有处理逻辑都写进 Dispatcher。

而是采用：

> **注册（Register）**

的方式。

整个过程如下：

程序初始化：

```
RpcHandler
↓
注册 Dispatcher
```

```
TopicHandler
↓
注册 Dispatcher
```

```
ServiceHandler
↓
注册 Dispatcher
```

之后。

收到消息：

Dispatcher 根据 MessageType。

直接找到已经注册好的 Handler。

整个流程如下：

```
初始化      
│
▼
注册 Handler      
│      
▼
收到 Message      
│      
▼
Dispatcher 查找 Handler      
│      
▼
执行 Handler
```

这种方式避免了 Dispatcher 内部维护大量业务逻辑。

新增一种消息时。

只需要：

新增 Handler。

然后完成注册。

Dispatcher 本身无需修改。

符合开闭原则。

----------

## 9.5 Dispatcher 为什么不用 switch

很多项目都会采用：

```
switch(mtype)
{
	case 
	...
```

本项目采用 Handler 注册以后。

Dispatcher 不需要知道：

有哪些 Message。

也不需要维护：

```
switch
```

或者：

```
if else
```

Dispatcher 内部维护的只是：

```
MessageType
↓
Handler
```

这一层映射关系。

收到消息以后：

直接查找。

然后执行。

整个分发过程更加统一。

同时也降低了 Dispatcher 与业务模块之间的耦合。

## 9.6 Handler 注册机制分析

Dispatcher 能够完成消息分发，真正依赖的是初始化阶段完成的 **Handler 注册**。

整个框架并不是收到消息以后再去判断：

```
这是 RPC 吗？
这是 Service 吗？
这是 Topic 吗？
```

而是在程序启动时，就已经建立好了消息类型与处理函数之间的对应关系。

因此，真正收到消息以后，Dispatcher 做的事情其实非常简单：

```
读取 MessageType        
│        
▼
查找对应 Handler        
│        
▼
执行 Handler
```

整个过程中，Dispatcher 并不需要关心 Handler 内部的实现。

----------

### 为什么采用注册，而不是直接创建 Handler

如果 Dispatcher 自己负责创建 Handler，例如：

```
RpcHandler rpc;
ServiceHandler service;
TopicHandler topic;
```

那么 Dispatcher 就必须依赖所有业务模块。

以后新增：

```
HeartbeatHandler
```

Dispatcher 又需要继续修改。

整个 Dispatcher 会越来越庞大。

项目采用注册机制以后。

Dispatcher 并不知道有哪些 Handler。

它只知道：

> **有人注册，我负责保存。**

至于是谁注册。

什么时候注册。

Dispatcher 都不用关心。

因此：

**对象创建与对象使用完全分离。**


### Dispatcher 与 Handler 的关系

整个关系可以表示成：

```
Dispatcher      
│      
│ 保存      
▼
MessageType      
│      
▼
Handler
```

Dispatcher 保存的不是：

```
RpcRequest
```

也不是：

```
RpcService
```

而是：

> **某一种 MessageType 对应一个处理函数。**

因此：

Dispatcher 本身没有任何业务逻辑。

它只是整个框架中的：

> **消息路由器（Router）。**

----------

## 9.7 Dispatcher 为什么能够支持扩展

阅读整个消息处理流程以后，我认为 Dispatcher 最大的优点就是：

**新增消息时，几乎不用修改已有代码。**

例如以后增加一种：

```
HeartbeatRequest
```

整个流程只需要：

```
新增 Message
↓
新增 Handler
↓
注册 Handler
```

Dispatcher 本身完全不用修改。

消息到达以后：

仍然按照：

```
MessageType
↓
Handler
```

完成分发。

整个流程保持不变。

因此，Dispatcher 很好地体现了：

> **对扩展开放，对修改关闭。**

这也是整个框架中最明显体现开闭原则的模块之一。

----------

## 9.8 Dispatcher 在整个框架中的位置

结合前面几章，可以把整个客户端到服务端的数据流重新整理一次。

```
RpcCaller      
│      
▼
Requestor      
│      
▼
Protocol      
│      
▼
TCP      
│      
▼
Protocol      
│      
▼
Dispatcher      
│      
▼
Handler      
│      
▼
业务模块
```

其中：

**Requestor**

负责：

> 请求发送。

----------

**Protocol**

负责：

> 数据封包、拆包。

----------

**Dispatcher**

负责：

> 找到正确 Handler。

----------

**Handler**

负责：

> 真正执行业务逻辑。

因此：

Dispatcher 实际上就是：

**网络层与业务层之间的一座桥梁。**

----------

## 9.9 Dispatcher 与 Requestor 的区别

这一点面试很容易问。

很多人容易把：

```
Dispatcher
Requestor
```

混在一起。

实际上两者职责完全不同。

**Requestor**

关注的是：

```
一个请求
↓
什么时候发送
↓
什么时候结束
↓
如何通知调用方
```

管理的是：

> **请求生命周期。**

----------

而 Dispatcher 关注的是：

```
收到消息
↓
交给谁处理
```

管理的是：

> **消息流向。**

因此：

Requestor 解决：

> **请求管理问题。**

Dispatcher 解决：

> **消息分发问题。**

两者虽然都会接触 Message。

但是关注点完全不同。

----------
## 9.10 Dispatcher 为什么能够降低模块耦合

回顾整个消息处理流程，可以发现 Dispatcher 并没有直接参与任何业务处理，它更像是整个框架中的一个**中转站**。

整个流程如下：

```
Protocol    
│    
▼
BaseMessage    
│    
▼
Dispatcher    
│    
▼
Handler    
│    
▼
Business
```

如果没有 Dispatcher。

那么 Protocol 收到消息以后，就必须知道：

-   RPC 请求应该交给谁；
-   服务注册应该交给谁；
-   Topic 消息应该交给谁。

这样 Protocol 将直接依赖：

```
RpcHandler
ServiceHandler
TopicHandler
```

网络层开始感知业务层。

整个框架的层次会越来越混乱。

加入 Dispatcher 后。

Protocol 只需要完成：

```
收到 Message
↓
交给 Dispatcher
```

至于：

最后由谁处理。

Protocol 完全不用关心。

因此：

Dispatcher 成为了网络层与业务层之间的隔离层。

----------

## 9.11 Dispatcher 的处理流程

结合整个框架，可以将 Dispatcher 的工作流程整理为：

```
Protocol 解包完成        
│        
▼
得到 BaseMessage        
│        
▼
读取 MessageType        
│        
▼
查找对应 Handler        
│        
▼
执行 Handler        
│        
▼
业务处理完成
```

可以发现。

Dispatcher 从头到尾只完成了一件事情：

> **根据消息类型完成路由。**

它不会：

-   修改消息内容；
-   解析业务字段；
-   管理请求生命周期；
-   维护网络连接。

因此整个模块保持了非常单一的职责。

----------

## 9.12 Dispatcher 的设计思想

阅读这一部分源码以后，我认为 Dispatcher 主要体现了三个设计思想。

----------

### （1）依赖抽象，而不是具体业务

Dispatcher 工作过程中始终面对：

```
BaseMessage
```

而不是：

```
RpcRequest
TopicRequest
ServiceRequest
```

因此。

新增任何新的 Message 类型。

Dispatcher 都不需要修改自己的整体流程。

----------

### （2）职责分离

整个消息处理过程被拆成了多个模块。

```
Protocol
↓
Dispatcher
↓
Handler
```

其中：

Protocol：

负责恢复 Message。

Dispatcher：

负责找到 Handler。

Handler：

负责执行业务逻辑。

每个模块只完成自己的工作。

----------

### （3）符合开闭原则

如果以后增加：

```
HeartbeatRequest
```

整个流程仍然保持：

```
新增 Message
↓
新增 Handler
↓
注册 Handler
```

Dispatcher 不需要增加：

```
if ...switch ...
```

整个框架的扩展成本比较低。

----------
# 10. 服务注册与发现（Registry）

## 10.1 为什么需要注册中心

刚开始学习 RPC 时，我一直有一个疑问。

客户端调用：

```
UserService.Login()
```

但是：

客户端怎么知道：

```
UserService
↓
运行在哪台机器？
```

如果直接把 IP 写死：

```
192.168.1.100:8000
```

虽然能够工作。

但是问题很多。

例如：

服务器迁移。

```
192.168.1.100
↓
192.168.1.110
```

所有客户端都需要修改。

如果：

新增一台 Provider。

客户端也完全不知道。

因此：

客户端不能直接保存：

```
IPPORT
```

而应该保存：

```
ServiceName
```

真正调用之前。

先去询问：

> **谁能够提供这个服务？**

Registry 就是解决这个问题。

----------

## 10.2 Registry 在整个框架中的位置

整个调用流程如下：

```
Provider      
│      
▼
注册服务      
│      
▼
Registry      
▲      
│
服务发现      
│      
▼
Caller
```

Provider 启动以后。

首先向 Registry 注册：

```
Method
↓
Host
```

Caller 调用 RPC 时。

首先询问 Registry：

```
Method
↓
有哪些 Provider
```

Registry 返回可用节点以后。

Caller 再真正建立连接。

因此：

Registry 并不参与：

RPC 调用。

它只负责：

> **告诉客户端应该连接谁。**

----------

## 10.3 为什么采用服务发现，而不是直接连接

如果没有 Registry。

整个流程就是：

```
Caller
↓
固定 IP
↓
Provider
```

这种方式的问题在于：

Provider 一旦变化。

Caller 全部失效。

而采用 Registry 后。

流程变成：

```
Caller
↓
Registry
↓
Provider
```

客户端永远只认识：

Registry。

Provider 如何变化。

客户端都不用修改。

因此：

Registry 成为了：

**服务与客户端之间的解耦层。**

----------

## 10.4 Registry 保存什么信息

阅读整个项目以后。

Registry 保存的并不是：

```
Connection
```

也不是：

```
Socket
```

它真正维护的是：

```
Method
↓
Host
```

也就是说：

对于每一个服务。

Registry 都知道：

```
哪些机器能够提供这个服务。
```

因此：

服务发现实际上就是：

```
Method
↓
Host 列表
```

的一次查询过程。

----------

## 10.5 为什么一个 Method 对应多个 Host

这一点是整个 Registry 最重要的设计之一。

如果：

```
Login()
```

只有：

```
Host A
```

那么：

Host A 宕机。

整个服务立即不可用。

因此：

一个 Method。

可以对应多个 Provider。

例如：

```
Login()
↓
Host1
Host2
Host3
```

Caller 查询以后。

再选择：

其中一个 Host。

真正建立连接。

因此：

Registry 不负责：

选择哪个 Provider。

它负责：

告诉客户端：

有哪些 Provider。

至于最终选择谁。

交给：

负载均衡模块完成。

因此：

Registry 与 LoadBalance 又完成了一次职责分离。

----------

## 10.6 服务注册流程

整个 Provider 启动以后。

首先建立与 Registry 的连接。

随后发送：

```
ServiceRequest
```

其中包含：

```
Method
Host
Operation
```

Registry 收到以后。

将：

```
Method
↓
Host
```

加入自己的服务表。

至此。

这个 Provider 就正式上线。

之后：

客户端便可以查询到该节点。

## 10.7 为什么注册信息中保存的是 Method，而不是 Service

刚开始看到整个服务注册流程时，我最开始的理解是：

Registry 应该维护：

```
ServiceName
↓
Host
```

后来结合整个项目重新整理以后发现。

本项目实际注册的是：

```
Method
↓
Host
```

也就是说。

客户端查询时。

也是根据：

```
Method
```

查询能够提供该方法的 Provider。

这种设计意味着：

Registry 并不关心：

```
这个 Provider 属于哪个 Service。
```

它只关心：

> **谁能够处理这个 Method。**

因此：

整个服务发现流程更加直接。

Caller 需要调用哪个方法。

Registry 就返回哪个方法对应的 Provider。

整个查询过程非常简单。

----------

## 10.8 Provider 上线与下线

Registry 保存的服务信息并不是永久存在。

随着 Provider 生命周期变化。

Registry 维护的数据也会不断更新。

整个生命周期可以整理成：

```
Provider 启动      
│
▼
发送注册请求      
│      
▼
Registry 添加服务      
│      
▼
客户端可发现      
│      
▼
Provider 下线      
│      
▼
发送注销请求      
│      
▼
Registry 删除服务
```

因此。

Registry 始终保存的是：

> **当前可用 Provider。**

而不是：

所有历史 Provider。

这样。

客户端查询到的结果始终保持最新。

----------

## 10.9 为什么 Registry 不直接建立连接

很多人第一次设计 RPC 时都会想到。

Registry 已经知道：

所有 Provider。

为什么不直接：

```
Caller
↓
Registry
↓
Provider
```

由 Registry 转发请求？

实际上。

本项目并没有这样设计。

原因很简单。

Registry 的职责只是：

```
服务管理
```

真正的 RPC 请求。

仍然由：

```
Caller
↓
Provider
```

直接建立连接。

这样有两个好处。

第一。

Registry 不参与业务调用。

不会成为：

所有 RPC 请求的数据通道。

第二。

Registry 的压力非常小。

它只负责：

-   注册；
-   注销；
-   查询。

真正的数据流量全部发生在：

Caller 与 Provider 之间。

因此。

Registry 更像：

> **电话簿。**

告诉客户端：

应该联系谁。

真正打电话。

还是客户端自己完成。

----------

## 10.10 Registry 与 Requestor 的关系

这两个模块也很容易混淆。

实际上职责完全不同。

**Registry**

负责：

```
Method
↓
Host
```

回答的是：

> **去哪里调用？**

----------

而：

**Requestor**

负责：

```
Host
↓
发送 RPC
↓
等待响应
```

回答的是：

> **如何完成调用？**

因此。

整个流程可以表示为：

```
Caller    
│    
▼
Registry    
│
获得 Host    
│    
▼
Requestor    
│
发送 RPC    
│    
▼
Provider
```

Registry 解决：

> **服务定位。**

Requestor 解决：

> **远程调用。**

两者前后衔接。

但完全属于两个不同阶段。

----------

## 10.11 Registry 与负载均衡的关系

阅读整个框架以后。

可以发现。

Registry 返回的是：

```
Host 集合
```

例如：

```
Login
↓
Host1
Host2
Host3
```

但是。

最终到底选择：

```
Host1
```

还是：

```
Host2
```

并不是 Registry 决定。

而是：

LoadBalance。

因此。

Registry 的职责到：

```
返回可用节点
```

就结束了。

后续：

如何选择节点。

交给负载均衡模块。

整个框架再次完成：

职责拆分。

----------
## 10.13 Registry 为什么没有保存 Connection

刚开始看整个项目的时候，我一直以为 Registry 应该保存：

```
Method    
│    
▼
TcpConnection
```

后来重新整理整个调用流程以后发现。

Registry 根本没有必要维护 Provider 的连接对象。

原因有两个。

第一。

Registry 与 Provider 的连接，仅用于：

-   服务注册；
-   服务注销；
-   服务发现。

真正的业务调用发生时。

客户端会重新与对应 Provider 建立通信。

因此。

Registry 并不是业务通信节点。

----------

第二。

如果 Registry 长期维护所有 Provider 的业务连接。

那么：

整个系统所有 RPC 数据都会经过 Registry。

Registry 将成为：

整个系统最大的性能瓶颈。

而项目采用的是：

```
Caller      
│      
▼
Provider
```

直接通信。

Registry 只负责：

```
告诉 Caller：Provider 在哪里。
```

整个数据流不会经过 Registry。

因此：

Registry 的职责始终保持非常纯粹。

----------

## 10.14 Registry 为什么能够支持动态扩容

RPC 项目真正运行以后。

Provider 的数量通常不是固定的。

例如：

刚开始：

```
Login
↓
Host1
```

后来由于访问量增加。

新增：

```
Host2
Host3
```

整个过程中。

Caller 并没有修改任何代码。

原因就在于：

Caller 每次调用之前。

都通过 Registry 获取当前可用节点。

因此。

Registry 返回的数据始终反映当前 Provider 状态。

新的 Provider 注册成功以后。

下一次服务发现。

客户端自然就能够查询到新的节点。

整个扩容过程对业务层透明。

----------

## 10.15 Registry 为什么能够支持故障恢复

动态扩容解决的是：

新增节点。

另外一个问题就是：

节点故障。

例如：

```
Login
↓
Host1
Host2
Host3
```

其中：

```
Host2
```

发生故障。

如果 Registry 仍然返回：

```
Host2
```

那么：

客户端仍然可能选择这个节点。

最终导致 RPC 调用失败。

因此。

Provider 下线以后。

Registry 必须同步更新自己的服务信息。

这样。

下一次查询时。

客户端只能得到：

```
Host1
Host3
```

整个服务发现过程始终保持一致。

因此。

Registry 保存的并不是：

历史 Provider。

而是：

**当前可用 Provider。**

----------

## 10.16 Registry 在整个框架中的作用

整理整个项目以后。

我觉得 Registry 更像：

整个 RPC 框架中的：

> **服务目录（Service Directory）。**

它回答的问题只有一个：

```
这个 Method目前有哪些 Provider 可以处理？
```

除此之外。

它不会：

-   执行业务；
-   转发请求；
-   维护请求生命周期；
-   决定最终选择哪个节点。

整个职责可以表示为：

```
Registry    
│    
├── 保存服务信息    
├── 更新服务信息    
└── 提供服务查询
```

因此。

Registry 是整个框架中的：

**控制面（Control Plane）**。

真正的数据流：

始终发生在：

```
Caller
↓
Provider
```

之间。

----------

## 10.17 本章设计总结

阅读 Registry 模块以后，我最大的收获有两点。

第一。

服务注册中心最大的意义，并不是保存：

```
IPPORT
```

而是把：

**服务名称（或 Method）**

与

**服务节点（Host）**

建立映射关系。

这样。

客户端只关心：

> 我要调用哪个服务。

至于：

服务部署在哪里。

有几个节点。

是否发生迁移。

全部由 Registry 负责管理。

----------

第二。

整个框架把：

```
服务定位↓节点选择↓远程调用
```

拆分到了三个不同模块。

分别对应：

```
Registry↓LoadBalance↓Requestor
```

每个模块都只负责一个问题。

模块之间通过清晰的接口进行协作。

整个架构层次非常明确。

----------
# 11. Channel 与连接管理

----------

## 11.1 为什么需要 Channel

阅读整个项目以后，我发现 `Channel` 的作用并不是发送 RPC 请求，而是**负责管理与远端节点之间的网络连接**。

如果没有 Channel，那么每一次 RPC 调用都需要经历：

```
创建 Socket
↓
建立 TCP 连接
↓
发送请求
↓
等待响应
↓
关闭连接
```

对于一次 RPC 来说，真正的业务数据可能只有几十字节，而建立 TCP 连接却需要完成三次握手。

如果每调用一次 RPC 都重新建立连接，那么大量时间都会消耗在连接建立与释放上，而不是业务处理本身。

因此，框架需要一种机制，将已经建立好的连接保存下来，在后续调用中继续使用。

这就是 Channel 存在的意义。

----------

## 11.2 Channel 在整个框架中的位置

整个调用流程可以整理为：

```
Caller    
│    
▼
Requestor    
│    
▼
Channel    
│
获取（或建立）连接    
│    
▼
Protocol    
│    
▼
Muduo    
│    
▼
Provider
```

其中：

**Requestor**

负责组织一次 RPC 请求。

**Protocol**

负责消息封包与解包。

而：

**Channel**

负责维护与远端 Provider 的通信连接。

因此，Channel 更偏向于整个框架中的**网络资源管理模块**。

----------

## 11.3 为什么不能每次 RPC 都重新连接

如果每次调用都重新建立连接，那么整个调用过程如下：

```
RPC1
connect
send
close
RPC2
connect
send
close
RPC3
connect
send
close
```

可以发现：

一次业务调用对应一次 TCP 建立与释放。

不仅增加了网络开销，还会降低整个系统的吞吐能力。

而采用连接复用以后：

```
connect
↓
RPC1
↓
RPC2
↓
RPC3
↓
...
↓
close
```

整个连接生命周期远大于一次 RPC 调用。

多个请求共享同一个连接。

这样既减少了频繁建立连接的开销，也提高了连接资源的利用率。

----------

## 11.4 Channel 的职责

结合整个项目来看，Channel 并不负责：

-   请求生命周期管理；
-   消息序列化；
-   服务发现；
-   消息分发。

它真正负责的是：

-   建立连接；
-   保存连接；
-   获取连接；
-   管理连接生命周期。

因此，Channel 关注的是：

> **连接是否可用。**

而不是：

> **请求是否完成。**

这也是它与 Requestor 最大的区别。

----------

## 11.5 Requestor 与 Channel 的关系

虽然 Requestor 和 Channel 都会参与一次 RPC 调用，但两者承担的职责完全不同。

Requestor 管理的是：

```
Request
↓
Response
```

也就是一次请求从发送到结束的完整生命周期。

而 Channel 管理的是：

```
Connection
↓
Connection
```

即网络连接本身。

一次连接可以承载很多个 Request。

而一个 Request 在整个生命周期中，只会使用其中一个 Channel。

因此，两者之间形成了：

```
Requestor
↓
使用 Channel
↓
发送请求
```

的协作关系，而不是包含关系。

## 11.6 为什么 Requestor 不直接管理连接

刚开始整理整个客户端架构时，我有一个疑问。

既然 Requestor 本来就负责发送 RPC。

为什么还需要再抽出一个 Channel？

直接让 Requestor 保存连接不是更简单吗？

后来重新梳理整个调用流程以后发现。

如果 Requestor 同时负责：

```
发送请求+管理连接
```

那么它就需要处理很多额外的问题，例如：

-   当前连接是否已经建立；
-   连接是否已经断开；
-   是否需要重新连接；
-   多个请求是否能够共享连接。

这样一来。

Requestor 的职责就不再只是请求管理，而开始承担网络连接管理。

整个模块会越来越复杂。

因此项目将：

```
请求管理
```

与：

```
连接管理
```

拆分到了两个独立模块。

Requestor 只需要：

```
我要向 HostA 发送请求
```

至于：

```
HostA 有没有连接？没有怎么办？连接失效怎么办？
```

全部交给 Channel 负责。

整个职责划分更加清晰。

----------

## 11.7 一个 Channel 可以服务多个 Request

理解了 Channel 以后，我觉得最容易混淆的一点就是：

**Channel 和 Request 并不是一一对应关系。**

真正的关系应该是：

```
Channel    
│    
├──── Request1    
├──── Request2    
├──── Request3    
└──── Request4
```

也就是说。

一次 TCP 连接建立以后。

多个 RPC 请求都可以通过这条连接发送。

因此。

Request 的生命周期通常很短。

```
创建
↓
发送
↓
收到响应
↓
结束
```

而 Channel 的生命周期则要长得多。

```
建立连接
↓
发送很多 RPC
↓
发送很多 RPC
↓
发送很多 RPC
↓
关闭连接
```

正因为两者生命周期不同。

所以项目才将它们拆成两个模块分别管理。

----------

## 11.8 Channel 与 Registry 的关系

结合前面几章来看。

很多模块都会接触：

```
Host
```

但是它们关注点完全不同。

Registry：

回答的是：

> **有哪些 Host 可以提供服务？**

Channel：

回答的是：

> **这个 Host 有没有可用连接？**

例如：

Registry 返回：

```
Host1
Host2
Host3
```

随后。

客户端选择：

```
Host2
```

真正发送请求之前。

首先询问：

```
Channel
↓
Host2 的连接是否已经存在？
```

如果存在。

直接复用。

如果不存在。

建立新的连接。

因此。

Registry 与 Channel 虽然都围绕 Host 工作。

但属于两个完全不同的阶段。

----------

## 11.9 Channel 在整个客户端中的作用

结合整个客户端的数据流。

可以整理为：

```
Requestor      
│      
▼
获得目标 Host      
│      
▼
Channel      
│
获得 Connection      
│      
▼
Protocol      
│      
▼
发送数据
```

可以发现。

Channel 实际上就是：

**连接资源管理中心。**

它向上提供：

```
Connection
```

向下屏蔽：

```
SocketTCPMuduo 
Connection
```

这些底层细节。

因此。

Requestor 根本不需要关心：

连接是如何建立的。

只需要得到一个：

```
Connection
```

即可发送 RPC。

----------

# 12. 发布订阅（Topic）

## 12.1 为什么需要 Topic

前面整个 RPC 框架实现的都是：

> **请求——响应（Request / Response）**

整个调用过程如下：

```
Caller    
│    
▼
Provider    
│    
▼
Response
```

也就是说。

一次请求。

对应一次响应。

通信双方是一对一关系。

但是，并不是所有业务都适合这种模式。

例如：

```
聊天室
消息通知
系统广播
日志推送
```

这些场景都有一个共同特点。

**发送方并不知道最终有多少接收方。**

例如：

聊天室发送一句：

```
Hello
```

真正收到消息的可能有：

```
UserA
UserB
UserC
UserD
```

因此。

RPC 的 Request / Response 模型已经不能满足这种需求。

项目于是增加了：

> **Topic（发布订阅）**

通信模型。

----------

## 12.2 Topic 与 RPC 的区别

两者最大的区别就在于：

RPC：

```
一个请求
↓
一个响应
```

而 Topic：

```
发布一条消息
↓
多个订阅者收到
```

因此。

RPC 更关注：

> **调用。**

Topic 更关注：

> **通知。**

整个通信方式完全不同。

----------

## 12.3 Topic 在整个框架中的位置

整个发布订阅流程如下：

```
Publisher      
│      
▼
Topic      
│      
▼
Subscriber A
Subscriber B
Subscriber C
```

Publisher 不需要知道：

到底有哪些 Subscriber。

它只负责：

```
向 Topic 发布消息
```

至于：

最终通知哪些客户端。

全部由 Topic 模块负责。

因此。

Publisher 与 Subscriber 完全解耦。

----------

## 12.4 为什么采用 Topic

如果没有 Topic。

发送方只能这样：

```
发送 UserA
↓
发送 UserB
↓
发送 UserC
```

如果：

新增：

```
UserD
```

发送方又必须修改代码。

整个系统耦合越来越高。

而采用 Topic 后。

发送方始终面对：

```
Topic
```

发送：

```
Publish()
```

即可。

至于：

有多少 Subscriber。

谁在线。

谁离线。

发送方完全不用关心。

因此。

Topic 真正完成的是：

> **发布者与订阅者之间的解耦。**

----------

## 12.5 Topic 的职责

阅读整个模块以后。

Topic 本身并不是：

消息内容。

而是：

一种消息组织方式。

它负责：

-   管理 Topic；
-   管理订阅关系；
-   接收发布消息；
-   将消息广播给对应订阅者。

因此。

整个模块围绕的是：

```
Topic
↓
Subscriber
```

之间的关系。

而不是：

具体消息内容。

----------

## 12.6 Publisher 与 Subscriber 的关系

整个通信关系可以整理为：

```
Publisher
↓
Topic
↓
Subscriber
```

其中：

Publisher 不知道：

有哪些 Subscriber。

Subscriber 也不知道：

Publisher 是谁。

双方唯一共同认识的是：

```
Topic
```

因此。

Topic 成为了整个消息广播过程中的：

**通信媒介。**

----------

## 12.7 Topic 与 RPC 为什么可以共存

刚开始我觉得。

既然已经有 RPC。

为什么还需要 Topic？

后来发现。

两者解决的是两个完全不同的问题。

RPC：

回答的是：

> **谁帮我完成这个任务？**

例如：

```
登录
支付
查询
```

必须得到返回结果。

----------

Topic：

回答的是：

> **我有一条消息需要通知所有关心它的人。**

例如：

```
聊天室
广播
状态更新
```

发送以后。

并不要求：

某一个固定对象返回结果。

因此。

整个项目同时支持：

```
RPC+Topic
```

实际上丰富了整个通信模型。

----------

# 13. 一次 RPC 调用的完整流程

## 13.1 从业务代码开始

整个 RPC 的入口实际上非常简单。

业务层并不会直接操作：

-   Socket；
-   TCP；
-   Protocol；
-   Dispatcher。

对于业务开发者来说。

整个过程仍然像调用普通函数一样。

例如：

```
UserService.Login()
```

业务层真正关心的是：

> **我要调用哪个远程接口。**

至于：

-   服务部署在哪里；
-   如何建立连接；
-   如何发送数据；
-   如何等待响应；

全部由 RPC 框架完成。

因此。

RPC 最大的目标就是：

> **让远程调用尽可能接近本地调用。**

----------

## 13.2 Requestor 接管整个请求

业务层发起调用以后。

真正进入 RPC 框架。

首先来到：

```
Requestor
```

Requestor 并不会立即发送数据。

而是首先完成一次请求的组织。

整个过程可以整理为：

```
创建 Request
↓
生成 rid
↓
保存请求描述
↓
准备发送
```

其中。

rid 是整个请求生命周期中最重要的信息。

以后：

收到 Response。

也是依靠：

```
rid
```

找到对应请求。

因此。

Requestor 真正管理的是：

**一次 RPC 请求的生命周期。**

----------

## 13.3 获取目标 Provider

Requestor 知道：

要调用哪个 Method。

但是：

它并不知道：

Provider 在哪里。

因此。

下一步需要进行：

服务发现。

整个流程如下：

```
Method
↓
Registry
↓
Host
```

Registry 返回：

能够处理当前 Method 的 Provider。

此时。

客户端终于知道：

下一步应该连接哪台机器。

Registry 到这里就结束了自己的工作。

----------

## 13.4 获取网络连接

得到目标 Host 以后。

并不会立即创建 Socket。

而是交给：

```
Channel
```

负责处理。

Channel 首先检查：

```
是否已经存在可用连接？
```

如果存在。

直接复用。

如果不存在。

建立新的 TCP 连接。

最终。

Requestor 得到：

一个可以发送数据的 Connection。

因此。

整个连接管理过程。

对于 Requestor 来说完全透明。

----------

## 13.5 Protocol 完成消息封装

得到连接以后。

请求对象并不能直接发送。

因为：

Socket 只能发送：

```
Byte Stream
```

而不能发送：

```
RpcRequest
```

因此。

Protocol 首先完成：

```
Message
↓
ByteStream
```

整个过程包括：

-   消息序列化；
-   协议头封装；
-   长度编码。

最终形成：

能够通过 TCP 发送的数据。

----------

## 13.6 网络发送

完成封包以后。

数据正式进入网络层。

整个流程如下：

```
Protocol
↓
Muduo
↓
TCP
↓
Provider
```

这一阶段。

RPC 框架已经完成了自己的客户端工作。

真正的数据开始通过网络发送到远端。

----------

## 13.7 服务端处理请求

服务端收到数据以后。

整个流程实际上与客户端相反。

首先：

Protocol 完成：

```
ByteStream
↓
Message
```

恢复出真正的：

```
RpcRequest
```

随后。

消息进入：

```
Dispatcher
```

Dispatcher 根据：

MessageType。

找到对应 Handler。

真正执行业务逻辑。

整个服务端流程如下：

```
Protocol
↓
Dispatcher
↓
Handler
↓
Service
```

----------

# 13.8 Response 返回客户端

业务处理完成以后。

服务端构造：

```
RpcResponse
```

随后再次经过：

```
Protocol
↓
TCP
↓
客户端
```

客户端收到数据以后。

再次完成：

```
ByteStream
↓
Message
```

恢复 Response。

----------

# 13.9 Requestor 完成请求生命周期

客户端恢复 Response 以后。

再次进入：

```
Requestor
```

首先读取：

```
rid
```

随后：

找到之前保存的请求描述。

根据调用方式。

分别执行：

```
同步FutureCallback
```

通知业务层。

随后：

删除请求描述。

整个 Request 生命周期正式结束。

因此。

一次 Request 的完整生命周期就是：

```
创建
↓
发送
↓
等待
↓
收到 Response
↓
通知业务层
↓
释放资源
```

----------

## 13.10 一张图看完整个调用过程

整理整个框架以后。

整个 RPC 调用过程可以总结为：

```
Business    
│    
▼
Requestor    
│    
▼
Registry    
│    
▼
Channel    
│    
▼
Protocol    
│    
▼
TCP    
│    
▼
Protocol    
│
▼
Dispatcher    
│    
▼
Handler    
│    
▼
Service    
│    
▼
Response    
│    
▼
Protocol    
│    
▼
TCP    
│    
▼
Protocol    
│    
▼
Requestor    
│    
▼
Business
```

# 14. 项目整体设计复盘

## 14.1 为什么整个项目要分层

刚开始阅读整个项目时，我觉得模块很多：

-   Message
-   Protocol
-   Dispatcher
-   Requestor
-   Registry
-   Channel
-   Topic

看起来有些复杂。

后来把整个调用流程完整串起来以后，我发现：

整个项目其实一直在回答几个不同的问题。

```
我要调用什么？
↓
去哪里调用？
↓
怎么建立连接？
↓
怎么发送数据？
↓
收到消息交给谁？
↓
如何通知调用者？
```

每一个问题，都对应一个独立模块。

因此整个框架不是为了"模块多"，而是为了：

> **一个模块只解决一个问题。**

这也是整个项目最重要的设计思想。

----------

## 14.2 每个模块负责什么

整个项目可以重新整理为：

```
Message
│
负责描述消息

Protocol
│
负责消息编解码

Registry
│
负责服务发现

Channel
│
负责连接管理

Requestor
│
负责请求生命周期

Dispatcher
│
负责消息分发

Topic
│
负责发布订阅
```

可以发现。

几乎没有两个模块解决同一个问题。

每个模块都有自己明确的职责。

因此整个框架阅读起来非常自然。

----------

# 14.3 为什么整个框架耦合度比较低

阅读完整个项目以后，我最大的感受就是：

**模块之间几乎都是"使用关系"，而不是"依赖实现关系"。**

例如：

Requestor 不关心：

```
Connection 怎么建立。
```

它只需要：

```
得到 Connection。
```

----------

Protocol 不关心：

```
消息如何处理。
```

它只需要：

```
恢复 Message。
```

----------

Dispatcher 不关心：

```
业务怎么执行。
```

它只需要：

```
找到 Handler。
```

因此。

每个模块都只知道：

> **我需要什么。**

而不知道：

> **别人内部怎么实现。**

整个项目很好地控制了模块之间的耦合。

----------

# 14.4 为什么扩展成本比较低

整个项目还有一个比较明显的特点。

新增功能时。

通常不用修改已有模块。

例如：

新增一种 Message。

只需要：

```
新增 Message
↓
新增 Handler
↓
完成注册
```

Dispatcher 不需要修改。

----------

新增一种调用方式。

只需要扩展：

Requestor。

Protocol 不需要修改。

----------

新增一种协议。

只需要修改：

Protocol。

业务层完全不用修改。

因此。

整个项目大多数模块都符合：

> **对扩展开放，对修改关闭。**

这也是整个框架比较容易继续演进的重要原因。

----------

## 14.5 整个项目涉及的设计思想

整理整个项目以后。

我认为主要体现了以下几个设计思想。

**单一职责原则**

每个模块只负责一个功能。

例如：

Protocol 负责协议。

Dispatcher 负责分发。

Requestor 负责请求管理。

----------

**分层设计**

整个调用流程天然分层：

```
业务层
↓
RPC 调用层
↓
协议层
↓
网络层
```

每层都有清晰边界。

----------

**抽象与封装**

整个项目大量通过抽象接口屏蔽底层实现。

例如：

业务层无需感知：

-   TCP；
-   Muduo；
-   Json。

而是直接操作：

```
RpcRequest
RpcResponse
```

这些业务对象。

----------

**模块解耦**

整个框架通过 Message、Dispatcher、Registry 等模块，把不同职责拆分开来，使模块之间保持较低耦合。

----------

# 14.6 如果继续完善这个项目

如果继续完善整个 RPC 框架。

我认为还有一些可以继续扩展的方向。

例如：

**序列化协议**

目前采用 Json。

后续可以增加：

-   Protobuf；
-   MessagePack；
-   FlatBuffers。

提高传输效率。

----------

**负载均衡策略**

增加：

-   随机；
-   轮询；
-   一致性 Hash；
-   最少连接。

根据业务场景选择不同策略。

----------

**服务治理**

例如：

-   心跳检测；
-   节点健康检查；
-   自动摘除故障节点；
-   自动恢复节点。

进一步提高整个系统稳定性。

----------

**异步能力**

进一步结合协程。

让整个异步调用更加自然。

例如：

```
co_await rpc.call(...)
```

避免 Future 与 Callback 带来的额外复杂度。

----------

# 14.7 整个项目最大的收获

完成整个项目以后，我认为最大的收获并不是实现了一个能够完成 RPC 调用的程序。

而是第一次真正理解了：

**一个中型 C++ 项目应该如何组织模块。**

以前更多关注：

```
一个类怎么写。一个函数怎么写。
```

而这个项目让我开始关注：

```
一个模块负责什么？为什么这样划分？模块之间如何协作？如何降低耦合？
```

这些问题比具体代码实现更加重要。

----------

<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk1NzA4Njg0M119
-->