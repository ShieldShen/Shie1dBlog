---
title:        "观察者模式"
description:  "最近会写一些关于响应式编程的东西，而Rx对于JVM的拓展RxJava，就是基于观察者模式来实现的，所以，先码一篇观察者模式的文章让大家温习一下。"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2017-07-05"
---
# 观察者模式
## 写在前面
首先！这篇文章是我当年看设计模式的时候做的笔记，原文链接在哪里我已经找不到了~~~ 所以记住， 这篇文章**不是原创！！！不是原创！！！ 侵删！侵删！**

观察者模式是使用频率最高的设计模式之一，它用于建立一种对象与对象之间的依赖关系，一个对象发生改变时将自动通知其他对象，其他对象将相应作出反应。在观察者模式中，发生改变的对象称为观察目标，而被通知的对象称为观察者，一个观察目标可以对应多个观察者，而且这些观察者之间可以没有任何相互联系，可以根据需要增加和删除观察者，使得系统更易于扩展。
## 定义
**观察者模式（Observer Pattern）**：定义对象之间的一种一对多依赖关系，使得每当一个对象状态发生改变时，其相关依赖对象皆得到通知并被自动更新。观察者模式的别名包括发布-订阅（Publish/Subscribe）模式、模型-视图（Model/View）模式、源-监听器（Source/Listener）模式或从属者（Dependents）模式。观察者模式是一种对象行为型模式。
### 图示
观察者模式结构中通常包括观察目标和观察者两个继承层次结构，其结构如图所示：
![观察者模式结构图](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/1341501815_4830.jpg "观察者模式结构图")
### 详细介绍
在观察者模式结构图中包含如下几个角色：
* Subject（目标）：目标又称为主题，它是指被观察的对象。在目标中定义了一个观察者集合，一个观察目标可以接受任意数量的观察者来观察，它提供一系列方法来增加和删除观察者对象，同时它定义了通知方法 notify()。目标类可以是接口，也可以是抽象类或具体类。
* ConcreteSubject（具体目标）：具体目标是目标类的子类，通常它包含有经常发生改变的数据，当它的状态发生改变时，向它的各个观察者发出通知；同时它还实现了在目标类中定义的抽象业务逻辑方法（如果有的话）。如果无须扩展目标类，则具体目标类可以省略。
* Observer（观察者）：观察者将对观察目标的改变做出反应，观察者一般定义为接口，该接口声明了更新数据的方法update()，因此又称为抽象观察者。
* ConcreteObserver（具体观察者）：在具体观察者中维护一个指向具体目标对象的引用，它存储具体观察者的有关状态，这些状态需要和具体目标的状态保持一致；它实现了在抽象观察者 Observer 中定义的 update() 方法。通常在实现时，可以调用具体目标类的 attach() 方法将自己添加到目标类的集合中或通过 detach() 方法将自己从目标类的集合中删除。
## 实现机制
**观察者模式描述了如何建立对象与对象之间的依赖关系，以及如何构造满足这种需求的系统。**观察者模式包含观察目标和观察者两类对象，一个目标可以有任意数目的与之相依赖的观察者，一旦观察目标的状态发生改变，所有的观察者都将得到通知。作为对这个通知的响应，每个观察者都将监视观察目标的状态以使其状态与目标状态同步，这种交互也称为发布-订阅（Publish-Subscribe）。观察目标是通知的发布者，它发出通知时并不需要知道谁是它的观察者，可以有任意数目的观察者订阅它并接收通知。
下面通过示意代码来对该模式进行进一步分析。首先我们定义一个抽象目标 Subject，典型代码如下所示：

```java
import java.util.*;
abstract class Subject {
    //定义一个观察者集合用于存储所有观察者对象
protected ArrayList observers<Observer> = new ArrayList();

//注册方法，用于向观察者集合中增加一个观察者
    public void attach(Observer observer) {
    observers.add(observer);
}

    //注销方法，用于在观察者集合中删除一个观察者
    public void detach(Observer observer) {
    observers.remove(observer);
}

    //声明抽象通知方法
    public abstract void notify();
}
```

具体目标类ConcreteSubject是实现了抽象目标类Subject的一个具体子类，其典型代码如下所示：

```java
class ConcreteSubject extends Subject {
    //实现通知方法
    public void notify() {
        //遍历观察者集合，调用每一个观察者的响应方法
        for(Object obs:observers) {
            ((Observer)obs).update();
        }
    }   
}
```

在具体观察者 ConcreteObserver 中实现了update()方法，其典型代码如下所示：

```java
class ConcreteObserver implements Observer {
    //实现响应方法
    public void update() {
        //具体响应代码
    }
}
```

