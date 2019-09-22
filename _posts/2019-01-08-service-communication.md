---
layout: post
cover: false
navigation: true
title: 分布式系统基础学习(02)--应用间通信(gRPC)
date: 2019-01-08 09:49:09
tags: ['分布式系统']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

我始终认为， 随着语言和计算机技术的发展， 分布式系统才是互联网技术的未来。 而现在众多的分布式开源技术， 如`Zookeper`, `SpringCloud`等的蓬勃发展也证明这一点。 在分布式的系统中， A服务可能由Java编写， B服务可能由Python编写， C服务可能由PHP编写， 服务于服务之间的通信固然可以使用`HTTP`协议以`RESTful-API`的方式进行通信， 但是由于许多不必要的数据传输， 例如HTTP请求头， 会降低服务之间的通信速度， 在高并发的场景下极有可能成为系统瓶颈。 所以， 就有了RPC通信机制。

<!---more--->

#### 1. 什么是RPC
RPC， 全称为 Romete Procedure Call， 远程过程调用， 实际上就是A机器调用B机器上的方法， B机器进行一系列的运算将结果返回给A机器。

那么在这个过程中就涉及到Server服务器寻址， 服务器之间TCP连接的建立， 对象序列化与反序列化的过程， 以及方法寻址的问题。

