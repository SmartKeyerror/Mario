---
layout: post
cover: false
navigation: true
title: Golang中的interface
date: 2019-12-25 08:25:25
tags: ['Golang']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


Golang除了方便使用的协程以外，最令我感到惊讶的就是`interface`，接口。在其它语言中，接口承担的主要作用为解耦和协议，但是在Golang中，`interface`还作为一种"通用"类型广泛使用于标准库和第三方库中。

<!---more--->

### 1. 面向接口编程

面向接口编程的核心就在于将接口和实现分离，封装不稳定的实现，暴露稳定的接口。上游系统面向接口编程而非面向实现编程，不依赖不稳定的实现细节，当实现发生变化时，上游系统可不做或者只需进行少量的修改，从而降低耦合性，提高拓展性。 换句话说，面向接口编程是一种可随时拔插替换的编程方法。


#### 1.1 接口的含义

不管是在Java语言还是在Go语言中，接口本身的定义均只包含方法，并不包含具体的实现。换句话说，接口实际上是定义了一组行为，但是没有定义这些行为到底该怎么进行。

接口描述了"如果你是...则必须能..."的分类思想，如果你是动物，那么必须能呼吸，移动和进食。如果你是植物，那么你必须能进行光合作用。但是，具体的动物如何呼吸(鱼用腮呼吸，狮子用肺呼吸)，如何移动(鸟既能飞又能跑，狮子不能飞)，是由实现了动物这个接口的具体动物所决定的。

接口将一类事物的行为提炼并抽象出来，从而达到简化事物复杂度的目的，以便相关的研究人员更关注于他们想要关注的，而忽略其它的细节。


#### 1.2 依赖反转原则

在SOLID原则中，依赖反转原则对于增强系统的可拓展性、降低代码的耦合性至关重要。依赖反转原则描述了这样一个概念: 

>高层次模块不应依赖于低层次模块的具体实现细节，两者都应该依赖于抽象

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Go/interface/DIP.png)

依赖反转原则简单来说就是额外地增加了一层抽象(接口)，该模块抽象了模块A所依赖的所有行为。而模块B则实现该抽象(接口)，并在运行时通过依赖注入的方式注入进模块A。如此一来，将来若想要替换掉模块B，只需要重新实现该接口，并在少量的代码中进行改动即可。


#### 1.3 接口的实际意义

接口的实际意义其实就是一个标准，或者说一种协议。例如SSD的M.2接口，不管是三星的970 evo plus，还是海盗船的MP 510，在内部硬件的实现细节上虽然存在差异，但是它们都能够在支持M.2的主板上正常运行。这其实就是标准化的意义: 兼容性和可交换性。

从抽象代码上来看，接口就是一种约束，用于约束对象的行为，使得对象标准化。


### 2. 实现接口

不同于Java语言中使用`implements`关键字实现接口，Golang中的某一个类型实现接口是隐式的: 只要类型实现了接口中的全部方法，就认为该类型实现了该接口。

```go
type LogData struct {}

type LogStorage interface {
    Insert(data LogData)
}

type MongoStorage struct {}

func (m *MongoStorage)Insert(data LogData) {/*...*/}
```

#### 2.1 接口和指针

当接口和指针在一起使用时，往往会产生一些令人迷惑的问题。方式的接收者有值接收者和指针接收者，那么也就会有两种实现接口的方式，而这两种实现方法在使用过程中需要特别小心。

```go
type LogData struct {}

type LogStorage interface {
    Insert(data LogData)
    InsertMany(dataSlice []LogData)
}

type MongoStorage struct {}

/* 指针接收者 */
func (m *MongoStorage)Insert(data LogData) {/*...*/}

/* 值接收者 */
func (m MongoStorage)InsertMany(dataSlice []LogData) {/*...*/}
```

在示例中，`MongoStorage`这一具体的实现存在一个指针接收者方法，一个值接收者方法。当尝试使用结构体初始化变量时，将无法通过编译:

```go
func main() {
    var m LogStorage = MongoStorage{}
}

./mian.go:31:6: cannot use MongoStorage literal (type MongoStorage) as type LogStorage in assignment:
	MongoStorage does not implement LogStorage (Insert method has pointer receiver)
```

原因在于尽管`InsertMany`方法使用了值接收者实现，但是`Insert`方法却使用了指针接收者实现。由于Go方法调用按值传递，通过对指针的解引用可以获取到该指针指向的值，但是却无法获取到某一个变量的指针，因为在内存中可能存在多个指向该变量的指针。

```go
func main() {
    var m LogStorage = &MongoStorage{}
    m.InsertMany([]LogData{})
}
```

当执行`m.InsertMany()`语句时，Go会将指向`MongoStorage{}`结构体的指针进行解引用，取出结构体`MongoStorage{}`并进行方法调用。

在实际应用中，为了节省实参的拷贝开销，通常都会使用指针接收者来实现接口中的方法。那么在定义变量时，需要指针变量。


