# 设计模式

## 7 大基本设计原则

| 标记 | 设计模式原则名称  |                     简单定义                     |
| :--: | :---------------: | :----------------------------------------------: |
| OCP  |     开闭原则      |              对扩展开放，对修改关闭              |
| SRP  |   单一职责原则    |       一个类只负责一个功能领域中的相应职责       |
| LSP  |   里氏代换原则    |  所有引用基类的地方必须能透明地使用其子类的对象  |
| DIP  |   依赖倒转原则    |          依赖于抽象，不能依赖于具体实现          |
| ISP  |   接口隔离原则    |      类之间的依赖关系应该建立在最小的接口上      |
| CARP | 合成/聚合复用原则 | 尽量使用合成/聚合，而不是通过继承达到复用的目的  |
| LOD  |    迪米特法则     | 一个软件实体应当尽可能少的与其他实体发生相互作用 |

详细内容参考链接：https://cloud.tencent.com/developer/article/1650116

<br>

## 单例模式



<br>

## 工厂模式

工厂模式用于封装和管理对象的创建，是一种创建型模式。

### 简单工厂模式

该模式对对象创建管理方式仅仅是简单的对不同类对象的创建进行了一层薄薄的封装。该模式通过向工厂传递类型来指定要创建的对象，其 UML 类图如下：

[![SimpleFactory](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/simpleFactory.png)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/simpleFactory.png)

**Phone 类**：手机标准规范类 (Abstract Product)

```java
public interface Phone {
    void ring();
}
```

**MiPhone 类**：小米手机（Product 1）

```java
public class MiPhone implements Phone {
    @Override
    public void ring() {
        System.out.println("MiPhone is ringing...");
    }
}
```

**IPhone 类**：苹果手机（Product 2）

```java
public class IPhone implements Phone {
    @Override
    public void ring() {
        System.out.println("IPhone is ringing...");
    }
}
```

**SimpleFactory 类**：手机工厂（Factory）

```java
public class SimpleFactory {
    private static final String MI_PHONE = "MiPhone";
    private static final String I_PHONE = "IPhone";

    public Phone build(String phone) {
        if (MI_PHONE.equalsIgnoreCase(phone)) {
            return new MiPhone();
        } else if (I_PHONE.equalsIgnoreCase(phone)) {
            return new IPhone();
        } else {
            return null;
        }
    }
}
```

**演示**：

```java
public class SimpleFactoryDemo {
    public static void main(String[] args) {
        SimpleFactory factory = new SimpleFactory();
        Phone miPhone = factory.build("MiPhone"); // MiPhone is ringing...
        miPhone.ring();
        Phone iPhone = factory.build("IPhone"); // IPhone is ringing...
        iPhone.ring();
    }
}
```



### 工厂方法模式

和简单工厂模式中工厂负责生产所有产品相比，工厂方法模式将生成具体产品的任务分发给具体的产品工厂，其 UML 类图如下：

[![MethodFactory](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/FactoryMethod.png)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/FactoryMethod.png)

**AbstractFactory 类**：生产不同产品的工厂的抽象类

```java
public interface AbstractFactory {
    Phone build();
}
```

**XiaoMiFactory 类**：生产小米手机的工厂（Concrete Factory 1）

```java
public class MiFactory implements AbstractFactory {
    @Override
    public Phone build() {
        return new MiPhone();
    }
}
```

**AppleFactory 类**：生产苹果手机的工厂（Concrete Factory 2）

```java
public class AppleFactory implements AbstractFactory{
    @Override
    public Phone build() {
        return new IPhone();
    }
}
```

**演示：**

```java
public class MethodFactoryDemo {
    public static void main(String[] args) {
        AbstractFactory appleFactory = new AppleFactory();
        appleFactory.build().ring(); // IPhone is ringing...
        AbstractFactory miFactory = new MiFactory();
        miFactory.build().ring(); // MiPhone is ringing...
    }
}
```



### 抽象工厂模式

上面两种模式不管工厂怎么拆分抽象，都只是针对一类产品 **Phone**（Abstract Product），如果要生成另一种产品，比如 PC，应该怎么表示呢？

最简单的方式是把工厂方法模式完全复制一份，用来生产 PC。但这样也就意味着我们要完全复制和修改 Phone 生产管理的所有代码，显然这是一个笨办法，并不利于扩展和维护。

抽象工厂模式通过在 AbstarctFactory 中增加创建产品的接口，并在具体子工厂中实现新加产品的创建，当然前提是子工厂支持生产该产品。否则继承的这个接口可以什么也不干。

[![AbstractFactory](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/abstractFactory.png)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/abstractFactory.png)

**AbstractFactory 类**：增加 PC 产品制造接口

```java
public interface AbstractFactory {
    Phone buildPhone();
    PC buildPC();
}
```

**MiFactory 类**：增加小米 PC 的制造（Concrete Factory 1）

```java
public class MiFactory implements AbstractFactory {
    @Override
    public Phone buildPhone() {
        return new MiPhone();
    }
    @Override
    public PC buildPC() {
        return new MiBook();
    }
}
```

