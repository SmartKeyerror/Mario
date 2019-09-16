---
layout: post
cover: false
navigation: true
title: Java基础编程(05)--常用的设计模式(02)
date: 2018-12-05 11:49:09
tags: ['Java基础', '设计模式']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---


在前面的一篇文章中大致了描述了单例模式， 原型模式， 策略模式， 责任链模式， 代理模式以及观察者模式， 本文接上。

<!---more--->

#### 7. 过滤器模式
过滤器模式有点儿类似于责任链模式， 只不过责任链模式是用于将请求和处理请求进行解耦， 而过滤器模式则更像一个滤网， 将符合我们要求的对象过滤出来。 为了更好的理解过滤器模式， 我们首先创建一个`Student`对象， 并赋予常见的属性：

```java
class Student {
    public String name;
    public int age;
    public boolean gender;

    public Student(String name, int age, boolean gender) {
        this.name = name;
        this.age = age;
        this.gender = gender;
    }
}
```

为了简化代码的书写， 这里的成员变量就直接使用`public`来进行修饰， 性别属性使用布尔值表示， `false`到底表示男还是女并不重要。 在另外一个地方保存了一组学生对象：

```java
List<Student> students = new ArrayList<Student>();
students.add(new Student("smart", 18, false));
students.add(new Student("foo", 20, true));
students.add(new Student("bar", 19, true));
```

现在我们想要对这一组学生进行一个数据分析， 比如找出年龄小于18岁的， 名字以's'开头的， 或者是性别为`false`的。 就有了第一版的代码：

```java
for (Student student : students) {
    if (student.name.startsWith("s"))
        System.out.println(student.name);
}
```

其余的筛选和这个基本上是一样的， 只不过筛选条件不同而已。 根据需求我们又写了根据年龄和性别筛选的代码。 然后又有需求来了， 筛选年龄小于19岁， 并且性别为`false`的学生， 好嘛， 那再加一个：

```java
for (Student student : students) {
    if ((!student.gender) && (student.age < 19))
        System.out.println(student.name);
}
```

这个时候就会发现代码重复的非常严重， 没有任何的可复用性， 不同的筛选规则不能够组合起来， 造成了大量重复的代码和臃肿的代码结构。

所以我们需要改变策略， 在每次筛选时直接创建一个新的容器对象， 筛选通过的对象添加至容器中， 并返回该容器。 这样一来我们就可以把多个筛选规则组合起来， 一个筛选器的结果是另一个筛选器的输入。

```java
public List<Student> filterAge(List<Student> students) {
    List<Student> filterList = new ArrayList<Student>();
    for (Student student : students) {
        if (student.age < 19)
            filterList.add(student);
    }
    return filterList;
}
```

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2010-41-09%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

该设计模式实在是没什么好说的， 比较简单， 所以就不再分`Java`实现和`Python`实现， 代码也比较简单， 就不贴了。

比较值得一提的就是Java 8中`Stream`以及`Python`中的列表推导式， 这两个特性很轻量， 也很实用。

#### 7.1 Stream
函数式编程提出已久， `Java`语言在1.8这个版本中才正式的推出了基于函数式编程的种种特性， 我们不得不正视这些特性， 因为有极大的可能函数式编程就是未来的编程首选。

Java 8 中的`Stream`是对集合对象功能的增强， 它专注于对集合对象进行各种非常便利， 高效的聚合操作， 或者是大批量的数据操作。 借助于新的`Lambda`表达式， 编程效率得到了非常大的提升。 另外， `Stream API`还提供了串行和并行两种模式进行汇聚操作， 我们可以使用`fork/join`来拆分任务并使用多线程的方法运行， 最终聚合形成结果， 提升系统性能。

那么上面我们写的年龄筛选， 就可以变成这样：

```java
List<Student> ageFilterList = students.parallelStream().
        filter(s -> s.age < 25).
        collect(Collectors.toList());
```

##### 7.2 列表推导式
`Python`中的列表推导式可谓是一大杀器， 威力极强， 代码逻辑非常清晰：

```python
age_list = [student for student in student_list if student.age < 25]
```

列表推导式还是很多花样可以玩儿， 在日常开发中使用的也比较多， 更多的内容如果读者感兴趣的话可以在`Google`中查阅相关资料(Ps: 不要用百度了)。


#### 8. 简单工厂模式
工厂模式的应用可以说是众多设计模式中应用最为广泛的一种， 提供了一种创建对象的便捷方法， 调用方并不需要知道一个对象的具体创建过程， 不必关系其内部的细节。 工厂模式常常会被分为简单工厂模式， 工厂模式， 以及抽象工厂模式， 在这一小节介绍简单工厂模式。

##### 8.1 Java实现
假设我们通过`socket`自己编写了一个`url`请求类， 层层封装， 使得客户端能够传入一个url参数， 并获得该url的HTML文本文档。 有时候用户会使用`https`开头的网址， 那么在此时我们就需要对SSL证书进行验证。 假设`http`与`https`这两种服务为两个不同的类实现， 但是它们有共同的接口：

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2012-02-02%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