在有些更加复杂的情况下，**具体观察者类 ConcreteObserver 的 update() 方法在执行时需要使用到具体目标类ConcreteSubject中的状态（属性）**，因此在 ConcreteObserver 与 ConcreteSubject 之间有时候还存在关联或依赖关系，在 ConcreteObserver 中定义一个 ConcreteSubject 实例，通过该实例获取存储在 ConcreteSubject 中的状态。如果 ConcreteObserver 的 update() 方法不需要使用到 ConcreteSubject 中的状态属性，则可以对观察者模式的标准结构进行简化，在具体观察者 ConcreteObserver 和具体目标 ConcreteSubject 之间无须维持对象引用。如果在具体层具有关联关系，系统的扩展性将受到一定的影响，增加新的具体目标类有时候需要修改原有观察者的代码，在一定程度上违反了“开闭原则”，但是如果原有观察者类无须关联新增的具体目标，则系统扩展性不受影响。
## Example
为了实现对象之间的联动，Sunny 软件公司开发人员决定使用观察者模式来进行多人联机对战游戏的设计，其基本结构如图所示：
![多人联机对战游戏结构图](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/1341503929_8319.jpg "多人联机对战游戏结构图")
在图中，AllyControlCenter 充当目标类，ConcreteAllyControlCenter 充当具体目标类，Observer 充当抽象观察者，Player 充当具体观察者。完整代码如下所示：

```java
import java.util.*;

//抽象观察类
interface Observer {
    public String getName();
    public void setName(String name);
    public void help(); //声明支援盟友方法
    public void beAttacked(AllyControlCenter acc); //声明遭受攻击方法
}

//战队成员类：具体观察者类
class Player implements Observer {
    private String name;

    public Player(String name) {
        this.name = name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    //支援盟友方法的实现
    public void help() {
        System.out.println("坚持住，" + this.name + "来救你！");
    }

    //遭受攻击方法的实现，当遭受攻击时将调用战队控制中心类的通知方法notifyObserver()来通知盟友
    public void beAttacked(AllyControlCenter acc) {
        System.out.println(this.name + "被攻击！");
        acc.notifyObserver(name);       
    }
}

//战队控制中心类：目标类
abstract class AllyControlCenter {
    protected String allyName; //战队名称
    protected ArrayList<Observer> players = new ArrayList<Observer>(); //定义一个集合用于存储战队成员

    public void setAllyName(String allyName) {
        this.allyName = allyName;
    }

    public String getAllyName() {
        return this.allyName;
    }

    //注册方法
    public void join(Observer obs) {
        System.out.println(obs.getName() + "加入" + this.allyName + "战队！");
        players.add(obs);
    }

    //注销方法
    public void quit(Observer obs) {
        System.out.println(obs.getName() + "退出" + this.allyName + "战队！");
        players.remove(obs);
    }

    //声明抽象通知方法
    public abstract void notifyObserver(String name);
}

//具体战队控制中心类：具体目标类
class ConcreteAllyControlCenter extends AllyControlCenter {
    public ConcreteAllyControlCenter(String allyName) {
        System.out.println(allyName + "战队组建成功！");
        System.out.println("----------------------------");
        this.allyName = allyName;
    }

    //实现通知方法
    public void notifyObserver(String name) {
        System.out.println(this.allyName + "战队紧急通知，盟友" + name + "遭受敌人攻击！");
        //遍历观察者集合，调用每一个盟友（自己除外）的支援方法
        for(Object obs : players) {
            if (!((Observer)obs).getName().equalsIgnoreCase(name)) {
                ((Observer)obs).help();
            }
        }       
    }
}
```

编写如下客户端测试代码：

```java
class Client {
    public static void main(String args[]) {
        //定义观察目标对象
AllyControlCenter acc;
        acc = new ConcreteAllyControlCenter("金庸群侠");

        //定义四个观察者对象
        Observer player1,player2,player3,player4;

        player1 = new Player("杨过");
        acc.join(player1);

        player2 = new Player("令狐冲");
        acc.join(player2);

        player3 = new Player("张无忌");
        acc.join(player3);

        player4 = new Player("段誉");
        acc.join(player4);

        //某成员遭受攻击
        Player1.beAttacked(acc);
    }
}
```

编译并运行程序，输出结果如下：

```
金庸群侠战队组建成功！
----------------------------
杨过加入金庸群侠战队！
令狐冲加入金庸群侠战队！
张无忌加入金庸群侠战队！
段誉加入金庸群侠战队！
杨过被攻击！
金庸群侠战队紧急通知，盟友杨过遭受敌人攻击！
坚持住，令狐冲来救你！
坚持住，张无忌来救你！
坚持住，段誉来救你！
```