![ | center ](https://www.cs.rutgers.edu/~pxk/417/notes/images/rpc-flow.png)

上图来源于https://www.cs.rutgers.edu/~pxk/417/notes/03-rpc.html， 对RPC的内部过程展示的非常详细。 `stub`以及`skeleton`是对旧有的client和server的一种描述， 现在少有这种称呼， 对于提供方法的服务器， 称为server， 调用方统称为stub。

假设A, B服务均由`Python`编写， 那么client传递参数给server， server经过计算之后返回结果即可， 在同平台下RPC的调用并没有很复杂。 但是如果跨平台呢？ 此时就需要有一个统一的描述语言来定义相关的数据类型， 达到多平台统一， 我们将其称为接口描述语言(Interface Description Language，IDL)， 其中， Google Protocol Buffers就是一种IDL。


#### 2. Protocol Buffers
什么是Protocol Buffers？ 简单来讲， Protocol Buffers就是一种使具有一定组织结构的数据自动序列化的机制， 使用特殊的编译器对其进行编译之后， 可以跨平台使用。

首先来看官网给出的demo， 基于proto3：

```protobuf
syntax = "proto3";

// message为关键字， 用于定义结构体， 相当于C中的struct； Python, Java中的class
message SearchRequest {
  // field后面的数值， = 1并不表示值为1， 而是表示该字段的序号， 该序号在一个message中必须唯一
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message Result {
  string url = 1;
  string title = 2;
  // repeated， 允许字段重复0次或多次， 相当于一个列表
  repeated string snippets = 3;
}
```

`message`是`.proto`文件的关键字， 类似于其它语言中的`class`， 用于定义一个对象(结构体)。

在proto3中移除了`required`和`optional`两个关键字， 仅保留了`repeated`， 用于表示列表。

基本数据类型在Protocol Buffers中也有提供， 包括`double`, `float`, `int32`, `int64`, `bool`, `bytes`以及`string`， 同时`repeated`提供了列表结构， 以及`map<K, V>`所提供的字典结构。 在有了这些数据类型以及数据结构的支持， 我们就可以在多平台上进行通信了。

更多的细节请参考官方文档： https://developers.google.com/protocol-buffers/docs/proto3

#### 3. 使用Protocol Buffers编写一个demo
学习一项新技术最好的方法就是跟着官方文档跑一遍demo， 只不过这里官方的demo有些啰嗦， 所以就自己稍加改造了一下， 本质上的原理都是一样的。

首先来看proto文件：
```protobuf
syntax = "proto3";

package ProtoBuf;

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

如果把这个Person转换成Json的话， 会更加的清晰， 也有助于我们查看：
```json
{
    "name": "xx",
    "id": xx,
    "email": "xx",
    "phones": [
        {"number": "xx", "type": 0},
    ]
}
```

在定义完.proto文件之后， 我们需要使用`protoc`这个编译器对`addressbook.proto`文件进行编译， 让其自动的生成`Python`, `C++`, `Java`代码， 可以认为这份自动生成的代码就是`skeleton`。 由于我们要生成的是`Python`代码， 所以运行：
```bash
protoc -I="." --python_out="." ./addressbook.proto
```
该命令将会在当前目录下生成`addressbook_pb2.py`， 里面写的是什么可以不用关心， 下面开始编写测试文件。 我们的测试是首先创建出一个`AddressBook`对象， 填充一个`Person`对象， 将其序列化后写入到一个文本文件中， 然后再读取该文本文件， 将其反序列化称为`Python`对象。

```python
from ProtoBuf import addressbook_pb2

address_book = addressbook_pb2.AddressBook()
# 添加一个person
person = address_book.people.add()
# 添加person属性
person.name = "smart"
person.id = 123
person.email = "smartkeyerror@gmail.com"
# person中添加一条phones, 并赋予属性
phone_number = person.phones.add()
phone_number.number = "1365656"
phone_number.type = addressbook_pb2.Person.HOME

# 将序列化的二进制数据写入文件
with open("temp.txt", "wb") as f:
    f.write(address_book.SerializeToString())

# 读取二进制文件
with open("temp.txt", "rb") as f:
    data = f.read()

address_book = addressbook_pb2.AddressBook()
# 反序列化
address_book.ParseFromString(data)

first_people = address_book.people[0]

print("name: ", first_people.name)
print("id: ", first_people.id)
print("email: ", first_people.email)
print("phone_number: ", first_people.phones[0].number)
print("phone_number_type", first_people.phones[0].type)
```

整个代码很简单， 没有什么很绕的地方， 如果`Java`程序想要读取该二进制文件并且反序列化称为一个`Java`对象该如何去做？ 首先使用`protoc`来生成`Java`相关的代码， 然后利用生成的代码按照官网指示去编写代码即可。 这样就完成了在多平台共享同一对象的作用。

通过上面的例子我们对Protocol Buffers已经有了一个基本的认识， `.proto`文件就是一个结构定义文件， 利用`protoc`编译器可以生成`Python`, `Java`等语言代码， 依托于自动生成的代码， 我们可以将一个保存在文件中的二进制数据转换成`Python`对象， `Java`对象。 如果将该数据放到网络中， 就可以以对象的方式来进行不同平台间的通信了。 `gRPC`就是这样的一种工具。


#### 4. gRPC
Protocol Buffers完成了不同语言之间的数据转换问题， 而`gRPC`则是使用Protocol Buffers进行不同服务间通信的RPC框架。 它帮我们建立了一个轻量且高效的`server`和`client`， 也就是说我们不必关心服务间怎么建立TCP连接， 怎么寻找方法调用， `gRPC`已经帮我们完成了这些功能。

首先， 还是跑一遍官网的demo， 过程就不贴了， 直接贴官方所给出的实例代码， 首先当然是`.proto`文件：

```protobuf
syntax = "proto3";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

删除了一些与`Python`语言无关的代码， `service`指的就是远程调用的方法， 就像定义一个接口一样， 定义方法名称， 入参， 返回值类型以及一个空的方法体即可。 `message`的定义也比较简单， 可以看到方法的调用传入一个`HelloRequest`对象， 其中包含一个`name`字段； 返回一个`HelloReply`对象， 包含一个`message`字段。

然后来分析`greeter_server.py`中的内容：
```python
from concurrent import futures
import time

import grpc

import helloworld_pb2
import helloworld_pb2_grpc

_ONE_DAY_IN_SECONDS = 60 * 60 * 24


class Greeter(helloworld_pb2_grpc.GreeterServicer):

    def SayHello(self, request, context):
        return helloworld_pb2.HelloReply(message='Hello, %s!' % request.name)


def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    helloworld_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    try:
        while True:
            time.sleep(_ONE_DAY_IN_SECONDS)
    except KeyboardInterrupt:
        server.stop(0)


if __name__ == '__main__':
    serve()
```

源代码还是值得看一看的， 对于`serve`方法， 首先是生成了包含10个线程的线程池， 并传入给了`grpc.server`方法。 非常缺略的过了一遍`grpc.server`方法， 应该是使用`Reactor`模型所实现的I/O多路复用模型， 至于底层是使用的`poll`还是`epoll`， 需要再详细的阅读， 这里的话知道这么一回事即可， 具体的源码分析先放一放。

然后将`Greeter`对象和`server`对象传入给了`add_GreeterServicer_to_server`这个方法， 很明显是由`gRPC`所提供的， 这应该是一个通用方法， 然后绑定50051端口， 开启事件循环， 准备接受客户端的连接。

`Greeter`类继承了一个不知道是啥的类， 然后添加了一个在`.proto`中所定义的方法， 只不过这里比`.proto`文件多了一个参数， `context`， 这个参数有什么意义后面再讲。

然后再来看客户端的代码：

```python
def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = helloworld_pb2_grpc.GreeterStub(channel)
        response = stub.SayHello(helloworld_pb2.HelloRequest(name='you'))
    print("Greeter client received: " + response.message)
```

比较简单， 这里使用`with`来管理`channel`资源， 也就意味着在示例中是每调用一次方法都建立一个新的连接， 在实际使用中可以保持这个长连接， 有没有心跳机制还是需要看源码。 然后获取了一个`stub`对象， 直接调用了`SayHello`方法， 并实例化了一个`HelloRequest`对象作为参数传入， 打印调用结果。 远程过程调用就此结束。

虽然官网给出的demo比较简单， 但是这的确是我们编写复杂逻辑的基础：

1）定义`.proto`文件: 该文件包括client发送的对象以及server返回的对象， 以及服务名称和该服务下所对应的方法

2）利用Google所提供`Python`类库生成`_pb2.py`以及`_pb2_grpc.py`文件， 前者为`.proto`文件所生成的数据结构文件， 后者服务于`gRPC`调用

3）定义服务端代码：首先编写与`.proto`文件中`service`相同名称的类， 并继承`_pb2_grpc.py`中所对应的`servic`类， 然后实现定义好的函数

4）启动服务端， 服务端为一个轻量级的`Reactor`模型， 根据服务器硬件条件定义线程池大小

5）编写客户端调用代码

#### 5. 在生产环境中使用gRPC
回顾一下我们使用`gRPC`框架的过程， 首先我们先得定义`.proto`文件， 编写函数调用的入参对象以及返回对象， 然后利用相关的编译工具生成`_pb2.py`以及`_pb2_grpc.py`文件， 前者为Protoco Buffers所需代码， 后者则为远程方法调用所用文件， 也就是说在client以及server中， 都需要这两个文件。

那么问题就来了， 通常我们的client和server是两个不同的项目， 不会共用同一个代码仓库， 如果我的`.proto`文件位于`server`中， 此时修改了`.proto`文件， 重新编译并生成了代码， 这个时候client端也需要进行更新， 虽然直接把文件复制粘贴过去是可以解决的， 但是并不是很优雅。

那么此时我们就可以使用`Git submodule`来管理公共子模块。 将`.proto`文件单独抽离成一个git项目， 然后在client和server端的仓库中使用该公共项目。

关于`Git submodule`的更多内容请见文章`Git最佳实践`。

#### 6. 小结
在本篇文章中我们通过官方文档了解了`Protocol Buffers`以及`gRPC`的简单使用， 基本上这些内容也是我们在生产中使用最为频繁， 且核心的内容， 剩下的就是使用时的一些细节以及数据结构的设计。

就我个人而言， 只有在需要高并发的场景下才会使用`gRPC`来进行远程调用， 普通场景下使用`RESTFul-API`来进行调用即可， 以获得最大的开发速度。

然而`RPC`框架远不止`gRPC`， 市场上有相当多的选择， 具体的选择， 还是需要看团队对框架的熟悉程度以及具体的业务场景。


