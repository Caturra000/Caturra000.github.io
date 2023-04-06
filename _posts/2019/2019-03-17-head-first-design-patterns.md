---
layout: post
title: Head First设计模式学习笔记
categories: [System Design]
---

本篇是head first设计模式的读书笔记，关键的定义我会摘录书里的原话（一般比较简短），自己思考的部分不一定很正确，有错误请指出（又没人看唉）
<!--more-->


PS.书上的设计原则和设计模式是混合编排的，因此本笔记也会跟随这种排版

# 策略模式

> 策略模式定义算法族，分别封装起来，让它们之间可以互相替换，此模式**让算法的变化独立于使用算法的客户**

## 关于继承/接口

在超类上添加的方法将会使得所有子类具备该方法，如果某一个子类不需要该方法，那么就要重写覆盖掉（空），如果多个方法不需要使用，那就要多次重写，显得很sb

如果改为利用接口实现这种难以预测的子类行为，仅在需要时才实现该接口，那代码重复的问题会更加严重，比如要修改同样100个子类的行为，那要修改100遍（现在JDK已经有default方法）

## *设计原则1：应用中需要变化的部分要单独封装，不可与不更改的代码耦合在一起*

这样在需要更改时只需改变单独封装的部分代码即可，使得应用更容易扩展

因此我们把一个类中需要更改的部分单独分开到一组新类中代表每个行为，方便在【运行时】动态地修改行为（由此我们需要利用多态）

## *设计原则2：针对接口编程，不针对实现编程*

也就是说原来的超类中已经把会更改的行为重新封装成新的行为类，行为类负责实现行为，不需要要原类来实现，只要确保该行为符合相应的interface即可，这样可以降低行为与固有的类的耦合程度

当然也可以使用抽象类，该原则的核心在于supertype

利用单独封装+针对接口的原则，我们可以轻易扩展行为，并且修改也不会影响到原有的类

## 实现上使用委托机制

1.原来的类中把变化的行为分别设为一个接口类和一个用于执行的方法，如`flyBehavior`接口和`performFly()`方法

2.要实现该行为，就在执行的方法中调用接口内部的实现，此时无需关注接口的具体对象是什么

3.如今只需关心该类所需要的具体行为是什么，依据需求可在运行时利用多态确定该行为接口的具体对象（编译时处理也可以，即把确定对象的过程放到子类的构造方法中去）

## 案例

一只抽象的鸭，不知道怎样飞和怎样叫

```java
public abstract class Duck {
    // 委托给行为类处理
    FlyBehavior flyBehavior;
    QuackBehavior quackBehavior;
    
    public void performFly() {
        flyBehavior.fly();
    }
    
    public void performQuack() {
        quackBehavior.quack();
    }
    // 所有的鸭子都会有的行为
    public void swim() {
        System.out.println("All ducks float!");
    }
    // 提供运行时可以更改的做法
    public void setFlyBehavior(FlyBehavior fb) {
        flyBehavior = fb;
    }
    
    public void setQuackBehavior(QuackBehavior qb){
        quackBehavior = qb;
    }
}
```

其中一个接口演示

```java
public interface FlyBehavior {
    public void fly();
}
```
其中一个实现接口的行为类演示
```java
public class FlyWithWings implements FlyBehavior {

    public void fly() {
        System.out.println("Fly With Wings!")l

    }

}
```

一只写死的鸭子

```java
public class MallardDuck extends Duck {
    
    public MallardDuck() {
        flyBehavior = new FlyWithWings();
        quackBehavior = new Quack();
    }
    ````
    public void display() {
        System.out.println("This is MarrlardDuck!");
    }
}
```

一只运行时改变行为的鸭子

```java
public class MiniDuckSimulator {
    public static void main(String[] args) {
        Duck modelDuck = new ModelDuck();
        modelDuck.setFlyBehavior(new FlyWithRocket());
        modelDuck.performFly();
    }
}
```

## *设计原则3：多用组合，少用继承*

继承的灵活性限制在编译期之内，而组合能在运行时发挥更强大的OO能力

update.19/03/23

我再梳理一下组合/委托/策略模式的关系

**委托通过组合的方式来实现，其目的是使组合具有与继承同样的复用功能**

组合：has-a的关系，A具有B，A使用B的方法，但A不知道B是具体是谁，交互时只能通过B拥有的接口来调用，可以认为是黑箱操作

委托：两个对象参与处理一个请求，接收请求的对象将操作委托给另一个对象（代理）

