---
layout: post
cover: false
navigation: true
title: Java基础编程(02)--接口与抽象类
date: 2018-11-30 02:49:09
tags: ['Java基础', '设计原则']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

在`Java`中， 我认为接口和抽象类是能够让"匠人"充分发挥其想象力和创造力的地方， 这两个类结构使得软件大师们能够编写出精美， 优雅和巧妙的代码。 而在我这种低端程序员手中， 它仅仅只是一个结构而已， 离品尝到其设计精髓不知还隔着多少座大山。

<!---more--->

#### 1. 抽象类
抽象类不能被实例化， 仅作为一个通用接口来操纵一系列的导出类。 在`Java`中提供了抽象方法： 仅有声明而没有方法体。
```java
public abstract void f();
```
包含抽象方法的类叫做抽象类， 如果一个类包含一个或多个抽象方法， 那么该类必须定义为抽象类(否则， 编译器会教你重新写代码)。

如果从一个抽象类继承， 并想创建新类的对象， 那么就必须为基类中的所有抽象方法提供方法定义。 如果不这样做的话， 导出类也是抽象类， 并且编译器会强制的要求使用`abstract`关键字来限定导出类。

```java
abstract class Shape {
    // 可以定义成员变量
    public String name = "biubiu";

    // 可以定义静态变量
    public static final String color = "blue";

    // 这是我们的抽象方法，没有函数体，只有函数定义
    public abstract void draw();

    // 成员方法
    public String getName() {
        return name;
    }

    // 构造方法也可以有
    Shape(){
        System.out.println("Shape....");
    }
}
```
除了不能实例化以外， 抽象类和普通的类并没有什么区别， 成员变量， 静态变量以及构造函数等等， 都可以在抽象类中进行定义， 并且子类也能够对其进行访问。

实现一个抽象类的方式是继承该抽象类， 并实现该抽象类中的所有抽象方法。


#### 2. 接口
接口在抽象化这个层面上比抽象类走的更远， 接口中只允许完全抽象的函数。在接口中可以添加一些字段(field)， 只不过这些字段均为`static` & `final` 。
```java
interface Men {

    String name = "smart";

    String getName();
}

public class SuperMen implements Men {

    public String getName() {
        // name = "keyerror"; 不能够改变interface中的Field
        return name;
    }

    public void printName() {
        System.out.println(name);
    }

    public static void main(String[] args) {
        SuperMen superMen = new SuperMen();
        superMen.printName();
    }
}
```
接口的定义极其简单， 将`class`替换成`interface`， 声明函数的定义即可。

接口表示"所有实现了该特定接口的类看起来都这样"，  并且我们可以通过实现不同的接口来达到多重继承的目的。

这里对接口和抽象类做一个比较：

| 特性 | 接口 | 抽象类
| - | - | - |
| 方法实现 | 无任何方法实现 | 可以有自己的方法实现
| 实现 | 子类使用implements来实现接口， 需要实现接口中的所有方法 | 子类extends抽象类， 实现父类的抽象方法即可。
| 构造器 | 不允许有构造器 | 可以有构造器
| 访问修饰符 | 只允许public修饰符存在 | 可以有private, public以及protected修饰符
| 类变量的定义 | 只有静态final变量可以进行定义 | 可以定义静态变量， 也可以定义成员变量
| 与普通类的区别 | 完全不同 | 除了不能实例化以外， 其余于普通类完全相同
| 多重继承 | 允许某一个类实现多个接口 | 只允许单根继承
| 添加新方法 | 接口添加新方法之后，所有实现了该接口的类都必须修改 | 抽象类可以提供一个默认的实现方式， 不必改变所有子类代码。|

#### 3. 软件设计的原则
既然在开篇提到了抽象类和接口的一个主要目的是用于软件设计， 那么不可避免就需要了解软件设计中有哪些常用的原则。

在`敏捷软件开发：原则、模式与实践`一书中， 列举了诸多被业界所公认的设计原则， 分别为： 单一职责原则， 开放-封闭原则， 里氏替换原则， 依赖倒置原则， 接口隔离原则。 本文将会对这5种设计原则进行理解和拓展。

#### 4. 单一职责原则
单一职责原则与我们常说的："一个函数或者一个类， 只做一件事情"有相似的地方：

> 就一个类而言， 应该仅有一个引起它变化的原因。

比如在获取一个url的html页面的函数中， 我们将所有的代码都写到里面：
```java
public void getUrl(String url) {
    /* 构建请求对象 */
    /* 发起请求 */
    /* 得到请求结果 */
    /* 对请求结果进行解码， 以及一些清洗 */
    /* 将结果保存至DB中 */
}
```

由于业务是多变的， 那么该函数可能会经常的发生变动， 在每次该代码的时候都需要理解这一行代码和下一行代码做了什么， 到后面代码会被改的非常混乱。 有人说我们可以加注释嘛， 文字注释不是注释代码的， 而是注释业务的， 代码才是最好的注释， 这一点非常重要。

将这一整个函数拆分为不同的函数， 每一个函数只做一件事情， 并给予恰当的函数名称， 函数名称还拥有解释代码的副作用， 一眼就能知道这个函数做了什么。

我们把视角从函数上升到一个类时也是一样的， 一个类不应该又是爬虫类， 又是获取代理IP的类， 应该将其拆分。


#### 5. 开放-封闭原则

> 软件实体(类， 模块， 函数等)应该对拓展开放， 而对修改关闭。

简单的来说就是当我们新增一个功能时， 不能对原有的代码直接进行修改， 而是对其进行拓展。 如果读者有接触过`Python`这门语言的话， 那么装饰器就是直接的体现了开放-封闭原则。