**AppleFactory 类**：增加苹果 PC 的制造（Concrete Factory 2）

```java
public class AppleFactory implements AbstractFactory {
    @Override
    public Phone buildPhone() {
        return new IPhone();
    }

    @Override
    public PC buildPC() {
        return new MacBook();
    }
}
```

**演示**：

```java
public class AbstractFactoryDemo {
    public static void main(String[] args) {
        AbstractFactory appleFactory = new AppleFactory();
        appleFactory.buildPhone().ring(); // IPhone is ringing...
        appleFactory.buildPC().show(); // MacBook is running...
        
        AbstractFactory miFactory = new MiFactory();
        miFactory.buildPhone().ring(); // MiPhone is ringing...
        miFactory.buildPC().show(); // MiBook is running...
    }
}
```

<br>

## 门面模式（外观模式）

**【模式介绍】**

外部与一个子系统的通信必须通过一个统一的外观对象进行，为子系统中的一组接口提供一个一致的界面，外观模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。

[![Facade](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/facade.jpg)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/facade.jpg)

门面模式一共有两种角色：

* 门面角色：客户端调用这个角色的方法。此角色知晓相关的子系统的功能和责任。正常情况下，本角色会将所有从客户端发来的请求委派到相应的子系统中去。

* 子系统角色：可以同时有一个或者多个子系统。每个子系统都不是一个单独的类，而是一个类的集合。每一个子系统都可以被客户端直接调用，或者被门面角色直接调用。子系统并不知道门面的存在，对于子系统而言，门面仅仅是另一个客户端而已。



**【实践案例】**

假设我们下班回家后，一共需要打开三个灯，然后就是打开热水器烧水准备洗澡，在等待的过程还会打开电视机看新闻。用一般的方法来实现的话，代码就会是下面这个样子。

**电灯、热水器和电视机类**

```java
public class Light {
    public void open(){
        System.out.println("Light has been opened!");
    }
}

public class Heater {
    public void open(){
        System.out.println("Heater has been opened!");
    }
}

public class TV {
    public void open(){
        System.out.println("TV has been opened!");
    }
}
```

**演示：**

在实际使用的过程中，我们需要直接去创建各种需要的对象，并且调用各自的 open 方法，非常的复杂。

```java
public class FacadeDemo {
    public static void main(String[] args){
        Light light1 = new Light();
        Light light2 = new Light();
        Light light3 = new Light();
        Heater heater = new Heater();
        TV tv = new TV();
        light1.open();
        light2.open();
        light3.open();
        heater.open();
        tv.open();
    }
}
```

**门面类：**

当我们使用了门面模式后，用户将只创建一个门面类，并调用其对应的方法，来调度所有的开关。

```java
public class Facade {
    private Light light1, light2, light3;
    private Heater heater;
    private TV tv;

    public Facade() {
        light1 = new Light();
        light2 = new Light();
        light3 = new Light();
        heater = new Heater();
        tv = new TV();
    }

    public void open() {
        light1.open();
        light2.open();
        light3.open();
        heater.open();
        tv.open();
    }
}
```

**演示：**

```java
public class FacadeDemo2 {
    public static void main(String[] args) {
        Facade facade = new Facade();
        facade.open();
    }
}
```

<br>

## 策略模式

**【模式介绍】**

策略模式是对算法的包装，是把使用算法的责任和算法本身分割开来，委派给不同的对象管理。策略模式通常把一个系列的算法包装到一系列的策略类里面，作为一个抽象策略类的子类。用一句话来说，就是：“准备一组算法，并将每一个算法封装起来，使得它们可以互换”。

[![Strategy](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/Strategy.png)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/Strategy.png)

这个模式涉及到三个角色：

- **环境 (Context) 角色：**持有一个策略类的引用，最终给客户端调用。
- **抽象策略 (Strategy) 角色：**策略类，定义了一个公共接口，各种不同的算法以不同的方式实现这个接口，Context 使用这个接口调用不同的算法，一般使用接口或抽象类实现。
- **具体策略 (Concrete Strategy) 角色：**实现了 Strategy 定义的接口，包装了相关的算法和行为，提供具体的算法实现。



**【实践案例】**

设计一个贩卖各类书籍的电子商务网站的购物车系统。现在需要计算每本书的单价，本网站可能对所有的高级会员提供每本20%的促销折扣；对中级会员提供每本10%的促销折扣；对初级会员没有折扣。

根据描述，折扣是根据以下的几个算法中的一个进行的：

- 算法一：对初级会员没有折扣。
- 算法二：对中级会员提供10%的促销折扣。
- 算法三：对高级会员提供20%的促销折扣。

[![Strategy Example](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/strategyExample.png)](https://github.com/IvanBao97/Notes/blob/main/DesignPattern/NoteImgs/strategyExample.png)

代码地址：



**【使用场景】**

1. 多个类只区别在表现行为不同，可以使用Strategy模式，在运行时动态选择具体要执行的行为。
2. 需要在不同情况下使用不同的策略(算法)，或者策略还可能在未来用其它方式来实现。
3. 对客户隐藏具体策略(算法)的实现细节，彼此完全独立。

<br>

## 享元模式