```java
interface BaseHttp {
    /* 这里的返回做简化处理, 直接返回字符串, 不再构建response对象了 */
    String get();
    Boolean IsCertificateValid();
}

class Http implements BaseHttp {
    @Override
    public String get() { return "Http Response"; }

    @Override
    /* http服务不需要验证证书， 所以直接返回true */
    public Boolean IsCertificateValid() { return true; }
}


class Https implements BaseHttp {
    @Override
    public String get() { return "Https Response"; }

    public Boolean IsCertificateValid() {
        /*  这里再做一个简化处理, 返回false */
        return false;
    }
}

class HttpFactory {
    public static BaseHttp create(String url) {
        if (url.startsWith("https"))
            return new Https();
        if (url.startsWith("http"))
            return new Http();
        /* 就算是以ip为url, 也需要添加协议类型, 所以这里直接抛出异常 */
        throw new RuntimeException("Invalid url");
    }
}
```

那么客户端在使用`Http`和`Https`服务时， 不需要关心URL是什么类型， 传递URL之后由`HttpFactory`进行判断， 并返回不同的对象即可。

```java
public class SimpleFactoryPattern {
    public static void main(String[] args) {
        /* 客户端调用 */
        BaseHttp http = HttpFactory.create("http://google.com");
        BaseHttp https = HttpFactory.create("https://google.com");

        System.out.println(http.IsCertificateValid());
        System.out.println(https.IsCertificateValid());
    }
}
```

#### 9. 工厂模式

通常来讲简单工厂模式能够满足大部分的需求， 但是有一个问题， 那么就是假如又有一个`ftp`的服务， 通过传递`ftp://ip`来获取远程的文件， 这个时候我们就需要改动`HttpFactory`， 再增加一层判断。 这样一来就违反了开放-封闭原则。

注: FTP协议和HTTP协议不是一个东西， FTP协议也不是基于HTTP协议的， 这里这么来写， 完全是出于理解工厂方法。 读者需要注意。

实现开放-封闭原则的一个手段就是再往上抽象一层， 审视简单工厂的UML类图， `BaseHttp`已经是一个接口了， 无法再抽象； 而`HttpFactory`却是一个实体类， 可以继续进行抽象。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2014-06-05%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

我们将`HttpFactory`抽象成为`BaseFactory`， 根据该接口(或者是抽象类)实现`HttpFactory`， 用于创建`Http`对象， `HttpsFactory`创建`Https`对象， `FtpFactory`创建`Ftp`对象。 这样一来如果我们再增加一个服务， 我们只需要再实现一个工厂， 然后创建该服务即可。

```java
interface BaseFactory {
    /* 将原有的工厂类抽象化 */
    BaseHttp create();
}

class HttpFactory implements BaseFactory {
    @Override
    /* 在具体的实现类直接创建出相对应的服务对象 */
    public BaseHttp create() { return new Http(); }
}

public class FactoryPattern {
    public static void main(String[] args) {
        /* 客户端使用 */
        BaseFactory httpFactory = new HttpFactory();
        BaseHttp http = httpFactory.create();
        System.out.println(http);
    }
}
```

工厂模式要比简单工厂模式稍微复杂一点点， 唯一的区别就是将工厂类进行了抽象化， 从而解决添加服务需要修改原有工厂类代码的问题。

但是这样一来又有问题了， 假如我有10个服务类， 均通过每个具体的工厂类创建， 那么客户端的程序员就需要知道这10个工厂类， 并且还需要知道每一个工厂类是干什么的， 加重了客户端程序员的开发难度。 而对于简单工厂模式， 我们只需要知道这个工厂类可以产生我们想要的对象即可， 对客户端coder更加友好。

当我们的服务对象改变并不会特别频繁时， 使用简单工厂模式； 当改动非常频繁， 且为必须的时候， 使用工厂模式， 此时文档的注释需要及时更新。

#### 10. 抽象工厂模式
简单工厂模式用于解决多个相同类型的对象的创建问题， 使得客户端传入一个简单的参数就可以得到想要的对象； 工厂模式在简单工厂模式之上进一步的抽象， 用于实现开放-封闭原则。 而抽象工厂模式是基于工厂模式， 对具体的产类进一步的划分。 看类图就明白了。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2015-16-24%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

可以看到我们的产品类型增加了， 并且相应的工厂类也增加了一个创建对应产品的方法。 我们可以这样来理解：

Apple生产iphone， Airpods以及ipad; 微软生产windowsPhone， Headphones以及Surface， 类图如下：

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2015-29-24%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

我们对工厂模式的代码进行一个简单的修改就可以得到抽象工厂模式的代码， 这里就不再写了， 没有太大的难度。

抽象工厂主要解决的问题就是一个工厂可能会生产多种产品的问题， 将不同产品抽象， 再将不同的工厂进行抽象， 具体的工厂生产所对应的产品， 最终就形成了一个完整的生产链。

#### 11. 建造者模式
建造者模式类似于我们在玩儿RPG游戏时创建角色的过程： 首先角色的攻击类型， 有魔法攻击和物理攻击； 角色的服饰以及颜色， 角色的脸型， 角色的发型以及颜色等等， 然后我们在每个属性中挑一个出来， 就组成了我们的一个角色， 例如物理攻击， 青色汉服， 瓜子脸， 黑色短发。