策略模式：强调算法族的封装和替换，使用者无需关注内部变化


# 观察者模式

> 观察者模式定义了对象之间的一对多依赖（Subject和Observers），当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新

## 常见观察者模式的设计

1.主题接口要求实现注册/删除/通知观察者的方法

2.观察者接口要求实现自行更新的方法

3.具体主题还可实现获取状态的方法

4.具体观察者可以是实现该接口的任意类，但它们必须注册主题

## *设计原则4：为了交互对象之间的松耦合设计而努力*

观察者模式的强大在于主题和观察者是松耦合的，它们依然可以交互，但彼此不知道具体的实现细节

这样的松耦合设计使得对象之间的依赖降到最低

## 案例

下面来设计一个天气通知和多个订阅者

用于主题的接口

```java
package com.design.observer;

public interface Subject {
    public void registerObserver(Observer o);
    public void removeObserver(Observer o);
    public void notifyObservers();
}
```

用于观察者的接口

```java
package com.design.observer;

public interface Observer {
    public void update(float temp, float humidity, float pressure);
}
```

用于天气显示公告板的接口

```java
package com.design.observer;

public interface DisplayElement {
    public void display();
}
```

主题的实现类

```java
package com.design.observer.impl;

import java.util.ArrayList;

import com.design.observer.Observer;
import com.design.observer.Subject;

public class WeatherData implements Subject {

    private ArrayList observers;
    private float temperature;
    private float humidity;
    private float pressure;
    
    public void registerObserver(Observer o) {
        observers.add(o);
    }

    public void removeObserver(Observer o) {
        int i = observers.indexOf(o);
        if(i >= 0) {
            observers.remove(i);
        }
    }

    public void notifyObservers() {
        for(int i = 0; i < observers.size(); i++) {
            Observer observer = (Observer)observers.get(i);
            observer.update(temperature, humidity, pressure);
        }
    }
    
    public void measurementsChanged() {
        notifyObservers();
    }
    
    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}
```

观察者的实现类

```java
package com.design.observer.impl;

import java.util.ArrayList;

import com.design.observer.Observer;
import com.design.observer.Subject;

public class WeatherData implements Subject {

    private ArrayList observers;
    private float temperature;
    private float humidity;
    private float pressure;
    
    public void registerObserver(Observer o) {
        observers.add(o);
    }

    public void removeObserver(Observer o) {
        int i = observers.indexOf(o);
        if(i >= 0) {
            observers.remove(i);
        }
    }

    public void notifyObservers() {
        for(int i = 0; i < observers.size(); i++) {
            Observer observer = (Observer)observers.get(i);
            observer.update(temperature, humidity, pressure);
        }
    }
    
    public void measurementsChanged() {
        notifyObservers();
    }
    
    public void setMeasurements(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}
```

这种写法有点粗糙，比如当订阅者订阅了多个主题时，到了需要更新的时候就不知道要update哪些，不过问题不大，稍微改改就能用了（利用instanceof），但Java已经为我们提供了一个`Observable`类（注意），可以直接使用一个设计良好的板子，但由于继承是单亲的，这样做是一种浪费，更何况还违背了多用组合少用继承的设计原则，因此多建议手写（又不难是吧）

# 装饰者模式

（给爱用继承的你）

> 装饰者模式动态地把责任附加在对象上，若要扩展功能，装饰者比继承更有弹性的替代方案

装饰者对象的类型反映了所装饰的对象

## *设计原则5：类应对扩展开放，对修改关闭（简称开闭原则）*

修改现有的代码可能会造成隐患，所以在接受新的功能时不应修改原有代码

装饰者和继承有相同的功能，那就是对新行为/功能的扩展

但他比继承更为体现出开闭原则，**扩展总是开放的，但却无法修改原有的Component**（相比继承对原有方法的复写）

## 案例

关于原书的案例，我觉得有点劝退，这里给出一个符合国人的例子：https://www.cnblogs.com/stonefeng/p/5679638.html

当然装饰者注重的不是cost()的嵌套（但确实很巧妙，这里通过扩展替代了修改），而是原有功能上的扩展（原书P92图一目了然）

## 实现上需要注意的地方

1.装饰者要带有被装饰者的引用，且和自身是相同的超类型（隐式地拥有相同的接口）

2.修改的实现是在被装饰者（委托给他）的行为前后添加新的行为来实现

3.装饰者与被装饰者满足树形关系，并非只是一对一



