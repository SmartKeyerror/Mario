---
layout: post
cover: false
navigation: true
title: Java基础编程(04)--常用的设计模式(01)
date: 2018-12-05 10:49:09
tags: ['Java基础', '设计模式']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


在前面`I/O`系统中介绍了装饰模式， 在`接口与抽象类`中介绍了适配器模式， 以及设计模式的基础， 设计原则。 设计模式其实并没有那么神秘， 那么复杂， 本质上仍然是六大设计原则的体现。 既然要写， 就把所有的设计模式统统讲完， 才有快感。 另外这篇文章同样也会结合`Python`语言中的设计模式一起进行梳理， 这样做会进一步的加深我们对设计模式的理解， 至少在我这里是这样的。

<!---more--->

#### 1. 单例模式
在众多的设计模式之中， 单例模式毫无疑问的是使用最为频繁的设计模式， 不管是在`Java`， 还是在`Python`中。 单例模式的含义是指在全局中仅有某一个对象的唯一实例， 例如日志记录对象。

##### 1.1 Java实现
在`Java`中， 单例模式的实现有很多种， 不过基本上都需要依赖静态变量以及私有的构造方法。

最简单的实现：
```java
public class SingletonClass {

    private volatile static SingletonClass singletonClass;

    private SingletonClass() {}

    public static SingletonClass getSingletonClass() {
        if (singletonClass == null) {
            singletonClass = new SingletonClass();
        }
        return singletonClass;
    }
}
```

在这种最简单的实现中， 使用了静态变量仅会被初始化一次的特性， 将对象的实例保存至静态变量中， 并通过静态方法将实例返回。 在静态方法中， 是一个`if-then`的结构， 很明显的是该方法是线程不安全的， 所以就有了线程安全版：

```java
public static synchronized SingletonClass getSingletonClass() {
    if (singletonClass == null) {
        singletonClass = new SingletonClass();
    }
    return singletonClass;
}
```

我们对整个方法进行同步， 这样一来既可以保证该方法的线程安全性。 但是`synchronized`最为一种重量级的锁， 在并发环境下所有的方法调用均为串行执行， 效率比较低， 所以我们需要尽可能的减少串行执行的线程数量， 采用双重校验锁的方式完成：

```java
public static  SingletonClass getSingletonClass() {
    /* 第一次校验是让实例已经被初始化之后直接返回 */
    if (singletonClass == null) {
        /* 如果此时singletonClass == null, 那么就需要线程安全的实例化对象 */
        synchronized (SingletonClass.class) {
            /* 再次判断, 此时为加锁判断, 保证变量不会被其它线程所修改, 即保持单例*/
            if (singletonClass == null)
                singletonClass = new SingletonClass();
        }
    }
    return singletonClass;
}
```

双重校验锁的内容在`Java并发编程--锁`中有提到， 可能那篇文章中的描述更容易被理解。 这种方式通常来讲是我使用最多的方式， 既能够保证线程安全性， 同时也有较好的性能。

除此之外， 还有2种较好的实现方式， 一种是使用静态内部类来实现：
```java
class NewSingletonClass {

    private static class SingletonContainer {
        private static final NewSingletonClass instance = new NewSingletonClass();
    }

    private NewSingletonClass() {}

    public static NewSingletonClass getInstance() {
        return SingletonContainer.instance;
    }
}
```

由于静态内部类的加载由JVM保证其线程安全性， 并且只有在调用内部类的静态变量时类才被加载， 所以这种写法也是线程安全性的， 并且代码比较简单。

最后一种写法就需要对`Java`的枚举类有一个比较深入的理解了， 在这里我们只需要知道创建一个枚举类型是线程安全的即可。

```java
enum Foo {
    INSTANCE;

    public void otherMethod() {
        System.out.println("Other methods..");
    }

    public static void main(String[] args) {
        /* 测试 */
        Foo foo = Foo.INSTANCE;
        foo.otherMethod();
    }
}
```

这种写法可能不是那么易懂， 但是的确要比上面所有的方式都简洁， 所以使用枚举类来实现单例已经称为了目前的主流。