在本实例中，实现了两次对象之间的联动，当一个游戏玩家 Player 对象的 beAttacked() 方法被调用时，将调用 AllyControlCenter 的 notifyObserver() 方法来进行处理，而在 notifyObserver() 方法中又将调用其他 Player 对象的 help() 方法。Player 的beAttacked()方法、AllyControlCenter 的 notifyObserver() 方法以及 Player 的 help() 方法构成了一个联动触发链，执行顺序如下所示：

```
Player.beAttacked() --> AllyControlCenter.notifyObserver() -->Player.help()。
```

### JDK 对观察者模式的支持
观察者模式在 Java 语言中的地位非常重要。在 JDK 的 java.util 包中，提供了 Observable 类以及 Observer 接口，它们构成了 JDK 对观察者模式的支持。如图所示：
![JDK提供的Observable类及Observer接口结构图](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/1341504430_1842.jpg "JDK提供的Observable类及Observer接口结构图")
1. Observer 接口
在 java.util.Observer 接口中只声明一个方法，它充当抽象观察者，其方法声明代码如下所示： void update(Observable o, Object arg);
当观察目标的状态发生变化时，该方法将会被调用，在 Observer 的子类中将实现 update() 方法，即具体观察者可以根据需要具有不同的更新行为。当调用观察目标类 Observable 的 notifyObservers() 方法时，将执行观察者类中的 update() 方法。
2.  Observable 类
java.util.Observable类充当观察目标类，在Observable中定义了一个向量Vector来存储观察者对象，它所包含的方法及说明见表：
![Observable类所包含方法及说明](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/212928.jpg "Observable类所包含方法及说明")
我们可以直接使用 Observer 接口和 Observable 类来作为观察者模式的抽象层，再自定义具体观察者类和具体观察目标类，通过使用 JDK 中的 Observer 接口和 Observable 类，可以更加方便地在 Java 语言中应用观察者模式。
### 观察者模式与 Java 事件处理
JDK 1.0 及更早版本的事件模型基于职责链模式，但是这种模型不适用于复杂的系统，因此在 JDK 1.1 及以后的各个版本中，事件处理模型采用基于观察者模式的委派事件模型（DelegationEvent Model, DEM），即一个 Java 组件所引发的事件并不由引发事件的对象自己来负责处理，而是委派给独立的事件处理对象负责。