暂略

# 工厂模式

> 工厂方法模式定义了一个创建对象的接口，但由子类决定要实例化的类是哪一个。工厂方法让类把实例化推迟到子类。

> 抽象工厂模式提供一个接口，用于创建相关或依赖对象的家族，而不需要明确制定具体的类

工厂处理创建对象的细节，它能更好地遵循前面提到的对修改关闭/针对接口编程/封装变化的部分等OO原则

试想一个类型在`new`定义具体对象，但对象种类繁多，如果在创建时用多层`if-else`来判断具体类型，到后期的增删改会是非常麻烦的一件事，并且如果在多处出现这种情况，后果更是严重，因此，把创建的部分封装成一个工厂来专门处理这些细节是有必要的

工厂模式相关的有三大件：简单工厂 && 工厂方法模式 && 抽象工厂模式


## 关于简单工厂

1.简单工厂并不算设计模式

2.简单工厂把主方法（客户）中的具体对象创建过程分离，好处不仅是后期扩展时修改主体更改了，还有从多个客户（其他方法）中复用同一个工厂，更加OOP

3.常用的写法：一种写法是使用类，另一种是直接用静态方法（但无法继承改变行为）


## 关于工厂方法

1.简单工厂只处理一个类（强调一对一），而工厂方法在主类（抽象的Creator类）中声明抽象的创建方法，让创建的执行延迟到子类的实现中

2.工厂方法的创建过程依赖于抽象的产品类（依赖倒置）

3.工厂方法中创建者类和产品类都是抽象的（这样的做法是哪怕只有一个具体创建者，也能实现更优的解耦）

简单的说简单工厂就是一手包办的工厂，而工厂方法更是一个大的框架


工厂方法是抽象的，依赖子类来处理对象的创建，这样超类的代码与子类的创建就能实现解耦（你需要子类创建的对象具有一定的规范，这时候在抽象的超类中有具体的执行方法和抽象的工厂方法，这就实现了子类的区分与相应的约束，也就是说各个子类决定创建的对象有所不同，但执行方法都在超类的掌控之下）（语死早，见谅）

## 关于抽象工厂

1.Factory定义为接口，用于处理Pizza的原料（内部成员）

2.抽象的Pizza类中对原料的处理方法(preapre)也是抽象的

3.具体的Pizza实例中有处理原料的工厂的引用，用于具体prepare过程中（需要的原料必须来自工厂）

4.从3可看出与工厂方法相比，抽象工厂对Pizza的约束更大（原料的产生只能来自工厂），此时一个Pizza的诞生是`Pizza pizza = new XjbPizza(XjbIngredientFactory)`

5.抽象工厂时产品家族和具体工厂实现解耦（抽象工厂的做法是组合，而工厂方法是继承）

## *设计原则6：要依赖抽象，不要依赖具体类（依赖倒置原则）*

对于一个多个具体类的类，我们可以从中构造一个抽象类来管理多个具体类，然后让该类依赖于单一的抽象类

### 依赖倒置更好的解释

1.高层组件不依赖于低层组件（谁依赖于谁，这就定义了高低层）

2.高层组件和低层组件（均为具体）应依赖于抽象

**3.具体类对抽象类的依赖是通过接口来实现的**（这个初次看书时困扰我很久，这样确实是实现了倒置的依赖）

4.倒置在于低层组件依赖于高层的抽象，高层组件也依赖于相同的抽象

### 如何更好的倒置?

1.变量不持有具体类的引用(不要new)

2.不能继承于具体类(不要依赖具体类)

3.不要覆盖基类【已实现】的方法(基类的方法应该被所有子类所共享)

抽象工厂中产品类的部分的内部成员和内部实现也依赖于抽象（工厂方法并不在乎产品类的约束）

## Pizza案例的分析

### 简单工厂中

1.PizzaStore是具体的，Pizza是抽象的，由具体的factory直接实例化Pizza(实现Pizza接口的类)

2.factory类的对象是PizzaStore的成员

### 工厂方法中（使用类，继承子类并复写工厂方法）

1.PizzaStore是抽象的，它定义了调用方法(order)的流程

2.用于创建的**工厂方法**（create）是抽象的，由子类实现(protected abstract)

3.产品类的产生由实例化的Store复写方法来实现原料处理

4.工厂方法实现了客户与具体类型的解耦

### 抽象工厂中（使用对象，组合）

1.Factory定义为接口，用于处理Pizza的原料（内部成员）