##### 1.2 Python实现
在`Python`中， 最简单， 最直接的方式就是使用`.pyc`文件的单一初始化来实现， 说白了就是模块儿导入。

```python
class Singleton():
    pass

singleton_class = Singleton()

# other module
from singleton import singleton_class
```

这种方式用的最多(因为真的很简单)， 不过前提是没有特别的定制化需求情况下。

在有定制化的需求之下， 例如一个用于拥有某一个对象的单个实例， 模块的方式无法完成， 此时可以使用装饰器或者是`__new__`方法来实现。

`__new__`方法在`Python`中为一个类的构造方法， 默认返回一个类的实例。 而`__init__`方法则是在该实例上进行属性的添加。

```python
class SimpleSingleton(object):
    instance_dict = {}
    lock = threading.Lock()

    def __new__(cls, username):
        if username not in cls.instance_dict:
            with cls.lock:
                if username not in cls.instance_dict:
                    instance = object.__new__(cls)
                    cls.instance_dict[username] = instance
                    return instance
        return cls.instance_dict[username]
```

这里仍然是使用双重校验锁的方式来创建单例， 只不过我们在这里额外的添加了一个`username`标志， 每个用户一个实例。

```python
def singleton_decorator(cls):
    instances = {}

    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance
```

这是一个比较典型的单例模式的装饰器， 很多博客都是这么写的， 装饰器本质上还是一个函数调用， 有函数调用的地方就需要保证线程安全性， 这上面的这一种写法并没有做到这一点， 所以我认为这种写法是错误的， 是非线程安全的。

所以说还是需要加锁， 写法与`__new__`方法中的实现基本相同， 也是使用一个双重校验锁：
```python
def singleton_decorator(cls):
    instances = {}
    lock = threading.Lock()

    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            with lock:
                if cls not in instances:
                    instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance
```

此外还可以使用`__metaclass__`元类的方式来实现， 该加锁还是需要加锁， 没什么区别。

#### 2. 原型模式
原型模式的目的在于在创建重复对象时提高性能， 本质上其实是一种内存的拷贝。 在`Java`中是通过实现`clonable`接口实现， 而在`Python`中则是通过标准库的函数所实现的。

提高拷贝， 就不得不提到深拷贝与浅拷贝。 当一个对象包含了另一个对象的引用时， 浅拷贝仅拷贝引用， 深拷贝则拷贝所引用对象的内容。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-10-26%2015-16-16%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)


##### 2.1 Java实现
只需要实现`clonable`接口即可， 默认实现的是浅拷贝， 如果想要实现深拷贝的话：
```java
@Override
protected Object clone() throws CloneNotSupportedException {
    MyClass template = (MyClass) super.clone();
    /* created即为对象所引用的对象, 深克隆必须对该对象也进行克隆 */
    template.created = (Date) template.created.clone();
    return template;
}
```

通常来讲深拷贝不会使用原型模式来实现， 而是使用序列化的方式实现。

##### 2.2 Python实现
`Python`实现的话就简单多了， 直接调标准库的函数即可。
```python
from copy import copy, deepcopy

information = {
    "name": "smart",
    "phones": ["136", "138"]
}

new_information = copy(information)
new_information["phones"].append("139")

print(information["phones"])       # ['136', '138', '139']
print(new_information["phones"])   # ['136', '138', '139']


deep_information = deepcopy(information)
deep_information["phones"].append("137")

print(information["phones"])       # ['136', '138', '139']
print(deep_information["phones"])  # ['136', '138', '139', '137']
```

从上面的示例代码可以很清晰的看出浅拷贝与深拷贝之间的区别， 通常在工程实践中， 浅拷贝只有在我们明确的知道对象中仅包含基本数据类型时才会使用， 否则一律使用深拷贝的方式进行对象的复制。


#### 3. 策略模式
策略模式为我们提供了在运行时更改类的行为或者算法的功能， 例如`Python`中的`sort`， `json`函数， 通过传入一个匿名函数来改变排序方式或者是序列化方式。