建造者模式就是这样， 将一个复杂的对象拆开， 一步一步的创建单个对象， 最终再将这些对象组合在一起， 就形成了我们需要的对象。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-28%2016-44-42%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

这里以构建一个汽车为例， 从火花塞的绝缘体开始， 到火花塞， 到引擎， 类图如上所示。

```java
/**
 * 首先需要定义我们的产品类， 这里有绝缘体， 火花塞以及引擎
 */
class Product {
    private String insulator;

    private String sparkPlugs;

    private String engine;

    public void setInsulator(String insulator) { this.insulator = insulator; }

    public void setSparkPlugs(String sparkPlugs) { this.sparkPlugs = sparkPlugs; }

    public void setEngine(String engine) { this.engine = engine; }
}

/**
 * 建造者接口, 定义一个产品如何被构建
 */
interface Builder {
    void buildInsulator();

    void buildSparkPlugs();

    void buildEngine();

    Product getProduct();
}

/**
 * 具体的构建者实现类, 因为构建过程由该类实现, 所以在此处实例化一个Product对象
 */
class ConcreteBuilder implements Builder {

    private Product product = new Product();

    /* 下面主要是调用Product所提供的setter方法来进行构建 */

    @Override
    public void buildInsulator() { product.setInsulator("insulator"); }

    @Override
    public void buildSparkPlugs() { product.setSparkPlugs("sparkPlugs"); }

    @Override
    public void buildEngine() { product.setEngine("engine"); }

    @Override
    public Product getProduct() { return product; }
}

/**
 * Director决定构建顺序
 */
class Director {
    public static void construct(Builder builder) {
        builder.buildInsulator();
        builder.buildSparkPlugs();
        builder.buildEngine();
    }
}


public class BuilderPattern {
    public static void main(String[] args) {
        Builder builder = new ConcreteBuilder();
        Director.construct(builder);

        Product product = builder.getProduct();
        System.out.println(product);
    }
}
```

建造者模式不可避免的会造成编写的代码很长， 这是绕不过去的。 如果想要使用建造者模式的话， 就要做好代码很长的准备。

#### 12. 外观模式
外观模式有些类似于代理模式， 但是要比代理模式更进一步。 我觉得外观模式就是一个工具箱， 里面儿放着一堆乱七八糟的方法和对象， 在调用方法时实际上是调用了所持有对象的方法， 对外屏蔽了所有的细节。

类图并不是很想画， 因为这个模式从代码层面上来看是非常简单的。

```java
interface Animal { void eat(); }

class Human implements Animal {
    @Override
    public void eat() { System.out.println("Human eat"); }
}

class Lion implements Animal {
    @Override
    public void eat() { System.out.println("Lion eat"); }
}

class Cat implements Animal {
    @Override
    public void eat() { System.out.println("Cat eat"); }
}

/**
 * 这里就是我们的百宝箱, 里面保存着各种各样的对象, 以及方法, 傻瓜式方法调用
 */
class Tools {
    private Human human;
    private Lion lion;
    private Cat cat;

    public Tools() {
        this.human = new Human(); this.lion = new Lion(); this.cat = new Cat();
    }

    public void humanEat() { human.eat(); }
    public void lionEat() { lion.eat(); }
    public void catEat() { cat.eat(); }
}
```

外观模式怎么说呢， 给人一种奇怪的感觉， 如同一个百宝箱一样， 持有着各种各样的对象， 各种各样的方法， 有一种父亲给儿子在上学前的晚上在书包里面放置各种文具和生活用品的感觉。


#### 13. 小结
在本篇文章中， 介绍了过滤器模式， 简单工厂模式， 工厂模式， 抽象工厂模式， 建造者模式以及最后的外观模式， 加上前一篇文章的6种设计模式， 以及`I/O系统`这篇文章中提到的装饰模式和适配器模式， 总计14个。

本想将所有的设计模式一次性码完， 但是有些低估其数量以及编写的demo数量了。 一方面时间有限， 另一方面是剩下的设计模式并没有给我一种很足的"设计"感， 以及其应用并不会很广泛， 并且比较简单， 所以关于设计模式的梳理， 到这里就暂时告一段落， 下面的两篇文章将会着重的对代理模式和观察者模式进行进一步的梳理， 包括`Django`源码的重新阅读。

私以为理解设计模式的两个要点为理解面向对象编程以及6大设计原则， 当然在`Python`这种支持函数式编程的语言中， 种种设计原则将会进一步的被简化， 甚至被弱化。

在最开始笔者对设计模式可谓是一脸懵逼， 一眼看过去根本不知道在说些什么。 转机发生在我学习了UML类图以及设计原则之后， 如果有小伙伴儿想要开始学习设计模式， 但是不知道从哪儿入手的话， 可以从这两个方面着手。

最后推荐一本关于设计原则的书籍， 作者为 Robert C·Martin：

>  Agile Software Development: Principles, Patterns, and Practices
中文译本为: 敏捷软件开发--原则、模式与实践