2.抽象的Pizza类中对原料的处理方法(preapre)也是抽象的

3.具体的Pizza实例中有处理原料的工厂的引用，用于具体prepare过程中（需要的原料必须来自工厂）

4.从3可看出与工厂方法相比，抽象工厂对Pizza的约束更大（原料的产生只能来自工厂），此时一个Pizza的诞生是由继承PizzaStore的子类中的工厂方法(create)中`Pizza pizza = new XjbPizza(XjbIngredientFactory)`实现，pizza构造方法中获得抽象的IngredientFactory引用，在复写方法中调用工厂对原料进行初始化（那么为什么既要有抽象的产品又有抽象的工厂？因为抽象产品中的抽象原料的实例只能由抽象的工厂的实例来获得，通过组合实现解耦，而不需严格约束的另一部分原料/信息可以直接用setter设定）

5.抽象工厂时产品家族和具体工厂实现解耦（抽象工厂的做法是组合，而工厂方法是继承）

6.从4可看出其实抽象工厂和工厂方法是可以混合使用的

# 单例模式

> 单例模式确保一个类只有一个实例，并提供全局访问点

单例模式比较简单，不过应用广泛，比如线程池、缓存、各种硬件驱动都使用了单例模式

单例模式与静态的全局变量相比，更为节省资源，对一些对资源敏感的对象而言更是如此，因为单例模式只在使用到的时候才会创建对象（延迟实例化）

如何实现：构造函数私有化，创建一个静态的方法调用该构造函数获取实例，并加以判断（再次调用，如果private static的单例已经创建非null，那就直接返回），这样在单线程下是没有问题的

## 多线程下的解决方法

1.把获取实例的方法加上同步`synchronized`（性能堪忧）

2.不使用延迟实例化，静态成员变量直接new，获取实例直接返回

3.`volatile`+方法内的`synchronized`（只在没创建的时候才同步，返回不需同步） update.有大佬指出这依然会有问题？待修改

4.static

5.enum



# 命令模式

> 命令模式将请求封装成对象，这可以让你使用不同的请求、队列，或者日志请求来参数化其他对象
>
> 命令模式也可以支持撤销操作

命令模式将动作的请求者和动作的执行者两个对象解耦

利用命令对象，把请求封装成特定对象，执行时只需调用请求对象即可，调用者不需要知道具体实现（挺常见的8，但要注意流程不是想象的简单粗暴）

流程：

0.设置Command接口，包括execute()和undo()方法

1.Client创建Reciver和ConcreteCommand

2.Invoker通过调用setCommand(slot,onCommand,offCommand)设置请求（大概）

3.ConcreteCommand.execute()由Reciver.action()真正实现

细节：
0.当没有命令时，应使用空对象的概念

（略有疑问：Command和Invoker也是分离的，也就是说Invoker根本不知道它的哪个slot对应哪个命令，好像你有一个遥控器但不知道第i个按钮是干嘛的样子...orz）

（个人认为）进一步的说，Invoker只是一个容器，从头到尾都是Client在实际操作Invoker和Command，可以观察P212的例子琢磨一下，站在这个角度来看，上面提到的问题就是说是遥控器不知道它自己的按钮是干嘛的，操作它的Client还是知道的（真实工具人，闻者流泪）

# 适配器模式与外观模式

> 适配器模式将一个类的接口，转换成客期望的另一个接口
>
> 适配器让原本接口不兼容的类可以合作无间
>
> 外观模式提供一个统一的接口，用来访问子系统中的一群接口
>
> 外观定义了一个高层接口，让子系统更容易使用

简单的来说适配器模式就是用来解决兼容性问题的方案

## *设计原则：只和密友谈话（最少知识原则）*



# 模板方法模式

> 模板方法在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。
>
> 模板方法使子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤

## 基本的设计方法

对于两个或多个步骤相似的算法，我们可以为它们抽象出一个基类，把算法的主要步骤放在里面，并且将步骤一致的行为也封装在里面（不希望在子类中改变，因此都设为`final`，不同之处再由子类分别实现（在基类中提供抽象方法）

对模板方法进行挂钩：钩子`hook`作为方法声明在抽象类中，但只有空的或默认的实现。钩子的存在使得子类可对算法的不同点进行挂钩

## *设计原则：Don't call us, we'll call you（好莱坞原则）*

前面提到，钩子在基类中提供空或默认的实现，针对算法的不同点，我们要在子类重写该钩子方法，并且被基类所调用，这样就实现了挂钩（初次见识到这种写法是在Servlet的初始化过程，感觉特别机智）。