##### 3.1 Java实现
这里以JDK源码为例， 在搜索文件时我们可以传入一个`FilenameFilter`对象来完成文件的指定搜索：
```java
File file = new File(".");
file.list(new FilenameFilter() {
    @Override
    public boolean accept(File dir, String name) {
        return name.endsWith(".java");
    }
});
```

`list`方法所接收的`FilenameFilter`对象就是一种策略， 来看一下具体的实现：
```java
public String[] list(FilenameFilter filter) {
    String names[] = list();
    if ((names == null) || (filter == null)) {
        return names;
    }
    List<String> v = new ArrayList<>();
    for (int i = 0 ; i < names.length ; i++) {
        /* 调用对象的accept方法， 若为true, 则添加至列表中 */
        if (filter.accept(this, names[i])) {
            v.add(names[i]);
        }
    }
    return v.toArray(new String[v.size()]);
}
```

`FilenameFilter`接口也比较简单：

```java
public interface FilenameFilter {
    boolean accept(File dir, String name);
}
```

只要实现了该接口的类， 都可以作为一种策略传入至`list`方法， 为代码提供了更多的灵活性。

##### 3.2 Python实现
`Python`这里以`json`函数为例， 我们首先定义2个对象：
```python
import json

class Phone(object):
    def __init__(self, brands, price):
        self.brands = brands
        self.price = price

class Student(object):
    def __init__(self, name, age, phone):
        self.name = name
        self.age = age
        # Student对象中持有Phone对象
        self.phone = phone

if __name__ == "__main__":
    phone = Phone("iphone", 7999)
    # 完成Student对象的创建
    student = Student("smart", 18, phone)
    # 尝试进行序列化
    json.dumps(student.__dict__)
```

在此时我们对`Student`对象直接调用`json.dumps`方法时会抛出一个`TypeError`， 告诉我们`Phone`类型不是可以被JSON序列化的， 所以在这个时候我们就需要传递一个策略进去：

```python
result = json.dumps(student.__dict__, default=lambda x: x.__dict__)
```

`json.dumps`方法同样可以接受一个策略， 参数名为`default`， 这里我们传入了一个匿名函数， 函数返回传入对象的`__dict__`属性， 当json在序列化遇到了`TypeError`时， 就会使用我们传递的策略尝试重新进行序列化。

策略模式在日常开发中使用的会比较多， 自己编写的机会并不是很多。 一般来说使用策略模式的情景还是比较明显的， 主要是满足客户端的多种定制化需求。

#### 4. 责任链模式
责任链模式有些类似于工作审批， 员工向组长提交审批， 组长向部门经理提交， 部门经理向总经理提交， 总经理直接处理， 不再向下传递。 如果该审批(例如请假2小时)组长能够直接处理， 那么审批不再向下传递。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-27%2014-37-18%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

##### 4.1 Java实现

责任链模式常见的类图如下：

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-27%2014-26-38%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

从这个模式的名称上我们可以大致的猜出应该会有类似于链表的结构存在系统中， 事实上也的确是这样。 通常来讲我们会用一个抽象类来定义一些基本的方法， 例如是否需要将请求提交至下一个处理器， 如何添加下一个处理器等方法。

```java
abstract class Handler {
    /* 持有下一个处理器对象 */
    private Handler nextHandler;
    /* level变量通常是用来判断是否需要继续往下执行处理器 */
    protected int level;

    public void setNextHandler(Handler nextHandler) {
        this.nextHandler = nextHandler;
    }

    public void handleMessage(int level, String message){
        if (this.level == level)  // 这里为了简便处理， 直接用的相等判断
            process(message);
            /* 当一个处理器处理完成之后， 是继续向下处理， 还是直接结束 */
            // return;
        if (nextHandler != null)
            /* 执行下一个处理器 */
            nextHandler.handleMessage(level, message);
    }

    public abstract void process(String message);
}
```

抽象的`Handler`类其实就是核心的设计思想了， 具体的处理器继承该抽象类， 实现抽象方法， 并添加一个接收`level`参数的构造器即可。

```java
class Handler1 extends Handler {
    Handler1(int level){ this.level = level; }

    @Override
    public void process(String message) { System.out.println("Handler1"); }
}
```

