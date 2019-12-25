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

#### 1. 面向接口编程

面向接口编程的核心就在于将接口和实现分离，封装不稳定的实现，暴露稳定的接口。上游系统面向接口编程而非面向实现编程，不依赖不稳定的实现细节，当实现发生变化时，上游系统可不做或者只需进行少量的修改，从而降低耦合性，提高拓展性。 换句话说，面向接口编程是一种可随时拔插替换的编程方法。

以图片存储服务为例，图片经过一系列的处理之后上传至阿里云OSS中保存。 以Java代码为例，为了代码复用，将其封装成一个类对外使用:

```java
public class AliyunOssImageStore {
    public void createBucketIfNotExist(string bucketName) throws Exception {
        // OSS通过bucket来划分整块OSS存储空间
        // 当所需要的bucket不存在时，进行创建，创建失败则抛出异常
    }
    
    private String generateAccessToken() {
        // 通过appKey以及appSecret生成access_token, 内部方法
    }
    
    public String uploadToOss(Image image, string bucket) {
        // 将图片上传至oss，返回图片URL地址
    }
}

// 使用
class OssProcess {
    public static void main(String[] args) {
        String bucketName = "user_thumbnail_image";
        AliyunOssImageStore store = new AliyunOssImageStore(/*省略参数*/);
        store.createBucketIfNotExist(bucketName);
        Image image = ... // 生成图片
        store.uploadToOss(image, bucketName);
    }
}
```

`AliyunOssImageStore`就是一个具体的实现类，实现了将图片上传至阿里云OSS的功能，在绝大多数时候，代码都能完好的工作。现在由于采购部门不满阿里云的价格，想要更换图片存储供应商，比如七牛。现在开发人员需要添加`QiniuImageStore`，并替换掉原有的实现方式，如此一来，必将会涉及到大面积的代码改动，包括业务代码以及重新编写测试用例。

更换一个OSS服务就需要大面积的更改业务代码，违反了开放-封闭原则，从而引入了额外的工作量以及风险，上游系统直接依赖具体的实现是不妥当的，因为具体的实现很有可能发生变化，一旦发生变化，波及的范围可能是整个系统。

解决该问题的一种方式是添加代理类，即在业务代码和具体实现之间额外添加一层抽象，或者说函数，假设叫`imageStoreProxy`。业务代码调用`imageStoreProxy`方法，而该方法调用具体的OSS实现，当OSS实现发生变动时，只需修改`imageStoreProxy`一处即可。但是这么做的后果就是降低了系统的灵活性并增加了系统的复杂度，`imageStoreProxy`需要满足上游系统所有的需求，该方法到后期将会变得非常臃肿而难以维护，最可能出现的情况就是该方法存在乱七八糟的`if-else`。

一个更好的方式就是使用接口，上游系统依赖接口，具体实现根据接口定义的方法进行实现，并在运行时选择具体的实现。

```java
public interface ImageStore {
    String upload(Image image, String bucket);
}

public class AliyunOssImageStore implements ImageStore {
    public String upload(Image image, String bucket) {
        // 阿里云OSS的具体实现
    }
}

public class QiniuImageStore implements ImageStore {
    public String upload(Image image, String bucket) {
        // 七牛云的具体实现
    }
}

// 使用
class OssProcess {
    public static void main(String[] args) {
        ImageStore store = new AliyunOssImageStore(/*省略参数*/);
        store.upload(/*省略参数*/); // 使用阿里云OSS
        
        ImageStore store = new QiniuImageStore(/*省略参数*/);
        store.upload(/*省略参数*/); // 使用七牛云
    }
}
```

合理的使用接口可以完全屏蔽具体的实现细节，当具体实现发生变化时，上游系统只需改动少量的代码即可适应该变化。 例如底层数据存储，MySQL与MongoDB在实现上完全不同，数据的CRUD有着非常大的区别，但是通过定义合理的接口，再根据接口封装MySQL与MongoDB的具体实现，上游系统则可以完全忽略其细节的区别，只关注自身的业务逻辑，从而实现耦合的解除。

常见ORM就是做的，例如Python中的`SQLAlchemy`，只需在配置文件或者是定义文件中进行少量的修改，即可实现底层数据存储应用的替换。

#### 2. 作为接口定义的interface

Golang中的`interface`承担的主要作用之一就是解耦与多态，与Java中的接口没有什么区别，只是在语法格式上不同而已。Golang中的接口实现不需要显式地使用`implements`关键字，某个类型只需要实现了接口中的所有方法，就说该类型实现了该接口。

```go
type OperationLog struct {
	Timestamp int32    // 时间戳
	Operation string   // 操作类型, create/update/delete
	// Other need fields
}

type OperationLogService interface {
	InsertOne(log *OperationLog) error      // 单个日志记录
	InsertMany(logs []*OperationLog) error  // 批量日志记录
}

type ElasticLogService struct {
	EsClient *ElasticClient
}

func (svc *ElasticLogService) InsertOne(log *OperationLog) error {
	svc.EsClient.PostOne(/*参数省略*/)
	return nil
}

func (svc *ElasticLogService) InsertMany(logs []*OperationLog) error {
	svc.EsClient.PostMany(/*参数省略*/)
	return nil
}

type MySQLLogService struct {
	Db *MySQLClient
}

func (svc *MySQLLogService) InsertOne(log *OperationLog) error {
	svc.Db.InsertToLogTable(/*参数省略*/)
	return nil
}

func (svc *MySQLLogService) InsertMany(logs []*OperationLog) error {
	svc.Db.InsertManyToLogTable(/*参数省略*/)
	return nil
}

func main() {
	var logService OperationLogService
	// 使用MySQL作为data storage
	logService = &ElasticLogService{/*...*/}
	logService.InsertOne(/*...*/)

	// 使用ES作为data storage
	logService = &MySQLLogService{/*...*/}
	logService.InsertOne(/*...*/)
}
```