简单概括：

1.行为由父类控制，子类实现

2.封装不变，扩展可变

3.共同代码容易维护

在这里总结一下：

策略模式是 1.封装可互换的行为 2.使用委托来选择行为

工厂方法模式是 由子类决定实例化具体类

模板方法模式是 由子类决定实现算法中的步骤



# 迭代器与组合模式

> 迭代器模式提供一种方法顺序访问一个聚合对象的各个元素，而又不暴露其内部的表示
>
> 组合模式允许你将对象组合成树形结构来表示整体/部分，组合能让客户以一致的方式处理个别对象以及对象组合

迭代器模式依赖于迭代器接口，多个数据结构使用统一的接口来实现多态地进行操作，而不需要知道具体的数据结构，使用者不必关心内部实现的细节，从而达到解耦的目的，比如要实现一个遍历不知何种数据结构的方法，只需传入Iterator对象并调用各自实现好的`next()`即可，调用者不会知道内部是什么

在Java中已经有`Iterator`接口，需要支持遍历和删除（后者不需要时可抛出一个运行时异常）

## *设计原则：一个类应该只有一个引起变化的原因*

书上写的有点那啥，个人认为是类保持高内聚以降低出错的可能和排查错误的时间成本

当多个数据结构合并到一起用一个超类列表来统一处理时，我们便能用迭代器方便的对各种数据结构逐一地遍历或其他操作，但注意到这些数据结构之间的关系已经是线性的，如果要求某个数据结构中再多一层展开，我们必须依照树形关系来维护（比如map的map，list的list），树形+遍历，不难想到递归，但在此之间要先设好规定

在树形的关系中，某个对象到底是代表整体的数据结构`Composite`，还是仅代表数据结构中的某个元素`Leaf`，这是需要区分的，但我们为它们都设同一个超类`Component`，这样便能在添加/修改过程中对`Component`做一个统一个处理（一个巧妙的设计是`Component.add(Component)`），消除了部分和整体的区分，当然个体并不支持添加/删除子结构的操作，要抛出不支持的异常。设计好后依然可以用外在为线性的结构来封装各种数据结构，不必真的使用显式的树形结构（虽然确实是树形），详见代码（暂留orz）

简而言之，组合模式可以让对象的集合和个别的对象一视同仁

PS.Uva上的集合栈计算器的做法也有用到这种思想

附：空迭代器在设计上并不意味着null，只是`hasNext()`永远返回`false`

# 状态模式

> 状态模式允许对象在内部状态改变时改变它的行为，对象看起来好像修改了它的类

简单的状态可以用一个整数值甚至布尔值表示，对应不同的动作用逻辑判断就能完成各种业务，但是这样是不容易扩展的，每增删一个状态或动作都要大改

一个做法是设计中的每个状态都封装成一个类，每个类都实现一个表示广义上状态的`State`接口，里面声明了各种动作`handle()`，而在具体实现类的内部，保存操作这些动作的对象`Context`的引用，以方便`Context`对象状态的改变（`Context`存有多个预设的状态和当前的状态的对象,以及对应的动作请求`request`），修改状态的过程其实只是几个状态引用类的改变，只是一个简单组合，自身类并没有实质上的改变（不管什么时候，调用`request()`都会**委托给当前状态类**去处理），与委托相比，原来的逻辑判断需要预先得知所有的情况才能处理事务，而经过委托和组合后根本不需要对越来越复杂的需求修改原来的代码，只对扩展开放

与策略模式相比，状态的转移显得比较被动，因为它不是主动调用的，是替代逻辑判断来改变状态，而策略模式是用于替代继承来改变行为

注意，在这里调用者无法直接修改`Context`的当前状态，也就是调用者无法与状态得到直接联系（想想在这个模式中状态变更每一步都得经过`Context`的处理）

对于多个`Context`实例，预设的状态对象是允许共享的（但不得有自己的内部状态）



# 代理模式

> 代理模式为另一个对象提供一个替身或占位符以控制对这个对象的访问

在某种情况下一个对象无法直接访问这个对象，代理就能起到客户端和被代理对象的中介作用

常见的代理如远程代理，对象在不同的JVM堆内不能直接访问，因此双方需要代理对象各自都要多一步处理才能完成，还有虚拟代理用于代替开销大的对象

代理不需要对原对象做出改动，符合开闭原则