我们还需要为客户端提供一个设置好责任链的`Handler`对象， 隐藏细节：

```java
public Handler getChainHandler() {
    Handler handler1 = new Handler1(1);
    Handler handler2 = new Handler2(2);
    handler1.setNextHandler(handler2);
    return handler1;
}
```

##### 4.2 Python实现
讲实话我在`Python`中还真的没见过很明显的责任链模式， 可能是我源码看的还不够多， 但是有一个地方很像责任链， 那就是`Django`中的中间件(Middleware)处理。

在1.9.8这个版本中， 中间件还是继承`MiddlewareMixin`， 并实现`process_request`或者是`process_response`方法所实现的， 最新版本情况未知， 想必改动不会太大。

在`Django`中， 请求被实例化成为一个请求对象之后， 首先调用配置的中间件的`process_request`方法， 做一些事情， 例如安全检测， 获取`Cookie`， 获取当前请求用户等等。 在响应时`response`对象将会以相反的方向执行`process_response`方法。

源代码我就不贴了， 有点儿长。 基本上`Django`这种对请求的处理和责任链模式还是有相似之处的。


#### 5. 代理模式

`Nginx`的其中一个作用就是隐藏真实的服务器地址， 向外暴露Nginx服务器的域名以及IP， 这样一来可以提高真实服务器的安全性。 请求首先进入Nginx服务器， 再由负载均衡器转发至对应的真实服务器中， 在这个过程中， Nginx服务器就是一个代理服务器。

代理模式与上面的过程是一样的， 为用户提供一个`Proxy`对象， 用户直接与`Proxy`对象进行交互， `Proxy`对象再与真实对象进行交互。

##### 5.1 Java实现
静态代理： 类图如下， 首先有一个公共接口， 约束真实对象与代理对象的行为， 然后在代理对象中可以持有一个真实对象的实例， 客户端在调用方法时， 由代理对象调用真实对象方法。 静态代理是在编译器就知道了所代理的对象类型。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-27%2015-47-55%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

```java
/* 代理类和真实类的统一接口 */
interface Action { void move(); }

/* 真实类的实现 */
class RealAction implements Action {
    @Override
    public void move() { System.out.println("Action!"); }
}

class Proxy implements Action {
    private RealAction realAction;

    @Override
    public void move() {
        /* 这里做了一些简化处理, 测试的话不考虑实际使用 */
        if (realAction == null)
            realAction = new RealAction();
        /* 调用真实类的相关方法 */
        realAction.move();
        /* 代理类自己也可以做一些额外的事情 */
        System.out.println("代理类额外做的事情");
    }
}
```

静态代理可以在不修改原有类对象的前提下， 对类进行功能的拓展。 但是由于公用同一个接口， 使得在修改接口时需要至少修改2个类。

动态代理要比静态代理稍微复杂一些， 但是本质没有改变多少。

首先来看由JDK所提供的基于反射的代理类， `java.lang.reflect.Proxy`， 其中有一个很重要的方法：

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
```

根据javadoc， `loader`为一个类加载器， `interfaces`为代理类将要实现的一组接口对象所组成的列表， `h`是一个`InvocationHandler`对象。

```java
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args);
}
```

在解释这些对象之前首先运行一个demo：

```java
class ProxyHolder {
    private Object target;

    public ProxyHolder(Object target) {
        this.target = target;
    }

    public Object getProxyInstance() {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        Object result = method.invoke(target, args);
                        System.out.println("代理类额外做的事情");
                        return result;
                    }
                }
        );
    }
}