### 3. 接口的值

Go语言中接口的定义形式为:

```go
type interfaceName interface {
    functionName(p Type) returnType
}
```

从接口定义中可以看到，`interface`是一个类型，那么既然是一个类型，就应该有值。接口的值由两部分组成: 接口的动态类型和该类型的值，前者称为动态类型，后者称为动态值。Go接口的动态类型和Java的RTTI一样，在运行时确定某个接口的具体类型。

```go
var w io.Writer                  // ①
w = os.Stdout                    // ②
w.Write([]byte("Hello World~"))  // ③
```

①: 声明了变量`w`，且其类型为`io.Writer`，由于`io.Writer`是一个接口定义，并且Golang会在变量被定义时即对变量进行初始化，那么变量`w`也会被初始化。**接口的零值就是将其动态类型和动态值均设置为nil**。

②: 将`os.Stdout`这一具体类型赋值给了`w`，相当于将一个具体类型隐式转换成了接口类型。那么此时，`w`就有了动态类型和动态值。其动态类型为`*os.File`，其动态值为`*os.file`。

③: 调用该接口值的`Write`方法，实际上调用的是`(*os.File).Write`方法。在调用方法时，仍然需要使用动态分发的手段来获取到方法地址。

#### 3.1 Go接口的实现

在Golang中，接口的实现其实有两种，一种是拥有方法的接口，另一种则是不拥有方法的接口，后者通常表示为`interface{}`，将在文章的后续进行描述。

由用户自定义的、带有方法的接口通过`iface`结构体实现，位于源码`/src/runtime/runtime2.go`中:

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}
```

其中`*itab`表示接口的动态类型，`unsafe.Pointer`则指向接口的动态值。**对于一个接口变量而言，只要其动态类型的值不为nil，接口值就不为nil**

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Go/interface/interface-value.png)


### 4. interface{}

`interface{}`表示不包含任何方法的接口，而Golang中不管是基本数据类型，还是复合数据类型，还是用户自定的类型，都至少包含零个方法。换句话说，所有的类型都实现了`interface{}`。

首先查看下`fmt.Println`函数的定义:

```go
func Println(a ...interface{}) (n int, err error) {
    return Fprintln(os.Stdout, a...)
}
```

`Println`方法接收任意多个`interface{}`类型的参数，这也是为什么`Println`方法能够接收任意类型的原因: **所有传入的参数均进行了隐式转换，转换成了`interface{}`类型**。

```go
func main() {
    fmt.Println(10)
    fmt.Println("123")
    fmt.Println(map[string]string{"name": "SmartKeyerror"})
    fmt.Println([]string{"foo", "bar"})
}
```

但是，需要特别注意的是: `interface{}`并不代表任意类型，它只是一种特殊的类型。

```go
func Bar(v []interface{}) {
    /*...*/
}

func main() {
    Bar([]string{"foo", "bar"})
}
```

上述代码将在编译期抛出异常:

```bash
./mian.go:35:14: cannot use []string literal (type []string) as type []interface {} in argument to Bar
```

`[]interface{}`和`[]string`是完全不同的类型，`interface{}`占用固定的内存空间，而`[]Type`则不能确定占用内存空间大小，它们自然不是同一种类型。

#### 4.1 interface{}的实现

和带有方法的接口一样，`interface{}`的定义也在`/src/runtime/runtime2.go`:

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

其中`_type`为Go语言类型的运行时表示，包括了一些元信息，包括大小、哈希值等等，而`data`用于保存实际运行时的数据，是一个指向原始数据的指针。`eface`和`iface`差别并不大，均包括运行时的动态值和动态类型。


### 5. 类型断言与类型分支

#### 5.1 类型断言

和Python中的`isinstance`、`issubclass`类似，Golang提供了类型断言来判断某个变量是否为某一种类型，其格式为`x.(T)`。

```go
func main() {
    var w io.Writer
    w = os.Stdout
    
    if _, ok := w.(*os.File); ok {
	    fmt.Println("Assert Right")
    }else {
	    fmt.Println("Assert Wrong")
    }
}
```

若`T`为具体类型，那么类型断言会检查x的动态类型是否为`T`。若断言成功，结果即为`x`的动态值，类型当然就是`T`。

若`T`为接口类型，那么类型断言会检查`x`的动态类型是否满足`T`。若断言成功，结果仍为接口值，不过此时的类型为接口类型`T`。

#### 5.2 类型分支

当某个函数接收`interface{}`类型参数时，需要在函数内部来确定其动态类型，此时可使用`x.(type)`类型分支。

```go
func foo(v interface{}) {
    switch v.(type) {
    case int:
	    fmt.Println("v is int")
    case string:
	    fmt.Println("v is string")
    default:
	    fmt.Println("unknown type")
    }
}

func main() {
    foo(10)
    foo("10")
    foo([]string{"smart"})
}
```