要想实现开放-封闭原则， 最为关键的一点就是抽象。 客户端在调用底层的方法时， 不直接持有底层的具体实现， 而是持有其抽象类或者接口。

在`Java`中， 装饰模式是对开放-封闭原则较为直接的体现， 该模式的具体实现请见`Java基础编程--I/O系统`。

#### 6. 里氏替换原则
里氏替换原则的原有声明非常的绕口， 所以这里就不再贴原有的声明， 放出我对其的理解。 里氏替换原则， 我认为就是子类对象能够随意的替换父类对象， 而程序逻辑不变。

首先举一个不符合里氏替换原则的例子：
```java
class Parent {
    public String getValue() { return "Biu"; }
}

class Child2 extends Parent {
    @Override
    public String getValue() {
        return "Po";
    }
}

public class LSP {
    public static void main(String[] args) {
        Parent parent = new Child2();
        if (parent.getValue() == "Biu") {
            System.out.println("True");
            return;
        }
        System.out.println("False");
    }
}
```

这里不是一个特别恰当的例子， `Child2`继承自`Parent`， 并覆盖了父类的`getValue`方法， 返回了一个与父类完全不同的字符串。 当我们的业务代码如同`main`方法中时(实际上该方法仅用于举例， 实际生产中没人会这么写)， 子类和父类的实例没有办法进行替换， 也就违背了里氏替换原则。

通常来讲我们使用继承， 一是为了代码的复用， 而是利用多态机制。

在使用代码复用时， 子类不应该覆盖父类的方法， 而应该使用其余的方法进行拓展， 这种情况下自然就不会出现问题。

在利用多态机制时， 更重要的是类的类型， 而不是类中的方法， 所以此时父类应该被定义为抽象类或者是接口， 具体选择哪一个视业务情况而定。


#### 7. 依赖倒置原则
依赖倒置原则， 其核心思想就是将代码划分成不同的层次， 使高层次模块不依赖于低层次模块， 两者应依赖其抽象。 抽象不应该依赖具体实现， 而是相反， 具体实现应该依赖抽象。

> 1. High-level modules should not depend on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on details. Details should depend on abstractions.

原则的原有声明很绕口， 又是依赖又是抽象的， 提取一些细节出来。 当高层次模块对低层次模块有依赖时(持有实例， 或者调用方法， 都称为依赖)， 高层次模块通常就是我们的主要业务逻辑代码， 需要有较强的复用性。 当产生上述依赖时， 应该将这种依赖进行拆分， 使这两个模块均依赖于同一个抽象。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-24%2016-53-36%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

图示就是这么个意思， 举一个现实生活中的例子： 程序员喜欢买键盘， 比如我， 就喜欢cherry轴的， 有时候会用红轴， 有时候会用黑轴， 都会拿来打代码：
```java
class Coder {
    private RedSwitch redSwitch;
    private BlackSwitch blackSwitch;
}

class RedSwitch {
    private String name = "红轴";
}

class BlackSwitch {
    private String name = "黑轴";
}
```

最开始我们使用直接实现的方式， `coder`类直接依赖轴体键盘的具体实现。 那么问题就来了， 我怎么能够进行任意的切换键盘？ 每一个键盘的特性是不一样的， 如何体现？ 这个时候直接依赖的问题就出现了， 因为耦合的非常严重， 导致灵活性大大降低， 所以我们对其进行分离。

```java
class Coder {
    private Switch aSwitch;

    public Switch getaSwitch() {
        return aSwitch;
    }

    public void setaSwitch(Switch aSwitch) {
        this.aSwitch = aSwitch;
    }
}


abstract class Switch {
    private String name;
}


class RedSwitch extends Switch{
    private String name = "红轴";
}


class BlackSwitch extends Switch{
    private String name = "黑轴";
}
```

我们在中间添加一个抽象类， 让`Coder`对象依赖该抽象类， 具体的轴体实现去继承该抽象类， 那么此时的灵活性就大大增强， 不管有多少种具体的实现， 上述代码都可以完成特定对象的注入。

这个其实就是依赖倒置原则， 我们也可以说是面向接口编程， 面向约定编程， 等等。


适配器模式可以说比较完整的体现了依赖倒置原则的内容， 适配器模式的主要核心概念就是在新接口和老接口之间进行适配， 让其协同完成工作。

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/2018-11-26%2011-16-03%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)

`services`和`NewService`是两个不同的接口， 但是现在我们需要使用新的实现来实现原有的功能， 而在`Application`类中依赖的是`services`接口， 所以适配器接口也要实现该接口， 然后使用代理的方式进行方法的覆盖， 最终达到方法改写的目的。

适配器模式同样也是依赖其接口， 而不依赖具体的实现。 所以说适配器模式对依赖倒置原则有一个较好的表达。

#### 8. 接口隔离原则
接口隔离原则与单一职责原则非常相似， 只不过该原则针对的对象是接口。 接口隔离原则说白了就是不要让一个接口太过于臃肿， 把接口设计成一个所有类方法的并集， 这样一来系统的耦合性就会非常的高， 而且是无意义的耦合。

一个接口只对应一个类， 或者是一小部分类， 剩余的类由一个或多个接口进行定义。

#### 9. 小结
在本篇文章中， 首先是对抽象类和接口进行了一个简单的介绍和比较， 后续又对5种设计原则进行了介绍和demo的演示， 而掌握这些设计原则仅仅只是摸到了设计的门槛而已， 还有根据这些设计原则所衍生的设计模式。 不管是设计原则也好， 还是设计模式也罢， 都需要根据不同的业务场景来进行选择。

讲实话， 这一部分关于设计的内容， 实操性和经验占了非常大的比重， 见的多了， 写的多了， 自然也就理解了。