public class DynamicProxy {
    public static void main(String[] args) {
        Action realAction = new RealAction();
        Action proxyInstance = (Action) new ProxyHolder(realAction).getProxyInstance();
        proxyInstance.move();
    }
}
```

可以看到我们在创建代理类的时候完全使用的是反射的机制， 在此期间根本不知道要代理的对象是什么， 而是使用`Object`对象来表示的， 并且代理方法是在`invoke`方法中所实现的。 上面的代码就是JDK所提供的动态代理。

如果接口中有多个方法需要进行代理的话， 也可以在`invoke`方法中集中进行处理。 其中， 传入的`Method`对象会包含正在被调用的接口方法。

不管是JDK动态代理， 还是静态代理， 都需要一个类实现一个接口， 那么对于单独的类想要实现动态代理， 该如何去做？

`cglib`代理通过构建目标对象子类的方式实现动态代理， 从而实现对目标对象功能的拓展。 因为这种方式不属于JDK的设计模式， 所以说将会在`AOP的实现`文章中给出。

##### 5.2 Python实现
由于`Python`是一种弱类型语言， 所以说其代理模式的实现就要比`Java`灵活的多。

```python
class ProxyFactory(object):

    def __init__(self, target):
        self.target = target

    def __getattribute__(self, item):
        target = object.__getattribute__(self, "target")
        attr = object.__getattribute__(target, item)

        def wrapper(*args, **kwargs):
            result = attr(*args, **kwargs)
            print("动态代理做点儿其它事情")
            return result
        return wrapper

class Test(object):
    def foo(self):
        print("foo")

if __name__ == "__main__":
    test = Test()
    proxy = ProxyFactory(test)
    proxy.foo()
```

首先要说明一点， `__getattr__`与`__getattribute__`是两个不同的方法， 但是都用于获取类属性或者是方法。 当这两个方法同时被定义时， 仅会调用`__getattribute__`方法， 除非显示的使用`instance.__getattr__`方法。

一般来说， `__getattr__`会在访问类中不存在的属性时调用， 而`__getattribute__`方法则属于无条件调用， 不管有没有， 都会调用。 函数， 也算是一种属性， 所以说在调用`proxy.foo`方法时， 首先调用`__getattribute__`获取函数对象。


#### 6. 观察者模式
观察者模式有些类似于`Redis`的发布/订阅， 多个客户端订阅某一个频道， 当频道内的键发生变化时`Redis`通知订阅端相应的变化。 主不过观察者模式是在对象层面上的"发布/订阅"， 多个观察者同时监听某一个对象， 当对象发生变化时， 通知所有的观察者， 观察者根据相应的变化做出相应的反应。

##### 6.1 Java实现

类图如下， 没有什么很复杂的地方， 只需要将`Observer`设置成为抽象类即可， 以便于复用。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-27%2018-44-01%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

```java
class Subject {
    /* 用于存储所有的观察者 */
    protected List<Observer> list = new ArrayList<Observer>();

    /* 添加一个观察者 */
    public void addObserver(Observer observer) { list.add(observer); }

    /* 通知所有观察者, 在该方法中可以传递更多的参数 */
    protected void notifyAllObservers() {
        for (Observer observer : list) {
            observer.receive();
        }
    }

    public void changeStatus() {
        System.out.println("被观察对象发生了改变");
        notifyAllObservers();
    }
}

abstract class Observer {
    protected Subject subject;
    public abstract void receive();
}

class Observer1 extends Observer {
    public Observer1(Subject subject) {
        this.subject = subject;
        this.subject.addObserver(this);
    }

    @Override
    public void receive() { System.out.println("观察者1接受到了反馈"); }
}

/* 测试类 */
public class ObserverPattern {
    public static void main(String[] args) {
        Subject subject = new Subject();
        Observer1 observer1 = new Observer1(subject);
        subject.changeStatus();
    }
}
```

这里给出了基于上面类图所实现的观察者模式demo， 整体来看比较简单。 但是实际生产中的观察者模式远比这复杂。 首先要考虑的就是线程安全性， 其次观察者模式分为推模式和拉模式， 两种模式的实现有较大的区别。 另外需要考虑的就是对象的引用问题， `GC`如何处理， 是否需要使用弱引用。

工程实践中的观察者较为复杂， 代码也比较多， 所以更加完善的分析放于后续的博文中。

##### 6.2 Python实现
观察者模式最直接的实现是在`Django`的信号量中， 具体的源码分析同样会单独写一篇出来。


#### 7. 小结
在最初的计划中是想要一篇文章将所有的设计模式一次性梳理完， 但是一篇写完的话实在是太长， 不方便阅读， 所以还是将其拆分成几篇文章， 每篇写几个。