这是一个简单的操作日志服务demo，可以看到，操作日志具体的实现有Elasticsearch以及MySQL两种方式，而这两种DB对数据的CURD有着非常大的差异，前者使用http API，而后者则采用SQL语句。由于系统本身的复杂性既需要存储容量大、支持全局搜索但数据易失的ES，又需要存储容量有限、不支持全局搜索但数据持久化要求程度高的MySQL。

在有了接口对相关方法进行强约束以后，`ElasticLogService`以及`MySQLLogService`可以无差别的对外提供服务，需要切换服务时，只需进行非常小的改动即可满足需求。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/golang/interface/log_service.png)

#### 3. 作为类型的interface

当使用

```go
type MyInterface interface {}
```

时，我们定义了一个接口，当接口中不存在任何方法时，称之为empty interface，通常写做`interface{}`。由于Golang并没有显式的`implements`关键字，而所有的类型至少包含零个个方法，所以Golang中所有的类型都隐式地实现了`interface{}`，也就意味着，当我们定义如下方法时:

```go
func DoSomething(v interface{}) {...}
```

函数将能够接收任意的数据类型，不管是`int`还是`int64`，或者是自定义的结构体。以內建函数`Printf`为例，其原型为:

```go
func Printf(format string, a ...interface{}) (n int, err error) {...}
```

其中的`a`即为任意类型的任意数量的参数，如

```go
fmt.Printf("human eat %s\n", food)
fmt.Printf("The T-shirt price is %.2f, %s", 9.15, "nine fifteen")
```

回到`DoSomething`方法，在函数内部，`v`的类型是什么?假如传入的参数类型为`int`，`v`的类型是否就是`int`?答案是`interface{}`，不管传入的参数是什么类型，Go都会在必要时对其进行类型转换，转换成`interface{}`类型，而`interface{}`类型，是有值的。

接口的值分为两部分，一个指向底层方法表的指针，和指向保存着具体值的指针。

```go
type MyInterface interface {
    PrintSelf()
}

type MyInt int
func (this MyInt) PrintSelf() {
    fmt.Printf("The value is: %d", this)
}

func main() {
    var s MyInterface
}
```

此时`s`无具体的类型，也无具体的值，两个指针均指向nil:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/golang/interface/nil-interface.png)

当执行:

```go
s = MyInt(10)
s.PrintSelf()
```

时, `s`拥了的具体的类型和确切的值:

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/jojo/golang/interface/not%20nil%20interface.png)

虽然`interface{}`作为参数时可以接收任何类型的参数，但是并不代表参数的类型是任意类型，变量在运行时的某一时刻，永远有一个具体的类型。

```go
func DoSomething(v interface{}) {...}
```

在`DoSomething`方法内部，如果想要获取变量`v`的类型，可以使用类型分支(`switch-case`)来进行类型检查，也可以使用反射直接获取变量类型:

```go
func DoSomething(v interface{}) {
	// 使用类型分支
	switch v.(type) {
	case nil:
		fmt.Println("nil")
	case string:
		fmt.Println("string")
	case int:
		fmt.Println("int")
	}
	
	// 使用反射
	fmt.Printf("v'type is: %T\n", v)
}
```

#### 4. 作为契约(协议)的interface

Golang语言本身并不支持泛型，尽管函数参数可以使用`interface{}`来接收任意类型的变量，但是对于`slice`而言，我们无法定义一个`[]interface{}`来接收任意类型的`[]T`，原因在于`T`和`interface{}`在存储空间中有着截然不同的表现形式。

```go
func PrintSlice(s []interface{}) {
	for _, v := range s {
		fmt.Println(v)
	}
}

func main() {
	var s []int
	for i := 0; i < 10; i++ {
		s = append(s, i)
	}
	PrintSlice(s)
}
```

在编译时期就会抛出异常:

```bash
cannot use s (type []int) as type []interface {} in argument to PrintSlice
```

尽然可以通过代码将`[]T`转换成`[]interface{}`，但是会带来一些效率上的损耗，并且很丑陋。解决此类问题的通用方法就是使用`interface`，如`sort.Interface`。

sort包提供了针对任意类型序列根据任意排序函数原地排序的功能，使用`sort.Interface`接口来指定通用排序算法和每个具体类型之间的协议。

```go
type Interface interface {
	Len() int            // 获取集合中元素个数的方法
	Less(i, j int) bool  // 排序的依据
	Swap(i, j int)       // 如何在集合中交换两个元素
}
```

简单地来说，只要类型实现了`sort.Interface`接口，就可以使用Golang内部提供的排序算法(根据元素排布动态地选择排序方式)，而无需自行实现。

```go
type Employee struct {
	Name string
	Salary int
}

type Employees []Employee

func (em Employees) Len() int{
	return len(em)
}

func (em Employees) Less(i, j int) bool {
	return em[i].Salary < em[j].Salary
}

func (em Employees) Swap(i, j int) {
	em[i], em[j] = em[j], em[i]
}

func main() {
	employees := Employees{
		Employee{"smart", 5000},
		Employee{"Aelam", 4500},
		Employee{"Lin", 8500},
	}

	sort.Sort(employees)
	fmt.Println(employees)
}
```

`sort.Sort`接收一个`sort.Interface`类型参数，而`Employees`实现了该接口，故在`sort.Sort`内部，可以完全不用管`Employees`的具体类型是什么，只需调用接口中定义好的方法，是接口和实现类型之间的契约。