在 DEM 模型中，目标角色（如界面组件）负责发布事件，而观察者角色（事件处理者）可以向目标订阅它所感兴趣的事件。当一个具体目标产生一个事件时，它将通知所有订阅者。事件的发布者称为事件源（Event Source），而订阅者称为事件监听器（Event Listener），在这个过程中还可以通过事件对象（Event Object）来传递与事件相关的信息，可以在事件监听者的实现类中实现事件处理，因此事件监听对象又可以称为事件处理对象。**事件源对象、事件监听对象（事件处理对象）和事件对象构成了 Java 事件处理模型的三要素。**事件源对象充当观察目标，而事件监听对象充当观察者。以按钮点击事件为例，其事件处理流程如下：
1. 如果用户在 GUI 中单击一个按钮，将触发一个事件（如 ActionEvent 类型的动作事件），JVM 将产生一个相应的 ActionEvent 类型的事件对象，在该事件对象中包含了有关事件和事件源的信息，此时按钮是事件源对象；
2. 将 ActionEvent 事件对象传递给事件监听对象（事件处理对象），JDK 提供了专门用于处理 ActionEvent 事件的接口 ActionListener，开发人员需提供一个 ActionListener 的实现类（如MyActionHandler），实现在 ActionListener 接口中声明的抽象事件处理方法 actionPerformed()，对所发生事件做出相应的处理；
3. 开发人员将 ActionListener 接口的实现类（如 MyActionHandler）对象注册到按钮中，可以通过按钮类的 addActionListener() 方法来实现注册；
4. JVM在触发事件时将调用按钮的 fireXXX() 方法，在该方法内部将调用注册到按钮中的事件处理对象的 actionPerformed() 方法，实现对事件的处理。
使用类似的方法，我们可自定义 GUI 组件，如包含两个文本框和两个按钮的登录组件 LoginBean，可以采用如图所示设计方案：
![自定义登录组件结构图【省略按钮、文本框等界面组件】](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/1341504872_7751.jpg "自定义登录组件结构图【省略按钮、文本框等界面组件】")
相关类说明如下：
1. LoginEvent 是事件类，它用于封装与事件有关的信息，它不是观察者模式的一部分，但是它可以在目标对象和观察者对象之间传递数据，在 AWT 事件模型中，所有的自定义事件类都是 java.util.EventObject 的子类。
2. LoginEventListener 充当抽象观察者，它声明了事件响应方法validateLogin()，用于处理事件，该方法也称为事件处理方法，validateLogin() 方法将一个 LoginEvent 类型的事件对象作为参数，用于传输与事件相关的数据，在其子类中实现该方法，实现具体的事件处理。
3. LoginBean 充当具体目标类，在这里我们没有定义抽象目标类，对观察者模式进行了一定的简化。在 LoginBean 中定义了抽象观察者 LoginEventListener 类型的对象 lel 和事件对象 LoginEvent，提供了注册方法 addLoginEventListener() 用于添加观察者，在 Java 事件处理中，通常使用的是一对一的观察者模式，而不是一对多的观察者模式，也就是说，一个观察目标中只定义一个观察者对象，而不是提供一个观察者对象的集合。在LoginBean中还定义了通知方法 fireLoginEvent()，该方法在 Java 事件处理模型中称为“点火方法”，在该方法内部实例化了一个事件对象 LoginEvent，将用户输入的信息传给观察者对象，并且调用了观察者对象的响应方法 validateLogin()。
4. LoginValidatorA 和 LoginValidatorB 充当具体观察者类，它们实现了在 LoginEventListener 接口中声明的抽象方法 validateLogin()，用于具体实现事件处理，该方法包含一个 LoginEvent 类型的参数，在 LoginValidatorA 和 LoginValidatorB 类中可以针对相同的事件提供不同的实现。
### 观察者模式与MVC
在当前流行的MVC（Model-View-Controller）架构中也应用了观察者模式，MVC是一种架构模式，它包含三个角色：模型（Model），视图（View）和控制器（Controller）。其中模型可对应于观察者模式中的观察目标，而视图对应于观察者，控制器可充当两者之间的中介者。当模型层的数据发生改变时，视图层将自动改变其显示内容。如图所示：
![MVC结构示意图](http://wiki.jikexueyuan.com/project/design-pattern-behavior/images/1341505104_3429.jpg "MVC结构示意图")
在图中，模型层提供的数据是视图层所观察的对象，在视图层中包含两个用于显示数据的图表对象，一个是柱状图，一个是饼状图，相同的数据拥有不同的图表显示方式，如果模型层的数据发生改变，两个图表对象将随之发生变化，这意味着图表对象依赖模型层提供的数据对象，因此数据对象的任何状态改变都应立即通知它们。同时，这两个图表之间相互独立，不存在任何联系，而且图表对象的个数没有任何限制，用户可以根据需要再增加新的图表对象，如折线图。在增加新的图表对象时，无须修改原有类库，满足“开闭原则”。
### 扩展
大家可以查阅相关资料对 MVC 模式进行深入学习，如 Oracle 公司提供的技术文档《Java SE Application Design With MVC》，参考链接：[MVC 模式进行深入学习](http://www.oracle.com/technetwork/articles/javase/index-142890.html)。
## 总结
观察者模式是一种使用频率非常高的设计模式，无论是移动应用、Web 应用或者桌面应用，观察者模式几乎无处不在，它为实现对象之间的联动提供了一套完整的解决方案，凡是涉及到一对一或者一对多的对象交互场景都可以使用观察者模式。观察者模式广泛应用于各种编程语言的 GUI 事件处理的实现，在基于事件的XML解析技术（如 SAX2）以及 Web 事件处理中也都使用了观察者模式。
### 优点
1. 观察者模式可以实现表示层和数据逻辑层的分离，定义了稳定的消息更新传递机制，并抽象了更新接口，使得可以有各种各样不同的表示层充当具体观察者角色。
2. 观察者模式在观察目标和观察者之间建立一个抽象的耦合。观察目标只需要维持一个抽象观察者的集合，无须了解其具体观察者。由于观察目标和观察者没有紧密地耦合在一起，因此它们可以属于不同的抽象化层次。
3. 观察者模式支持广播通信，观察目标会向所有已注册的观察者对象发送通知，简化了一对多系统设计的难度。
4. 观察者模式满足“开闭原则”的要求，增加新的具体观察者无须修改原有系统代码，在具体观察者与观察目标之间不存在关联关系的情况下，增加新的观察目标也很方便。
### 缺点
1. 如果一个观察目标对象有很多直接和间接观察者，将所有的观察者都通知到会花费很多时间。
2. 如果在观察者和观察目标之间存在循环依赖，观察目标会触发它们之间进行循环调用，可能导致系统崩溃。
3. 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化。
### 适用场景
1. 一个抽象模型有两个方面，其中一个方面依赖于另一个方面，将这两个方面封装在独立的对象中使它们可以各自独立地改变和复用。
2. 一个对象的改变将导致一个或多个其他对象也发生改变，而并不知道具体有多少对象将发生改变，也不知道这些对象是谁。
3. 需要在系统中创建一个触发链，A 对象的行为将影响B对象，B 对象的行为将影响 C 对象……，可以使用观察者模式创建一种链式触发机制。