代理和被代理类实现相同的接口，客户端可直接调用，并且代理类还可以在委托到被代理类前后做一些额外的业务处理（日志、缓存etc）以及一些访问权限的控制（比如用instanceof判断一个类的来源）

简单的实现方法：在代理类的构造过程中保存被代理类的对象引用（接口方法（仅实现相同的接口） / 继承方法（继承被代理实现类））

静态代理和动态代理的区别：前者是在编译期完成的，后者是运行时完成的（JVM反射创建代理对象）

相比静态代理，动态代理显得更加的灵活，比如多一个类需要代理（静态的话委托也是没问题的（吧））、类多一个方法而不用修改原来的代理实现（必须依赖于运行时）

Java实现动态代理的方法：`java.lang.reflect`提供`Proxy`类和`InvocationHandler`接口（针对接口编程）

`public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)`

并且调用`Object invoke(Object proxy,Method method,Object[] args)`

说起来比较费劲，看了1h也觉得没啥劲，直接来个具体的栗子

## 栗子

```java

package com.design.proxy;

public interface UserDAO { // 被代理类和代理类共同的接口
    
    public abstract void add();

    public abstract void delete();     

    public abstract void update();    

    public abstract void find();
}

package com.design.proxy;

public class UserDAOImpl implements UserDAO {

    public void add() {
        System.out.println("add");
    }

    public void delete() {
        System.out.println("delete");
    }

    public void update() {
        System.out.println("update");
    }

    public void find() {
        System.out.println("find");
    }

}

package com.design.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

//需要实现的动态代理
public class InvocationHandlerImpl implements InvocationHandler{

    private Object obj;
    
    public InvocationHandlerImpl(Object obj) {
        this.obj = obj;
    }
    
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        System.out.println("Checking...");
        Object result = method.invoke(obj, args);
        System.out.println("Recording...");
        
        return result;
        
    }
    
}

package com.design.proxy;

import java.lang.reflect.Proxy;

//生成动态代理的简单工厂
public class ProxyFactory {
    public static UserDAO getProxy(UserDAO proxy) {
        return (UserDAO) Proxy.newProxyInstance(
                Thread.currentThread().getContextClassLoader(), 
                proxy.getClass().getInterfaces(), 
                new InvocationHandlerImpl(proxy));
    }
}

package com.design.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

//测试
public class TestProxy {
    
    public static void main(String[] args) {
        UserDAO ud = new UserDAOImpl();
        ud.add();
        ud.update();
        
        InvocationHandler h = new InvocationHandlerImpl(ud);
        UserDAO proxy = (UserDAO) Proxy.newProxyInstance(
                                            ud.getClass().getClassLoader(), 
                                            ud.getClass().getInterfaces(), 
                                            h);
        proxy.delete();
        
        ProxyFactory.getProxy(new UserDAOImpl()).find();
    }
}
```

更强大的方法：`cglib`

# 生成器模式

> 使用生成器模式封装一个·产品的构造过程，并允许按步骤构造

流程：

1.设计Builder接口，里面含有各种add()

2.Client存有一个Builder的具体实现类，并拥有构造步骤用的constructPlanner()

3.最后返回一个Planner

# 原型模式

> 通过复制现有实例还实现创建实例

比较easy，思想就是通过现有实现类创建新的实现类，具体就是实现clone

# 访问者模式

> 当你想要为一个对象的组合增加新的能力，且封装并不重要时，可采用访问者模式

书上比较简略，而且这个组合我觉得有点微妙

一方面，类内组合的对象需要添加新的方法，不可避免地对接口进行扩展，虽然符合开闭（不会引入旧的部分的BUG），但一旦扩展，你的所有组合对象（该接口的实现类）都要改动

另一方面，我们为这些组合添加的访问者来共同实现新增的功能（比如多个），但这仍可能需要接口的扩展，所有的类也要改动（此时如果是抽象类或者default的话利用模板方法可能会避免）

组合之间明显的树形关系变为更复杂的图，我认为这会对未来的进一步扩展带来麻烦


责任链模式：请求和接收解耦

蝇量模式：创建多个虚拟的对象

中介模式：多个对象间的解耦

备忘录模式：？这也能叫模式？？


--工厂模式篇重写，update 19/03/30

--修正了命令模式的错误认知orz，update 19/04/22

--to be continued:
0.补一下剩余的模式（GOF版本看不下去..）

1.整理书上的问题

2.横向对比

3.存在的缺点