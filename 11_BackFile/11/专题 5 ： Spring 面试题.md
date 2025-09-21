```
技术自由圈
```
# 牛逼的职业发展之路

40 岁老架构尼恩用一张图揭秘: Java 工程师的高端职业发展路径，走向食物链顶端的之路

链接：https://www.processon.com/view/link/618a2b62e0b34d73f7eb3cd


```
技术自由圈^
```
# 史上最全：价值 10 W 的架构师知识图谱

此图梳理于尼恩的多个 3 高生产项目：多个亿级人民币的大型 SAAS 平台和智慧城市项目

链接：https://www.processon.com/view/link/60fb9421637689719d


```
技术自由圈
```
# 牛逼的架构师哲学

40 岁老架构师尼恩对自己的 20 年的开发、架构经验总结

链接：https://www.processon.com/view/link/616f801963768961e9d9aec


```
技术自由圈
```
# 牛逼的 3 高架构知识宇宙

尼恩 3 高架构知识宇宙，帮助大家穿透 3 高架构，走向技术自由，远离中年危机

链接：https://www.processon.com/view/link/635097d2e0b34d40be778ab


```
技术自由圈
```
# 尼恩 Java 面试宝典

40 个专题（卷王专供+ 史上最全 + 2023 面试必备）
详情：https://www.cnblogs.com/crazymakercircle/p/13917138.html


```
技术自由圈^
```
# 未来职业，如何突围：三栖架构师


## 专题 5 ： Spring 面试题（史上最全、定期更

## 新）

#### 本文版本说明：V

```
尼恩面试宝典 ，早期叫做 《Java面试红宝书》
```
```
此文的格式，由markdown 通过程序转成而来，由于很多表格，没有来的及调整，出现一个格式
问题，尼恩在此给大家道歉啦。
```
```
由于社群很多小伙伴，在面试，不断的交流最新的面试难题，所以，《尼恩Java面试宝典》， 后
面会不断升级，迭代。
```
```
本专题，作为 《尼恩Java面试宝典》的第 10 个专题， 《尼恩Java面试宝典》一共 30 个面试专
题。
```
###### 升级说明：

**V 77 升级说明（2023-6-16）：**

大厂面试题：读过 Spring 源码吗？说说 Spring 事务是怎么实现的？

**V 3 升级说明（2022-5-16）：**

对 spring 三级缓存， **使用成品、半成品、原材料工厂，这样的浅显易懂的比方** ，使得复杂的概念，变
得更容易好懂

###### 《尼恩面试宝典》升级的规划为：


后续基本上， **每一个月，都会发布一次** ，最新版本，可以扫描扫架构师尼恩微信，发送 “领取电子书”
获取。

尼恩的微信二维码在哪里呢 ？ 请参见文末

###### 面试问题交流说明：

如果遇到面试难题，或者职业发展问题，或者中年危机问题，都可以来疯狂创客圈社群交流，

加入交流群，加尼恩微信即可，

尼恩的微信二维码在哪里呢 ？ 具体可以百度搜索 **疯狂创客圈总目录**

###### 什么是 spring?

Spring 是 **一个轻量级 Java 开发框架** ，最早有 **Rod Johnson** 创建，目的是为了解决企业级应用开发的业务
逻辑层和其他各层的耦合问题。它是一个分层的 JavaSE/JavaEE full-stack（一站式）轻量级开源框架，
为开发 Java 应用程序提供全面的基础架构支持。Spring 负责基础架构，因此 Java 开发者可以专注于应用
程序的开发。

Spring 最根本的使命是 **解决企业级应用开发的复杂性，即简化 Java 开发** 。

Spring 可以做很多事情，它为企业级开发提供给了丰富的功能，但是这些功能的底层都依赖于它的两个
核心特性，也就是 **依赖注入（dependency injection，DI）和面向切面编程（aspect-oriented
programming，AOP）** 。

为了降低 Java 开发的复杂性，Spring 采取了以下 4 种关键策略

```
基于POJO的轻量级和最小侵入性编程；
通过依赖注入和面向接口实现松耦合；
基于切面和惯例进行声明式编程；
通过切面和模板减少样板式代码。
```
###### Spring 框架的设计目标，设计理念，和核心是什么

**Spring 设计目标** ：Spring 为开发者提供一个一站式轻量级应用开发平台；

**Spring 设计理念** ：在 JavaEE 开发中，支持 POJO 和 JavaBean 开发方式，使应用面向接口开发，充分支持
OO（面向对象）设计方法；Spring 通过 IoC 容器实现对象耦合关系的管理，并实现依赖反转，将对象之
间的依赖关系交给 IoC 容器，实现解耦；

**Spring 框架的核心** ：IoC 容器和 AOP 模块。通过 IoC 容器管理 POJO 对象以及他们之间的耦合关系；通过
AOP 以动态非侵入的方式增强服务。

IoC 让相互协作的组件保持松散的耦合，而 AOP 编程允许你把遍布于应用各层的功能分离出来形成可重
用的功能组件。

###### Spring 的优缺点是什么？


优点

```
方便解耦，简化开发
Spring就是一个大工厂，可以将所有对象的创建和依赖关系的维护，交给Spring管理。
AOP编程的支持
Spring提供面向切面编程，可以方便的实现对程序进行权限拦截、运行监控等功能。
声明式事务的支持
只需要通过配置就可以完成对事务的管理，而无需手动编程。
方便程序的测试
Spring对Junit4支持，可以通过注解方便的测试Spring程序。
方便集成各种优秀框架
Spring不排斥各种优秀的开源框架，其内部提供了对各种优秀框架的直接支持（如：Struts、
Hibernate、MyBatis等）。
降低JavaEE API的使用难度
Spring对JavaEE开发中非常难用的一些API（JDBC、JavaMail、远程调用等），都提供了封装，使
这些API应用难度大大降低。
```
缺点

```
Spring明明一个很轻量级的框架，却给人感觉大而全
Spring依赖反射，反射影响性能
使用门槛升高，入门Spring需要较长时间
```
###### Spring 有哪些应用场景

**应用场景** ：JavaEE 企业应用开发，包括 SSH、SSM 等

**Spring 价值** ：

```
Spring是非侵入式的框架，目标是使应用程序代码对框架依赖最小化；
Spring提供一个一致的编程模型，使应用直接使用POJO开发，与运行环境隔离开来；
Spring推动应用设计风格向面向对象和面向接口开发转变，提高了代码的重用性和可测试性；
```
###### Spring 由哪些模块组成？

Spring 总共大约有 20 个模块，由 1300 多个不同的文件构成。而这些组件被分别整合在核心容器

（Core Container） 、 AOP（Aspect Oriented Programming）和设备支持（Instrmentation） 、

```
数据访问与集成（Data Access/Integeration） 、 Web、 消息（Messaging） 、 Test等 6 个模块
```
中。以下是 Spring 5 的模块结构图：


```
spring core：提供了框架的基本组成部分，包括控制反转（Inversion of Control，IOC）和依赖
注入（Dependency Injection，DI）功能。
spring beans：提供了BeanFactory，是工厂模式的一个经典实现，Spring将管理对象称为
Bean。
spring context：构建于 core 封装包基础上的 context 封装包，提供了一种框架式的对象访问方
法。
spring jdbc：提供了一个JDBC的抽象层，消除了烦琐的JDBC编码和数据库厂商特有的错误代码解
析， 用于简化JDBC。
spring aop：提供了面向切面的编程实现，让你可以自定义拦截器、切点等。
spring Web：提供了针对 Web 开发的集成特性，例如文件上传，利用 servlet listeners 进行 ioc
容器初始化和针对 Web 的 ApplicationContext。
spring test：主要为测试提供支持的，支持使用JUnit或TestNG对Spring组件进行单元测试和集成
测试。
```
###### Spring 框架中都用到了哪些设计模式？

```
1. 工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；
2. 单例模式：Bean默认为单例模式。
3. 代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；
4. 模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。
5. 观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的
对象都会得到通知被制动更新，如Spring中listener的实现–ApplicationListener。
```
###### 详细讲解一下核心容器（spring context 应用上下文) 模块

这是基本的 Spring 模块，提供 spring 框架的基础功能，BeanFactory 是任何以 spring 为基础的应用的
核心。Spring 框架建立在此模块之上，它使 Spring 成为一个容器。

Bean 工厂是工厂模式的一个实现，提供了控制反转功能，用来把应用的配置和依赖从真正的应用代码
中分离。最常用的就是 org. springframework. beans. factory. xml. XmlBeanFactory ，它根据 XML 文件
中的定义加载 beans。该容器从 XML 文件读取配置元数据并用它去创建一个完全配置的系统或应用。


###### Spring 框架中有哪些不同类型的事件

Spring 提供了以下 5 种标准的事件：

```
1. 上下文更新事件（ContextRefreshedEvent）：在调用ConfigurableApplicationContext 接口中
的refresh()方法时被触发。
2. 上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext的
Start()方法开始/重新开始容器时触发该事件。
3. 上下文停止事件（ContextStoppedEvent）：当容器调用ConfigurableApplicationContext的
Stop()方法停止容器时触发该事件。
4. 上下文关闭事件（ContextClosedEvent）：当ApplicationContext被关闭时触发该事件。容器被
关闭时，其管理的所有单例Bean都被销毁。
5. 请求处理事件（RequestHandledEvent）：在Web应用中，当一个http请求（request）结束触发
该事件。如果一个bean实现了ApplicationListener接口，当一个ApplicationEvent 被发布以后，
bean会自动被通知。
```
###### Spring 应用程序有哪些不同组件？

Spring 应用一般有以下组件：

```
接口 - 定义功能。
Bean 类 - 它包含属性，setter 和 getter 方法，函数等。
Bean 配置文件 - 包含类的信息以及如何配置它们。
Spring 面向切面编程（AOP） - 提供面向切面编程的功能。
用户程序 - 它使用接口。
```
###### 使用 Spring 有哪些方式？

使用 Spring 有以下方式：

```
作为一个成熟的 Spring Web 应用程序。
作为第三方 Web 框架，使用 Spring Frameworks 中间层。
作为企业级 Java Bean，它可以包装现有的 POJO（Plain Old Java Objects）。
用于远程使用。
```
#### Spring 控制反转 (IOC)（ 13 ）

###### 什么是 Spring IOC 容器？

控制反转即 IoC (Inversion of Control)，它把传统上由程序代码直接操控的对象的调用权交给容器，通
过容器来实现对象组件的装配和管理。所谓的“控制反转”概念就是对组件对象控制权的转移，从程序代
码本身转移到了外部容器。

Spring IOC 负责创建对象，管理对象（通过依赖注入（DI），装配对象，配置对象，并且管理这些对象
的整个生命周期。

###### 控制反转 (IoC) 有什么作用


```
管理对象的创建和依赖关系的维护。对象的创建并不是一件简单的事，在对象关系比较复杂时，如
果依赖关系需要程序猿来维护的话，那是相当头疼的
解耦，由容器去维护具体的对象
托管了类的产生过程，比如我们需要在类的产生过程中做一些处理，最直接的例子就是代理，如果
有容器程序可以把这部分处理交给容器，应用程序则无需去关心类是如何完成代理的
```
###### IOC 的优点是什么？

```
IOC 或 依赖注入把应用的代码量降到最低。
它使应用容易测试，单元测试不再需要单例和JNDI查找机制。
最小的代价和最小的侵入性使松散耦合得以实现。
IOC容器支持加载服务时的饿汉式初始化和懒加载。
```
###### Spring IoC 的实现机制

Spring 中的 IoC 的实现原理就是工厂模式加反射机制。

示例：

```
interface Fruit {
public abstract void eat();
}
```
```
class Apple implements Fruit {
public void eat(){
System.out.println("Apple");
}
}
```
```
class Orange implements Fruit {
public void eat(){
System.out.println("Orange");
}
}
```
```
class Factory {
public static Fruit getInstance(String ClassName) {
Fruit f=null;
try {
f=(Fruit)Class.forName(ClassName).newInstance();
} catch (Exception e) {
e.printStackTrace();
}
return f;
}
}
```
```
class Client {
public static void main(String[] a) {
Fruit f=Factory.getInstance("io.github.dunwu.spring.Apple");
if(f!=null){
f.eat();
}
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
```

###### Spring 的 IoC 支持哪些功能

Spring 的 IoC 设计支持以下功能：

```
依赖注入
依赖检查
自动装配
支持集合
指定初始化方法和销毁方法
支持回调某些方法（但是需要实现 Spring 接口，略有侵入）
```
其中，最重要的就是依赖注入，从 XML 的配置上说，即 ref 标签。对应 Spring
RuntimeBeanReference 对象。

对于 IoC 来说，最重要的就是容器。容器管理着 Bean 的生命周期，控制着 Bean 的依赖注入。

###### BeanFactory 和 ApplicationContext 有什么区别？

BeanFactory 和 ApplicationContext 是 Spring 的两大核心接口，都可以当做 Spring 的容器。

其中 ApplicationContext 是 BeanFactory 的子接口。

**依赖关系**

BeanFactory：是 Spring 里面最底层的接口，包含了各种 Bean 的定义，读取 bean 配置文档，管理 bean
的加载、实例化，控制 bean 的生命周期，维护 bean 之间的依赖关系。

ApplicationContext 接口作为 BeanFactory 的派生，除了提供 BeanFactory 所具有的功能外，还提供了
更完整的框架功能：

```
继承MessageSource，因此支持国际化。
统一的资源文件访问方式。
提供在监听器中注册bean的事件。
同时加载多个配置文件。
载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web
层。
```
**加载方式**

BeanFactroy 采用的是延迟加载形式来注入 Bean 的，即只有在使用到某个 Bean 时 (调用 getBean ())，才
对该 Bean 进行加载实例化。这样，我们就不能发现一些存在的 Spring 的配置问题。如果 Bean 的某一个
属性没有注入，BeanFacotry 加载后，直至第一次使用调用 getBean 方法才会抛出异常。

ApplicationContext，它是在容器启动时，一次性创建了所有的 Bean。这样，在容器启动时，我们就可
以发现 Spring 中存在的配置错误，这样有利于检查所依赖属性是否注入。 ApplicationContext 启动后
预载入所有的单实例 Bean，通过预载入单实例 bean ,确保当你需要的时候，你就不用等待，因为它们
已经创建好了。

相对于基本的 BeanFactory，ApplicationContext 唯一的不足是占用内存空间。当应用程序配置 Bean 较
多时，程序启动较慢。

**创建方式**

BeanFactory 通常以编程的方式被创建，ApplicationContext 还能以声明的方式创建，如使用
ContextLoader。

**注册方式**


BeanFactory 和 ApplicationContext 都支持 BeanPostProcessor、BeanFactoryPostProcessor 的使用，
但两者之间的区别是：BeanFactory 需要手动注册，而 ApplicationContext 则是自动注册。

###### Spring 如何设计容器的，BeanFactory 和 ApplicationContext 的

###### 关系详解

Spring 作者 Rod Johnson 设计了两个接口用以表示容器。

```
BeanFactory
ApplicationContext
```
BeanFactory 简单粗暴，可以理解为就是个 HashMap，Key 是 BeanName，Value 是 Bean 实例。通
常只提供注册（put），获取（get）这两个功能。我们可以称之为 **“低级容器”** 。

ApplicationContext 可以称之为 **“高级容器”** 。因为他比 BeanFactory 多了更多的功能。他继承了多个
接口。因此具备了更多的功能。例如资源的获取，支持多种消息（例如 JSP tag 的支持），对
BeanFactory 多了工具级别的支持等待。所以你看他的名字，已经不是 BeanFactory 之类的工厂了，
而是 “应用上下文”，代表着整个大容器的所有功能。该接口定义了一个 refresh 方法，此方法是所有
阅读 Spring 源码的人的最熟悉的方法，用于刷新整个容器，即重新加载/刷新所有的 bean。

当然，除了这两个大接口，还有其他的辅助接口，这里就不介绍他们了。

BeanFactory 和 ApplicationContext 的关系

为了更直观的展示 “低级容器” 和 “高级容器” 的关系，这里通过常用的
ClassPathXmlApplicationContext 类来展示整个容器的层级 UML 关系。

有点复杂？ 先不要慌，我来解释一下。

最上面的是 BeanFactory，下面的 3 个绿色的，都是功能扩展接口，这里就不展开讲。

看下面的隶属 ApplicationContext 粉红色的 “高级容器”，依赖着 “低级容器”，这里说的是依赖，不是继
承哦。他依赖着 “低级容器” 的 getBean 功能。而高级容器有更多的功能：支持不同的信息源头，可以
访问文件资源，支持应用事件（Observer 模式）。

通常用户看到的就是 “高级容器”。但 BeanFactory 也非常够用啦！


左边灰色区域的是 “低级容器”，只负载加载 Bean，获取 Bean。容器其他的高级功能是没有的。例如
上图画的 refresh 刷新 Bean 工厂所有配置，生命周期事件回调等。

小结

说了这么多，不知道你有没有理解 Spring IoC？ 这里小结一下：IoC 在 Spring 里，只需要低级容器就可
以实现， 2 个步骤：

```
1. 加载配置文件，解析成 BeanDefinition 放在 Map 里。
2. 调用 getBean 的时候，从 BeanDefinition 所属的 Map 里，拿出 Class 对象进行实例化，同时，
如果有依赖关系，将递归调用 getBean 方法 —— 完成依赖注入。
```
上面就是 Spring 低级容器（BeanFactory）的 IoC。

至于高级容器 ApplicationContext，他包含了低级容器的功能，当他执行 refresh 模板方法的时候，将
刷新整个容器的 Bean。同时其作为高级容器，包含了太多的功能。一句话，他不仅仅是 IoC。他支持
不同信息源头，支持 BeanFactory 工具类，支持层级容器，支持访问文件资源，支持事件发布通知，
支持接口回调等等。

###### ApplicationContext 通常的实现是什么？

**FileSystemXmlApplicationContext** ：此容器从一个 XML 文件中加载 beans 的定义，XML Bean 配置
文件的全路径名必须提供给它的构造函数。

**ClassPathXmlApplicationContext** ：此容器也从一个 XML 文件中加载 beans 的定义，这里，你需要正
确设置 classpath 因为这个容器将在 classpath 里找 bean 配置。

**WebXmlApplicationContext** ：此容器加载一个 XML 文件，此文件定义了一个 WEB 应用的所有 bean。

###### 什么是 Spring 的依赖注入？

控制反转 IoC 是一个很大的概念，可以用不同的方式来实现。其主要实现方式有两种：依赖注入和依赖
查找

依赖注入：相对于 IoC 而言，依赖注入 (DI) 更加准确地描述了 IoC 的设计理念。所谓依赖注入
（Dependency Injection），即组件之间的依赖关系由容器在应用系统运行期来决定，也就是由容器
动态地将某种依赖关系的目标对象实例注入到应用系统中的各个关联的组件之中。组件不做定位查询，
只提供普通的 Java 方法让容器去决定依赖关系。

###### 依赖注入的基本原则

依赖注入的基本原则是：应用组件不应该负责查找资源或者其他依赖的协作对象。配置对象的工作应该
由 IoC 容器负责，“查找资源”的逻辑应该从应用组件的代码中抽取出来，交给 IoC 容器负责。容器全权负
责组件的装配，它会把符合依赖关系的对象通过属性（JavaBean 中的 setter）或者是构造器传递给需要
的对象。

###### 依赖注入有什么优势

依赖注入之所以更流行是因为它是一种更可取的方式：让容器全权负责依赖查询，受管组件只需要暴露
JavaBean 的 setter 方法或者带参数的构造器或者接口，使容器可以在初始化时组装对象的依赖关系。其
与依赖查找方式相比，主要优势为：

```
查找定位操作与应用代码完全无关。
不依赖于容器的API，可以很容易地在任何容器以外使用应用对象。
不需要特殊的接口，绝大多数对象可以做到完全不必依赖容器。
```
###### 有哪些不同类型的依赖注入实现方式？


```
构造函数注入 setter 注入
```
```
没有部分注入 有部分注入
```
```
不会覆盖 setter 属性 会覆盖 setter 属性
```
```
任意修改都会创建一个新实例 任意修改不会创建一个新实例
```
```
适用于设置很多属性 适用于设置少量属性
```
依赖注入是时下最流行的 IoC 实现方式，依赖注入分为接口注入（Interface Injection），Setter 方法注
入（Setter Injection）和构造器注入（Constructor Injection）三种方式。其中接口注入由于在灵活
性和易用性比较差，现在从 Spring 4 开始已被废弃。

**构造器依赖注入** ：构造器依赖注入通过容器触发一个类的构造器来实现的，该类有一系列参数，每个参
数代表一个对其他类的依赖。

**Setter 方法注入** ：Setter 方法注入是容器通过调用无参构造器或无参 static 工厂方法实例化 bean 之后，
调用该 bean 的 setter 方法，即实现了基于 setter 的依赖注入。

###### 构造器依赖注入和 Setter 方法注入的区别

两种依赖方式都可以使用，构造器注入和 Setter 方法注入。最好的解决方案是用构造器参数实现强制依
赖，setter 方法实现可选依赖。

#### Spring Beans（ 19 ）

###### 什么是 Spring beans？

Spring beans 是那些形成 Spring 应用的主干的 java 对象。它们被 Spring IOC 容器初始化，装配，和管
理。这些 beans 通过容器中配置的元数据创建。比如，以 XML 文件中的形式定义。

###### 一个 Spring Bean 定义包含什么？

一个 Spring Bean 的定义包含容器必知的所有配置元数据，包括如何创建一个 bean，它的生命周期详情
及它的依赖。

###### 如何给 Spring 容器提供配置元数据？Spring 有几种配置方式

这里有三种重要的方法给 Spring 容器提供配置元数据。

```
XML配置文件。
基于注解的配置。
基于java的配置。
```
###### Spring 配置文件包含了哪些信息

Spring 配置文件是个 XML 文件，这个文件包含了类信息，描述了如何配置它们，以及如何相互调用。


###### Spring 基于 xml 注入 bean 的几种方式

```
1. Set方法注入；
2. 构造器注入：①通过index设置参数的位置；②通过type设置参数类型；
3. 静态工厂注入；
4. 实例工厂；
```
###### 你怎样定义类的作用域？

当定义一个在 Spring 里，我们还能给这个 bean 声明一个作用域。它可以通过 bean 定义中的 scope 属性
来定义。如，当 Spring 要在需要的时候每次生产一个新的 bean 实例，bean 的 scope 属性被指定为
prototype。另一方面，一个 bean 每次使用的时候必须返回同一个实例，这个 bean 的 scope 属性必须
设为 singleton。

###### 解释 Spring 支持的几种 bean 的作用域

Spring 框架支持以下五种 bean 的作用域：

```
singleton : bean在每个Spring ioc 容器中只有一个实例。
prototype ：一个bean的定义可以有多个实例。
request ：每次http请求都会创建一个bean，该作用域仅在基于web的Spring
ApplicationContext情形下有效。
session ：在一个HTTP Session中，一个bean定义对应一个实例。该作用域仅在基于web的
Spring ApplicationContext情形下有效。
global-session ：在一个全局的HTTP Session中，一个bean定义对应一个实例。该作用域仅在基
于web的Spring ApplicationContext情形下有效。
```
**注意：** 缺省的 Spring bean 的作用域是 Singleton。使用 prototype 作用域需要慎重的思考，因为频繁
创建和销毁 bean 会带来很大的性能开销。

###### Spring 框架中的单例 bean 是线程安全的吗？

不是，Spring 框架中的单例 bean 不是线程安全的。

spring 中的 bean 默认是单例模式，spring 框架并没有对单例 bean 进行多线程的封装处理。

实际上大部分时候 spring bean 无状态的（比如 dao 类），所有某种程度上来说 bean 也是安全的，但
如果 bean 有状态的话（比如 view model 对象），那就要开发者自己去保证线程安全了，最简单的就
是改变 bean 的作用域，把“singleton”变更为“prototype”，这样请求 bean 相当于 new Bean () 了，所
以就可以保证线程安全了。

```
有状态就是有数据存储功能。
无状态就是不会保存数据。
```
###### Spring 如何处理线程并发问题？

在一般情况下，只有无状态的 Bean 才可以在多线程环境下共享，在 Spring 中，绝大部分 Bean 都可以声
明为 singleton 作用域，因为 Spring 对一些 Bean 中非线程安全状态采用 ThreadLocal 进行处理，解决线
程安全问题。

ThreadLocal 和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。同步机制采用了“时间换
空间”的方式，仅提供一份变量，不同的线程在访问前需要获取锁，没获得锁的线程则需要排队。而
ThreadLocal 采用了“空间换时间”的方式。

ThreadLocal 会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为
每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal 提供了线程
安全的共享对象，在编写多线程代码时，可以把不安全的变量封装进 ThreadLocal。


###### 问题：解释 Spring 框架中 bean 的生命周期

```
注意，这个是重点问题
```
Spring Bean 的生命周期是 Spring 面试热点问题。这个问题即考察对 Spring 的微观了解，又考察对
Spring 的宏观认识，想要答好并不容易！本文希望能够从源码角度入手，帮助面试者彻底搞定 Spring
Bean 的生命周期。

**参考答案:**

首先，回答阶段的数量： **只有四个！**

```
是的，Spring Bean的生命周期只有这四个阶段。把这四个阶段和每个阶段对应的扩展点糅合在
一起虽然没有问题，但是这样非常凌乱，难以记忆。
```
要彻底搞清楚 Spring 的生命周期，首先要把这四个阶段牢牢记住。实例化和属性赋值对应构造方法和
setter 方法的注入，初始化和销毁是用户能自定义扩展的两个阶段。在这四步之间穿插的各种扩展点，
稍后会讲。

```
1. 实例化 Instantiation
2. 属性赋值 Populate
3. 初始化 Initialization
4. 销毁 Destruction
```
**实例化 -> 属性赋值 -> 初始化 -> 销毁**

**各个阶段的工作:**


```
1. 实例化，创建一个Bean对象
2. 填充属性，为属性赋值
3. 初始化
4. 如果实现了xxxAware接口，通过不同类型的Aware接口拿到Spring容器的资源
如果实现了BeanPostProcessor接口，则会回调该接口的
postProcessBeforeInitialzation和postProcessAfterInitialization方法
如果配置了init-method方法，则会执行init-method配置的方法
5. 销毁
6. 容器关闭后，如果Bean实现了DisposableBean接口，则会回调该接口的destroy方法
如果配置了destroy-method方法，则会执行destroy-method配置的方法
```
**源码学习：**

前三个阶段，主要逻辑都在 doCreate () 方法中，逻辑很清晰，就是顺序调用以下三个方法，这三个方法
与三个生命周期阶段一一对应，非常重要，在后续扩展接口分析中也会涉及。

```
1. createBeanInstance() -> 实例化
2. populateBean() -> 属性赋值
3. initializeBean() -> 初始化
```
```
注：bean的生命周期是从将bean定义全部注册到BeanFacotry中以后开始的。
```
源码如下，能证明实例化，属性赋值和初始化这三个生命周期的存在。关于本文的 Spring 源码都将忽略
无关部分，便于理解：

**前三个阶段的源码：**

上面这些这个实例化 Bean 的方法是在 getBean () 方法中调用的，而 getBean 是在
finishBeanFactoryInitialization 方法中调用的，用来实例化单例非懒加载 Bean，源码如下：

```
// 忽略了无关代码
protected Object doCreateBean(final String beanName, final
RootBeanDefinition mbd, final @Nullable Object[] args)
throws BeanCreationException {
// Instantiate the bean.
BeanWrapper instanceWrapper = null;
if (instanceWrapper == null) {
// 实例化阶段！
instanceWrapper = createBeanInstance(beanName, mbd, args);
}
// Initialize the bean instance.
Object exposedObject = bean;
try {
// 属性赋值阶段！
populateBean(beanName, mbd, instanceWrapper);
// 初始化阶段！
exposedObject = initializeBean(beanName, exposedObject, mbd);
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
```
```
@Override
public void refresh() throws BeansException, IllegalStateException {
synchronized (this.startupShutdownMonitor) {
```
```
1
2
3
```

**销毁 Bean 阶段:**

至于销毁，是在容器关闭时调用的，详见 ConfigurableApplicationContext #close ()

**高分答题的技巧:**

```
如果回答了上面的答案可以拿到 100 分的话，加上下面的内容，就是 120 分
```
**生命周期常用扩展点**

Spring 生命周期相关的常用扩展点非常多，所以问题不是不知道，而是记不住或者记不牢。其实记不住
的根本原因还是不够了解，这里通过源码+分类的方式帮大家记忆。

区分影响一个 bean 或者多个 bean 是从源码分析得出的.

以 BeanPostProcessor 为例：

```
1. 从refresh方法来看,BeanPostProcessor 实例化比正常的bean早.
2. 从initializeBean方法看,每个bean初始化前后都调用所有BeanPostProcessor的
postProcessBeforeInitialization和postProcessAfterInitialization方法.
```
**第一大类：影响多个 Bean 的接口**

```
try {
// Allows post-processing of the bean factory in context
subclasses.
postProcessBeanFactory(beanFactory);
// Invoke factory processors registered as beans in the
context.
invokeBeanFactoryPostProcessors(beanFactory);
// Register bean processors that intercept bean creation.
```
```
// 所有BeanPostProcesser初始化的调用点
registerBeanPostProcessors(beanFactory);
// Initialize message source for this context.
initMessageSource();
// Initialize event multicaster for this context.
initApplicationEventMulticaster();
// Initialize other special beans in specific context
subclasses.
onRefresh();
// Check for listener beans and register them.
registerListeners();
// Instantiate all remaining (non-lazy-init) singletons.
```
```
// 所有单例非懒加载Bean的调用点
finishBeanFactoryInitialization(beanFactory);
// Last step: publish corresponding event.
finishRefresh();
}
}
}
```
```
4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
```
```
18
19
20
21
22
23
24
25
26
27
28
29
```

实现了这些接口的 Bean 会切入到多个 Bean 的生命周期中。正因为如此，这些接口的功能非常强大，
Spring 内部扩展也经常使用这些接口，例如自动注入以及 AOP 的实现都和他们有关。

```
InstantiationAwareBeanPostProcessor
BeanPostProcessor
```
这两兄弟可能是 Spring 扩展中 **最重要** 的两个接口！InstantiationAwareBeanPostProcessor 作用于 **实例
化** 阶段的前后，BeanPostProcessor 作用于 **初始化** 阶段的前后。正好和第一、第三个生命周期阶段对
应。通过图能更好理解：

**InstantiationAwareBeanPostProcessor**

InstantiationAwareBeanPostProcessor 实际上继承了 BeanPostProcessor 接口，严格意义上来看他们
不是两兄弟，而是两父子。但是从生命周期角度我们重点关注其特有的对实例化阶段的影响，图中省略
了从 BeanPostProcessor 继承的方法。

```
1 InstantiationAwareBeanPostProcessor extends BeanPostProcessor
```

**InstantiationAwareBeanPostProcessor 源码分析：**

```
postProcessBeforeInstantiation调用点 ，忽略无关代码：
```
可以看到，postProcessBeforeInstantiation 在 doCreateBean 之前调用，也就是在 bean 实例化之前调
用的，英文源码注释解释道该方法的返回值会替换原本的 Bean 作为代理，这也是 Aop 等功能实现的关键
点。

```
postProcessAfterInstantiation调用点， 忽略无关代码：
```
```
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd,
@Nullable Object[] args)
throws BeanCreationException {
try {
// Give BeanPostProcessors a chance to return a proxy instead of
the target bean instance.
// postProcessBeforeInstantiation方法调用点，这里就不跟进了，
// 有兴趣的同学可以自己看下，就是for循环调用所有的
InstantiationAwareBeanPostProcessor
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
if (bean != null) {
return bean;
}
}
```
```
try {
// 上文提到的doCreateBean方法，可以看到
// postProcessBeforeInstantiation方法在创建Bean之前调用
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
if (logger.isTraceEnabled()) {
logger.trace("Finished creating instance of bean '" + beanName
+ "'");
}
return beanInstance;
}
```
```
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
```
```
20
21
22
23
24
```
```
protected void populateBean(String beanName, RootBeanDefinition mbd,
@Nullable BeanWrapper bw) {
// Give any InstantiationAwareBeanPostProcessors the opportunity to
modify the
// state of the bean before properties are set. This can be used, for
example,
// to support styles of field injection.
boolean continueWithPropertyPopulation = true;
```
```
// InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation()
// 方法作为属性赋值的前置检查条件，在属性赋值之前执行，能够影响是否进行属性赋值！
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
for (BeanPostProcessor bp : getBeanPostProcessors()) {
if (bp instanceof InstantiationAwareBeanPostProcessor) {
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```

可以看到该方法在属性赋值方法内，但是在真正执行赋值操作之前。其返回值为 boolean，返回 false 时
可以阻断属性赋值阶段（continueWithPropertyPopulation = false;）。

**BeanPostProcessor**

关于 BeanPostProcessor 执行阶段的源码穿插在下文 Aware 接口的调用时机分析中，因为部分 Aware 功
能的就是通过他实现的! 只需要先记住 BeanPostProcessor 在初始化前后调用就可以了。

**接口源码：**

**第二大类：只调用一次的接口**

这一大类接口的特点是功能丰富，常用于用户自定义扩展。

第二大类中又可以分为两类：

```
1. Aware类型的接口
2. 生命周期接口
```
**无所不知的 Aware**

```
InstantiationAwareBeanPostProcessor ibp =
(InstantiationAwareBeanPostProcessor) bp;
if
(!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
continueWithPropertyPopulation = false;
break;
}
}
}
}
```
```
// 忽略后续的属性赋值操作代码
}
```
```
12
```
```
13
```
```
14
15
16
17
18
19
20
21
22
```
```
public interface BeanPostProcessor {
//bean初始化之前调用
@Nullable
default Object postProcessBeforeInitialization(Object bean, String
beanName) throws BeansException {
return bean;
}
```
```
//bean初始化之后调用
@Nullable
default Object postProcessAfterInitialization(Object bean, String
beanName) throws BeansException {
return bean;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```
```
11
12
13
14
```

Aware 类型的接口的作用就是让我们能够拿到 Spring 容器中的一些资源。基本都能够见名知意，Aware
之前的名字就是可以拿到什么资源，例如 BeanNameAware 可以拿到 BeanName，以此类推。调用时机
需要注意： **所有的 Aware 方法都是在初始化阶段之前调用的！**

Aware 接口众多，这里同样通过分类的方式帮助大家记忆。

Aware 接口具体可以分为两组，至于为什么这么分，详见下面的源码分析。如下排列顺序同样也是
Aware 接口的执行顺序，能够见名知意的接口不再解释。

**Aware Group 1**

```
1. BeanNameAware
2. BeanClassLoaderAware
3. BeanFactoryAware
```
**Aware Group 2**

```
1. EnvironmentAware
2. EmbeddedValueResolverAware 这个知道的人可能不多，实现该接口能够获取Spring EL解析
器，用户的自定义注解需要支持spel表达式的时候可以使用，非常方便。
3. ApplicationContextAware(ResourceLoaderAware\ApplicationEventPublisherAware\Message
SourceAware) 这几个接口可能让人有点懵，实际上这几个接口可以一起记，其返回值实质上都
是当前的ApplicationContext对象，因为ApplicationContext是一个复合接口，如下：
```
这里涉及到另一道面试题，ApplicationContext 和 BeanFactory 的区别，可以从 ApplicationContext 继
承的这几个接口入手，除去 BeanFactory 相关的两个接口就是 ApplicationContext 独有的功能，这里不
详细说明。

**Aware 调用时机源码分析**

详情如下，忽略了部分无关代码。代码位置就是我们上文提到的 initializeBean 方法详情，这也说明了
Aware 都是在初始化阶段之前调用的！

```
public interface ApplicationContext extends EnvironmentCapable,
ListableBeanFactory, HierarchicalBeanFactory,
MessageSource, ApplicationEventPublisher, ResourcePatternResolver {}
```
```
1
```
```
2
```
```
// 见名知意，初始化阶段调用的方法
protected Object initializeBean(final String beanName, final Object bean,
@Nullable RootBeanDefinition mbd) {
// 这里调用的是Group1中的三个Bean开头的Aware
invokeAwareMethods(beanName, bean);
```
```
Object wrappedBean = bean;
```
```
// 这里调用的是Group2中的几个Aware，
// 而实质上这里就是前面所说的BeanPostProcessor的调用点！
// 也就是说与Group1中的Aware不同，这里是通过
BeanPostProcessor（ApplicationContextAwareProcessor）实现的。
wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean,
beanName);
```
```
// 这个是初始化方法，下文要介绍的InitializingBean调用点就是在这个方法里面
invokeInitMethods(beanName, wrappedBean, mbd);
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```
```
11
```
```
12
13
14
```

可以看到并不是所有的 Aware 接口都使用同样的方式调用。Bean××Aware 都是在代码中直接调用的，
而 ApplicationContext 相关的 Aware 都是通过 BeanPostProcessor #postProcessBeforeInitialization ()
实现的。感兴趣的可以自己看一下 ApplicationContextAwareProcessor 这个类的源码，就是判断当前
创建的 Bean 是否实现了相关的 Aware 方法，如果实现了会调用回调方法将资源传递给 Bean。

至于 Spring 为什么这么实现，应该没什么特殊的考量。也许和 Spring 的版本升级有关。基于对修改关
闭，对扩展开放的原则，Spring 对一些新的 Aware 采用了扩展的方式添加。

BeanPostProcessor 的调用时机也能在这里体现，包围住 invokeInitMethods 方法，也就说明了在初始
化阶段的前后执行。

关于 Aware 接口的执行顺序，其实只需要记住第一组在第二组执行之前就行了。每组中各个 Aware 方法
的调用顺序其实没有必要记，有需要的时候点进源码一看便知。

**简单的两个生命周期接口**

至于剩下的两个生命周期接口就很简单了，实例化和属性赋值都是 Spring 帮助我们做的，能够自己实现
的有初始化和销毁两个生命周期阶段。

**InitializingBean 接口**

InitializingBean 顾名思义，是初始化 Bean 相关的接口。

接口定义：

看方法名，是在读完 Properties 文件，之后执行的方法。afterPropertiesSet () 方法是在初始化过程中被
调用的。

InitializingBean 对应生命周期的初始化阶段，在上面源码的 invokeInitMethods (beanName,
wrappedBean, mbd); 方法中调用。

有一点需要注意，因为 Aware 方法都是执行在初始化方法之前，所以可以在初始化方法中放心大胆的使
用 Aware 接口获取的资源，这也是我们自定义扩展 Spring 的常用方式。

除了实现 InitializingBean 接口之外还能通过注解（@PostConstruct）或者 xml 配置的方式指定初始化方
法（init-method），至于这几种定义方式的调用顺序其实没有必要记。因为这几个方法对应的都是同
一个生命周期，只是实现方式不同，我们一般只采用其中一种方式。

**三种实现指定初始化方法的方法：**

```
// BeanPostProcessor的另一个调用点
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean,
beanName);
```
```
return wrappedBean;
}
```
```
15
16
17
```
```
18
19
20
```
```
public interface InitializingBean {
```
```
void afterPropertiesSet() throws Exception;
```
```
}
```
```
1
2
3
4
5
```

```
使用@PostConstruct注解，该注解作用于void方法上
在配置文件中配置init-method方法
```
```
将类实现InitializingBean接口
```
**执行：**

```
<bean id="student" class="com.demo.Student" init-method="init2">
<property name="name" value="小明"></property>
<property name="age" value="20"></property>
<property name="school" ref="school"></property>
</bean>
```
```
1
2
3
4
5
```
```
@Component("student")
public class Student implements InitializingBean{
private String name;
private int age;
...
}
```
```
1 2 3 4 5 6
```
```
@Component("student")
public class Student implements InitializingBean{
private String name;
private int age;
```
```
public String getName() {
return name;
}
public void setName(String name) {
this.name = name;
}
public int getAge() {
return age;
}
public void setAge(int age) {
this.age = age;
}
```
```
//1.使用postconstrtct注解
@PostConstruct
public void init(){
System.out.println("执行 init方法");
}
```
```
//2.在xml配置文件中配置init-method方法
public void init2(){
System.out.println("执行init2方法 ");
}
```
```
//3.实现InitializingBean接口
public void afterPropertiesSet() throws Exception {
System.out.println("执行init3方法");
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
```

通过测试我们可以得出结论，三种实现方式的执行顺序是：

**Constructor > @PostConstruct > InitializingBean > init-method**

**DisposableBean 接口**

DisposableBean 类似于 InitializingBean，对应生命周期的销毁阶段， **以
ConfigurableApplicationContext #close () 方法作为入口** ，实现是通过循环获取所有实现了
DisposableBean 接口的 Bean 然后调用其 destroy () 方法。

接口定义：

定义一个实现了 DisposableBean 接口的 Bean：

执行：

执行结果：

```
public interface DisposableBean {
void destroy() throws Exception;
}
```
```
1
2
3
```
```
public class IndexBean implements InitializingBean,DisposableBean {
public void destroy() throws Exception {
System.out.println("destroy");
}
public void afterPropertiesSet() throws Exception {
System.out.println("init-afterPropertiesSet()");
}
public void test(){
System.out.println("init-test()");
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
public class Main {
public static void main(String[] args) {
AbstractApplicationContext applicationContext=new
ClassPathXmlApplicationContext("classpath:application-usertag.xml");
System.out.println("init-success");
applicationContext.registerShutdownHook();
}
}
```
```
1 2 3 4 5 6 7
```
```
init-afterPropertiesSet()
init-test()
init-success
destroy
```
```
1
2
3
4
```

也就是说，在对象销毁的时候，会去调用 DisposableBean 的 destroy 方法。在进入到销毁过程时先去调
用一下 DisposableBean 的 destroy 方法，然后后执行 destroy-method 声明的方法（用来销毁 Bean 中的
各项数据）。

**扩展阅读: BeanPostProcessor 注册时机与执行顺序**

首先要明确一个概念，在 spring 中一切皆 bean

所有的组件都会被作为一个 bean 装配到 spring 容器中，过程如下图：

所以我们前面所讲的那些拓展点，也都会被作为一个个 bean 装配到 spring 容器中

**注册时机**

我们知道 BeanPostProcessor 也会注册为 Bean，那么 Spring 是如何保证 BeanPostProcessor 在我们的
业务 Bean 之前初始化完成呢？

请看我们熟悉的 refresh () 方法的源码，省略部分无关代码（refresh 的详细注解见 refresh ()）：

```
@Override
public void refresh() throws BeansException, IllegalStateException {
synchronized (this.startupShutdownMonitor) {
try {
// Allows post-processing of the bean factory in context
subclasses.
postProcessBeanFactory(beanFactory);
```
```
// Invoke factory processors registered as beans in the
context.
invokeBeanFactoryPostProcessors(beanFactory);
```
```
// Register bean processors that intercept bean creation.
// 注册所有BeanPostProcesser的方法
registerBeanPostProcessors(beanFactory);
```
```
// Initialize message source for this context.
initMessageSource();
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
```

可以看出，Spring 是先执行 registerBeanPostProcessors () 进行 BeanPostProcessors 的注册，然后再执
行 finishBeanFactoryInitialization 创建我们的单例非懒加载的 Bean。

**执行顺序**

BeanPostProcessor 有很多个，而且每个 BeanPostProcessor 都影响多个 Bean，其执行顺序至关重

要，必须能够控制其执行顺序才行。关于执行顺序这里需要引入两个排序相关的接口：
PriorityOrdered、Ordered

```
PriorityOrdered是一等公民，首先被执行，PriorityOrdered公民之间通过接口返回值排序
Ordered是二等公民，然后执行，Ordered公民之间通过接口返回值排序
都没有实现是三等公民，最后执行
```
在以下源码中，可以很清晰的看到 Spring 注册各种类型 BeanPostProcessor 的逻辑，根据实现不同排序
接口进行分组。优先级高的先加入，优先级低的后加入。

```
// Initialize event multicaster for this context.
initApplicationEventMulticaster();
```
```
// Initialize other special beans in specific context
subclasses.
onRefresh();
```
```
// Check for listener beans and register them.
registerListeners();
```
```
// Instantiate all remaining (non-lazy-init) singletons.
// 所有单例非懒加载Bean的创建方法
finishBeanFactoryInitialization(beanFactory);
```
```
// Last step: publish corresponding event.
finishRefresh();
}
}
```
```
18
19
20
21
```
```
22
23
24
25
26
27
28
29
30
31
32
33
34
```
```
// First, invoke the BeanDefinitionRegistryPostProcessors that implement
PriorityOrdered.
// 首先，加入实现了PriorityOrdered接口的BeanPostProcessors，顺便根据
PriorityOrdered排了序
String[] postProcessorNames =
beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class,
true, false);
for (String ppName : postProcessorNames) {
if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
currentRegistryProcessors.add(beanFactory.getBean(ppName,
BeanDefinitionRegistryPostProcessor.class));
processedBeans.add(ppName);
}
}
```
```
sortPostProcessors(currentRegistryProcessors, beanFactory);
registryProcessors.addAll(currentRegistryProcessors);
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors,
registry);
currentRegistryProcessors.clear();
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
```
```
15
```

根据排序接口返回值排序，默认升序排序，返回值越低优先级越高。

```
// Next, invoke the BeanDefinitionRegistryPostProcessors that implement
Ordered.
// 然后，加入实现了Ordered接口的BeanPostProcessors，顺便根据Ordered排了序
postProcessorNames =
beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class,
true, false);
for (String ppName : postProcessorNames) {
if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName,
Ordered.class)) {
currentRegistryProcessors.add(beanFactory.getBean(ppName,
BeanDefinitionRegistryPostProcessor.class));
processedBeans.add(ppName);
}
}
sortPostProcessors(currentRegistryProcessors, beanFactory);
registryProcessors.addAll(currentRegistryProcessors);
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors,
registry);
currentRegistryProcessors.clear();
// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no
further ones appear.
```
```
// 最后加入其他常规的BeanPostProcessors
boolean reiterate = true;
while (reiterate) {
reiterate = false;
postProcessorNames =
beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class,
true, false);
for (String ppName : postProcessorNames) {
if (!processedBeans.contains(ppName)) {
currentRegistryProcessors.add(beanFactory.getBean(ppName,
BeanDefinitionRegistryPostProcessor.class));
processedBeans.add(ppName);
reiterate = true;
}
}
sortPostProcessors(currentRegistryProcessors, beanFactory);
registryProcessors.addAll(currentRegistryProcessors);
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors,
registry);
currentRegistryProcessors.clear();
}
```
```
16
17
```
```
18
19
```
```
20
21
```
```
22
```
```
23
24
25
26
27
28
```
```
29
30
```
```
31
32
33
34
35
36
```
```
37
38
39
```
```
40
41
42
43
44
45
46
```
```
47
48
```
```
/**
* Useful constant for the highest precedence value.
* @see java.lang.Integer#MIN_VALUE
*/
int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;
/**
* Useful constant for the lowest precedence value.
* @see java.lang.Integer#MAX_VALUE
*/
int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```

PriorityOrdered、Ordered 接口作为 Spring 整个框架通用的排序接口，在 Spring 中应用广泛，也是非常
重要的接口。

**Bean 的生命周期流程图**

**总结**

Spring Bean 的生命周期分为四个阶段和多个扩展点。扩展点又可以分为影响多个 Bean 和影响单个

Bean。整理如下：


**四个阶段**

```
实例化 Instantiation
属性赋值 Populate
初始化 Initialization
销毁 Destruction
```
**多个扩展点**

```
影响多个Bean
BeanPostProcessor
InstantiationAwareBeanPostProcessor
影响单个Bean
Aware
Aware Group1
BeanNameAware
BeanClassLoaderAware
BeanFactoryAware
Aware Group2
EnvironmentAware
EmbeddedValueResolverAware
ApplicationContextAware(ResourceLoaderAware\ApplicationEventPublisherA
ware\MessageSourceAware)
生命周期
InitializingBean
DisposableBean
```
###### 哪些是重要的 bean 生命周期方法？ 你能重载它们吗？

有两个重要的 bean 生命周期方法，第一个是 setup ，它是在容器加载 bean 的时候被调用。第二个方法
是 teardown 它是在容器卸载类的时候被调用。

bean 标签有两个重要的属性（init-method 和 destroy-method）。用它们你可以自己定制初始化和注
销方法。它们也有相应的注解（@PostConstruct 和@PreDestroy）。

###### 什么是 Spring 的内部 bean？什么是 Spring inner beans？

在 Spring 框架中，当一个 bean 仅被用作另一个 bean 的属性时，它能被声明为一个内部 bean。内部
bean 可以用 setter 注入“属性”和构造方法注入“构造参数”的方式来实现，内部 bean 通常是匿名的，它们
的 Scope 一般是 prototype。

###### 在 Spring 中如何注入一个 java 集合？

Spring 提供以下几种集合的配置元素：

类型用于注入一列值，允许有相同的值。

类型用于注入一组值，不允许有相同的值。

类型用于注入一组键值对，键和值都可以为任意类型。

类型用于注入一组键值对，键和值都只能为 String 类型。


###### 什么是 bean 装配？

装配，或 bean 装配是指在 Spring 容器中把 bean 组装到一起，前提是容器需要知道 bean 的依赖关系，
如何通过依赖注入来把它们装配到一起。

###### 什么是 bean 的自动装配？

在 Spring 框架中，在配置文件中设定 bean 的依赖关系是一个很好的机制，Spring 容器能够自动装配相
互合作的 bean，这意味着容器不需要和配置，能通过 Bean 工厂自动处理 bean 之间的协作。这意味着
Spring 可以通过向 Bean Factory 中注入的方式自动搞定 bean 之间的依赖关系。自动装配可以设置在每
个 bean 上，也可以设定在特定的 bean 上。

###### 解释不同方式的自动装配，spring 自动装配 bean 有哪些方式？

在 spring 中，对象无需自己查找或创建与其关联的其他对象，由容器负责把需要相互协作的对象引用赋
予各个对象，使用 autowire 来配置自动装载模式。

在 Spring 框架 xml 配置中共有 5 种自动装配：

```
no：默认的方式是不进行自动装配的，通过手工设置ref属性来进行装配bean。
byName：通过bean的名称进行自动装配，如果一个bean的 property 与另一bean 的name 相
同，就进行自动装配。
byType：通过参数的数据类型进行自动装配。
constructor：利用构造函数进行装配，并且构造函数的参数通过byType进行装配。
autodetect：自动探测，如果有构造方法，通过 construct的方式自动装配，否则使用 byType的
方式自动装配。
```
###### 使用@Autowired 注解自动装配的过程是怎样的？

使用@Autowired 注解来自动装配指定的 bean。在使用@Autowired 注解之前需要在 Spring 配置文件进
行配置，<context:annotation-config />。

在启动 spring IoC 时，容器自动装载了一个 AutowiredAnnotationBeanPostProcessor 后置处理器，当
容器扫描到@Autowied、@Resource 或@Inject 时，就会在 IoC 容器自动查找需要的 bean，并装配给该
对象的属性。在使用@Autowired 时，首先在容器中查询对应类型的 bean：

```
如果查询结果刚好为一个，就将该bean装配给@Autowired指定的数据；
如果查询的结果不止一个，那么@Autowired会根据名称来查找；
如果上述查找的结果为空，那么会抛出异常。解决方法时，使用required=false。
```
###### 自动装配有哪些局限性？

自动装配的局限性是：

**重写** ：你仍需用和配置来定义依赖，意味着总要重写自动装配。

**基本数据类型** ：你不能自动装配简单的属性，如基本数据类型，String 字符串，和类。

**模糊特性** ：自动装配不如显式装配精确，如果有可能，建议使用显式装配。

###### 你可以在 Spring 中注入一个 null 和一个空字符串吗？

可以。


###### 问题： FactoryBean 和 BeanFactory 有什么区别？

**简要的答案：**

```
BeanFactory 是 Bean 的工厂， ApplicationContext 的父类，IOC 容器的核心，负责生产和管理
Bean 对象。
```
```
FactoryBean 是 Bean，可以通过实现 FactoryBean 接口定制实例化 Bean 的逻辑，通过代理一
个Bean对象，对方法前后做一些操作。
```
**具体的介绍：**

**（ 1 ） BeanFactory 是 ioc 容器的底层实现接口，是 ApplicationContext 顶级接**

**口**

spring 不允许我们直接操作 BeanFactory bean 工厂，所以为我们提供了 ApplicationContext 这个接
口此接口集成 BeanFactory 接口，ApplicationContext 包含 BeanFactory 的所有功能, 同时还进行更多的
扩展。

BeanFactory 接口又衍生出以下接口，其中我们经常用到的是 ApplicationContext 接口

**ApplicationContext 继承图**


**ConfiguableApplicationContext** 中添加了一些方法：

主要作用在 ioc 容器进行相应的刷新，关闭等操作！

```
... 其他省略
```
```
//刷新ioc容器上下文
void refresh() throws BeansException, IllegalStateException;
```
```
// 关闭此应用程序上下文，释放所有资源并锁定，销毁所有缓存的单例bean。
@Override
void close();
```
```
//确定此应用程序上下文是否处于活动状态，即，是否至少刷新一次且尚未关闭。
boolean isActive();
```
```
... 其他省略
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
```

**（ 2 ） FactoryBean 是 spirng 提供的工厂 bean 的一个接口**

FactoryBean 接口提供三个方法，用来创建对象，
FactoryBean 具体返回的对象是由 getObject 方法决定的。

创建一个 FactoryBean 用来生产 User 对象

```
FileSystemXmlApplicationContext 和ClassPathXmlApplicationContext 是用来读取xml
文件创建bean对象
ClassPathXmlApplicationContext ： 读取类路径下xml 创建bean
FileSystemXmlApplicationContext ：读取文件系统下xml创建bean
AnnotationConfigApplicationContext 主要是注解开发获取ioc中的bean实例
```
```
1
```
```
2
3
4
```
```
*/
public interface FactoryBean<T> {
```
```
//创建的具体bean对象的类型
@Nullable
T getObject() throws Exception;
```
```
//工厂bean 具体创建具体对象是由此getObject()方法来返回的
@Nullable
Class<?> getObjectType();
```
```
//是否单例
default boolean isSingleton() {
return true;
}
```
```
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
```
```
@Component
public class FactoryBeanTest implements FactoryBean<User> {
```
```
//创建的具体bean对象的类型
@Override
public Class<?> getObjectType() {
return User.class;
}
```
```
//是否单例
@Override
public boolean isSingleton() {
return true;
}
```
```
//工厂bean 具体创建具体对象是由此getObject()方法来返回的
@Override
public User getObject() throws Exception {
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
```

**Junit 测试**

**结果**

**简单的总结：**

###### 尼恩独家解读：循环依赖与三级缓存

循环依赖，其实就是循环引用，就是两个或者两个以上的 bean 互相引用对方，最终形成一个闭环，如
A 依赖 B，B 依赖 C，C 依赖 A。如下图所示：

```
return new User();
}
}
```
```
21
22
23
```
```
@RunWith(SpringRunner.class)
@SpringBootTest(classes = {FactoryBeanTest.class})
@WebAppConfiguration
public class SpringBootDemoApplicationTests {
@Autowired
private ApplicationContext applicationContext;
```
```
@Test
public void tesst() {
FactoryBeanTest bean1 =
applicationContext.getBean(FactoryBeanTest.class);
try {
User object = bean1.getObject();
System.out.println(object==object);
System.out.println(object);
} catch (Exception e) {
e.printStackTrace();
}
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```
```
11
12
13
14
15
16
17
18
19
```
```
true
User [id=null, name=null, age= 0 ]
```
```
1
2
```
```
BeanFactory是个bean 工厂，是一个工厂类(接口)， 它负责生产和管理bean的一个工厂
是ioc 容器最底层的接口，是个ioc容器，是spring用来管理和装配普通bean的ioc容器（这些bean
成为普通bean）。
```
```
FactoryBean是个bean，在IOC容器的基础上给Bean的实现加上了一个简单工厂模式和装饰模式，是一
个可以生产对象和装饰对象的工厂bean，由spring管理后，生产的对象是由getObject()方法决定的
（从容器中获取到的对象不是
“ FactoryBeanTest ” 对象）。
```
```
1
2
```
```
3
4
```
```
5
```

Spring 中的循环依赖，其实就是一个死循环的过程，

在初始化 A 的时候发现依赖了 B，这时就会去初始化 B，然后又发现 B 依赖 C，跑去初始化 C，初始化
C 的时候发现依赖了 A，则又会去初始化 A，依次循环永不退出，除非有终结条件。

一般来说，Spring 循环依赖的情况有两种：

```
构造器的循环依赖。
field 属性的循环依赖。
```
对于构造器的循环依赖，Spring 是无法解决的，只能抛出 BeanCurrentlyInCreationException 异

常表示循环依赖，

基于 field 属性的循环依赖，也要分两种细分的情况来对待：

```
scope 为 singleton 的循环依赖。
scope 为 prototype 的循环依赖。Spring 无法解决，直接抛出
BeanCurrentlyInCreationException 异常。
```
**三级缓存解决方案**

spring 内部有三级缓存：

```
成品 ：一级缓存 singletonObjects ，用于保存实例化、注入、初始化完成的bean实例 ：
半成品 ：二级缓存 earlySingletonObjects ，用于保存实例化完成的bean实例
原材料工厂 ： 三级缓存 singletonFactories ，用于保存bean的创建工厂，以便于后面扩展有机会
创建代理对象。
```

```
三级缓存是singletonFactories ，但是是不完整的Bean的工厂Factory,是当一个Bean在new之后
（没有属性填充、初始化），就put进去。所以，是原材料工厂
二级缓存是对三级缓存的过渡性处理，只要通过getObject()方法从三级缓存的BeanFactory中
取出Bean一次，原材料就变成变成品，就put到二级缓存 ， 所以，二级缓存里边的bean，都是半
成品
一级缓存里面是完整的Bean,是当一个Bean完全创建后(完成属性填充、彻底的完成初始化）才put
进去， 所以，是成品
```
**singleton 类型 bean 的创建过程**

Spring 处理不同 scope 时，如果是 singleton 调用的，创建过程如下图所示：


也就是说，一级缓存里面是完整的 Bean，是产品

**从源码解读，doGetBean 的查找 bean 的次序**

在 AbstractBeanFactory 的 doGetBean () 方法中，我们根据 BeanName 去获取 Singleton Bean 的时

候，会先从缓存获取。
代码如下：

```
//DefaultSingletonBeanRegistry.java
```
```
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference)
{
// 从一级缓存缓存 singletonObjects 中加载 bean
Object singletonObject = this.singletonObjects.get(beanName);
// 缓存中的 bean 为空，且当前 bean 正在创建
if (singletonObject == null &&
isSingletonCurrentlyInCreation(beanName)) {
// 加锁
```
```
1 2 3 4 5 6 7 8 9
```

**getSingleton () 的查找 bean 的次序比较清晰：**

首先，尝试从一级缓存 singletonObjects （成品仓库）中获取单例 Bean。

如果获取不到，则从二级缓存 earlySingletonObjects （半成品仓库）中获取单例 Bean。

如果仍然获取不到，则从三级缓存 singletonFactories（原材料工厂仓库）中获取单例 BeanFactory
原材料工厂。

最后，如果从三级缓存中拿到了 BeanFactory，则通过 getObject () 生产一个最原始的 bean，可以理解为
最原始的 bean 实例（原材料），没有进行属性填充，也没有完成初始化

但是，由于此处是提取 bean，所以 bean 原材料已经被使用了一次，所以，顺手把 Bean 存入二级缓存
中，并把该 Bean 的（原材料工厂）从三级缓存中删除

**三级缓存（原材料工厂仓库）里边的（原材料工厂 ）在哪里设置呢**

在 **AbstractAutowireCapableBeanFactory** 的 doCreateBean () 方法中，有这么一段代码：

```
synchronized (this.singletonObjects) {
// 从 二级缓存 earlySingletonObjects 中获取
singletonObject = this.earlySingletonObjects.get(beanName);
// earlySingletonObjects 中没有，且允许提前创建
if (singletonObject == null && allowEarlyReference) {
// 从 三级缓存 singletonFactories 中获取对应的 ObjectFactory
ObjectFactory<?> singletonFactory =
this.singletonFactories.get(beanName);
if (singletonFactory != null) {
//从单例工厂中获取bean
singletonObject = singletonFactory.getObject();
// 添加到二级缓存
this.earlySingletonObjects.put(beanName,
singletonObject);
// 从三级缓存中删除
this.singletonFactories.remove(beanName);
}
}
}
}
return singletonObject;
}
```
```
10
11
12
13
14
15
16
```
```
17
18
19
20
21
```
```
22
23
24
25
26
27
28
29
```

这段代码就是 put 三级缓存 singletonFactories 的地方，其核心逻辑是，当满足以下 3 个条件时，把

bean 加入三级缓存中：

```
单例
允许循环依赖
当前单例Bean正在创建
```
```
addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) 方法，将原始
```
的 bean 匿名工厂 （原材料工厂）加入到三级缓存，代码如下：

从这段代码我们可以看出，singletonFactories 这个三级缓存才是解决 Spring Bean 循环依赖的关键。

注意：这段代码发生在 createBeanInstance (...) 方法之后，也就是说这个 bean 其实已经被创建

出来了，但是它还没有完善（没有进行属性填充和初始化），

```
// AbstractAutowireCapableBeanFactory.java
```
```
boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
&& this.allowCircularReferences // 允许循环依赖
&& isSingletonCurrentlyInCreation(beanName)); // 当前单例 bean 是否正
在被创建
if (earlySingletonExposure) {
if (logger.isTraceEnabled()) {
logger.trace("Eagerly caching bean '" + beanName +
"' to allow for resolving potential circular references");
}
// 为了后期避免循环依赖，提前将创建的 bean 实例，加入到三级缓存
singletonFactories 中
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName,
mbd, bean));
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
12
```
```
13
```
```
// DefaultSingletonBeanRegistry.java
```
```
protected void addSingletonFactory(String beanName, ObjectFactory<?>
singletonFactory) {
Assert.notNull(singletonFactory, "Singleton factory must not be null");
synchronized (this.singletonObjects) {
if (!this.singletonObjects.containsKey(beanName)) {
this.singletonFactories.put(beanName, singletonFactory);
this.earlySingletonObjects.remove(beanName);
this.registeredSingletons.add(beanName);
}
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
```
```
protected Object doCreateBean(String beanName, RootBeanDefinition mbd,
@Nullable Object[] args) throws BeanCreationException {
BeanWrapper instanceWrapper = null;
if (mbd.isSingleton()) {
instanceWrapper =
(BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
}
```
```
1
```
```
2
3
4
```
```
5
```

设置到三级缓存之后，对于其他依赖它的对象而言已经足够了（已经有内存地址了，可以根据对象引用
定位到堆中对象），能够被认出来了。

addSingletonFactory 方法的第二参数，通过匿名对象的方式，创建了一个 ObjectFactory 工厂，这个
工厂只有一个方法，源码如下

```
if (instanceWrapper == null) {
instanceWrapper = this.createBeanInstance(beanName, mbd, args); //这
个 bean 其实已经被创建出来了
}
```
```
Object bean = instanceWrapper.getWrappedInstance();
Class<?> beanType = instanceWrapper.getWrappedClass();
if (beanType != NullBean.class) {
mbd.resolvedTargetType = beanType;
}
```
```
synchronized(mbd.postProcessingLock) {
if (!mbd.postProcessed) {
try {
this.applyMergedBeanDefinitionPostProcessors(mbd, beanType,
beanName);
} catch (Throwable var17) {
throw new
BeanCreationException(mbd.getResourceDescription(), beanName, "Post-
processing of merged bean definition failed", var17);
}
```
```
mbd.postProcessed = true;
}
}
```
```
boolean earlySingletonExposure = mbd.isSingleton() &&
this.allowCircularReferences &&
this.isSingletonCurrentlyInCreation(beanName);
if (earlySingletonExposure) {
if (this.logger.isTraceEnabled()) {
this.logger.trace("Eagerly caching bean '" + beanName + "' to
allow for resolving potential circular references");
}
```
```
this.addSingletonFactory(beanName, () -> {
return this.getEarlyBeanReference(beanName, mbd, bean);
});
}
}
```
```
6
7
8
```
```
9
10
11
12
13
14
15
16
17
18
19
20
```
```
21
22
```
```
23
24
25
26
27
28
29
```
```
30
31
32
```
```
33
34
35
36
37
38
39
```
```
@FunctionalInterface
public interface ObjectFactory<T> {
T getObject() throws BeansException;
}
```
```
1
2
3
4
```

**一级缓存在哪里设置呢？**

到这里我们发现三级缓存 singletonFactories 和二级缓存 earlySingletonObjects 中的值都有出处了，
那一级缓存在哪里设置的呢？

在类 DefaultSingletonBeanRegistry 中，可以发现这个 addSingleton (String beanName, Object

singletonObject) 方法，代码如下：

###### 高频面试题：Spring 如何解决循环依赖？

在关于 Spring 的面试中，我们经常会被问到一个问题：Spring 是如何解决循环依赖的问题的。

这个问题算是关于 Spring 的一个高频面试题，因为如果不刻意研读，相信即使读过源码，面试者也不一
定能够一下子思考出个中奥秘。

本文主要针对这个问题，从源码的角度对其实现原理进行讲解。

**循环依赖的简单例子**

比如几个 Bean 之间的互相引用：

```
// DefaultSingletonBeanRegistry.java
```
```
protected void addSingleton(String beanName, Object singletonObject) {
synchronized (this.singletonObjects) {
//添加至一级缓存，同时从二级、三级缓存中删除。
this.singletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
this.earlySingletonObjects.remove(beanName);
this.registeredSingletons.add(beanName);
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```

甚至自己“循环”依赖自己：


**原型 (Prototype) 的场景是不支持循环依赖的**

```
先说明前提：原型(Prototype)的场景是不支持循环依赖的.
```
```
单例的场景才能存在循环依赖
```
原型 (Prototype) 的场景通常会走到 AbstractBeanFactory 类中下面的判断，抛出异常。

原因很好理解，创建新的 A 时，发现要注入原型字段 B，又创建新的 B 发现要注入原型字段A...

这就套娃了, Spring 就先抛出了 BeanCurrentlyInCreationException

什么是原型 (Prototype) 的场景？

通过如下方式，可以将该类的 bean 设置为原型模式

在 Spring 中，@Service 默认都是单例的。

```
if (isPrototypeCurrentlyInCreation(beanName)) {
throw new BeanCurrentlyInCreationException(beanName);
}
```
```
1
2
3
```
```
@Service
@Scope("prototype")
public class MyReportExporter extends AbstractReportExporter{
...
}
```
```
1
2
3
4
5
```

```
所谓单例，就是Spring的IOC机制只创建该类的一个实例，
```
```
每次请求，都会用这同一个实例进行处理，因此若存在全局变量，本次请求的值肯定会影响下一
次请求时该变量的值。
```
若不想影响下次请求，就需要用到原型模式，即@Scope (“prototype”)

```
原型模式，指的是每次调用时，会重新创建该类的一个实例，比较类似于我们自己自己new的对
象实例。
```
**具体例子：循环依赖的代码片段**

我们先看看当时出问题的代码片段：

这两段代码中定义了两个 Service 类：TestService 1 和 TestService 2，

在 TestService 1 中注入了 TestService 2 的实例，同时在 TestService 2 中注入了 TestService 1 的实例，这里
构成了循环依赖。

```
只不过，这不是普通的循环依赖，因为TestService1的test1方法上加了一个@Async注解。
```
大家猜猜程序启动后运行结果会怎样？

报错了。。。原因是出现了循环依赖。

**「不科学呀，spring 不是号称能解决循环依赖问题吗，怎么还会出现？」**

```
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
@Async
public void test1() {
}
}
```
```
@Service
publicclass TestService2 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
```
```
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error
creating bean with name 'testService1': Bean with name 'testService1' has
been injected into other beans [testService2] in its raw version as part of
a circular reference, but has eventually been wrapped. This means that said
other beans do not use the final version of the bean. This is often the
result of over-eager type matching - consider using 'getBeanNamesOfType' with
the 'allowEagerInit' flag turned off, for example.
```
```
1
```

如果把上面的代码稍微调整一下：

把 TestService 1 的 test 1 方法上的@Async 注解去掉，TestService 1 和 TestService 2 都需要注入对方

的实例，同样构成了循环依赖。

但是重新启动项目，发现它能够正常运行。这又是为什么？

带着这两个问题，让我们一起开始 spring 循环依赖的探秘之旅。

**什么是循环依赖？**

循环依赖：说白是一个或多个对象实例之间存在直接或间接的依赖关系，这种依赖关系构成了构成一个
环形调用。

第一种情况：自己依赖自己的直接依赖

第二种情况：两个对象之间的直接依赖

```
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
public void test1() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```

第三种情况：多个对象之间的间接依赖

前面两种情况的直接循环依赖比较直观，非常好识别，但是第三种间接循环依赖的情况有时候因为业务
代码调用层级很深，不容易识别出来。

**循环依赖的 N 种场景**

spring 中出现循环依赖主要有以下场景：


**场景 1 ：单例的 setter 注入**

这种注入方式应该是 spring 用的最多的，代码如下：

这是一个经典的循环依赖，但是它能正常运行，得益于 spring 的内部机制，

让我们根本无法感知它有问题，因为 spring 默默帮我们解决了。

spring 内部有三级缓存：

```
成品 ：一级缓存 singletonObjects ，用于保存实例化、注入、初始化完成的bean实例 ：
半成品 ：二级缓存 earlySingletonObjects ，用于保存实例化完成的bean实例
原材料工厂 ： 三级缓存 singletonFactories ，用于保存bean创建工厂，以便于后面扩展有机会创
建代理对象。
```
一张图告诉你，spring 三级缓存之间的关系

```
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
public void test1() {
}
}
```
```
@Service
publicclass TestService2 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
```

一张图告诉你，spring 是如何解决循环依赖的：

细心的朋友可能会发现在这种场景中 **第二级缓存** 作用不大。

**那么问题来了，为什么要用第二级缓存呢？**

试想一下，如果出现以下这种情况，我们要如何处理？

```
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
@Autowired
private TestService3 testService3;
```
```
public void test1() {
}
}
```
```
@Service
publicclass TestService2 {
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
```

TestService 1 依赖于 TestService 2 和 TestService 3，

```
而TestService2依赖于TestService1，
同时TestService3也依赖于TestService1。
```
按照上图的流程可以把 TestService 1 注入到 TestService 2，并且 TestService 1 的实例是从第三级缓存中获
取的。

假设不用第二级缓存，TestService 1 注入到 TestService 3 的流程如图：

```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
@Service
publicclass TestService3 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test3() {
}
}
```
```
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
```

TestService 1 注入到 TestService 3 又需要从第三级缓存中获取实例，而第三级缓存里保存的并非真正的
实例对象，而是 ObjectFactory 对象。

```
说白了，两次从三级缓存中获取都是ObjectFactory对象，而通过它创建的实例对象每次可能都
不一样的。
```
**这样不是有问题？**

为了解决这个问题，spring 引入的第二级缓存。

前一个图其实 TestService 1 对象的实例已经被添加到第二级缓存中了，

而在 TestService 1 注入到 TestService 3 时，只用从第二级缓存中获取该对象即可。

还有个问题，第三级缓存中为什么要添加 ObjectFactory 对象，直接保存实例对象不行吗？

```
答：不行，因为假如你想对添加到三级缓存中的实例对象进行增强，直接用实例对象是行不通
的。
```
针对这种场景 spring 是怎么做的呢？

答案就在 AbstractAutowireCapableBeanFactory 类 doCreateBean 方法的这段代码中：


它定义了一个匿名内部类，通过 getEarlyBeanReference 方法获取代理对象，其实底层是通过

```
AbstractAutoProxyCreator类的getEarlyBeanReference生成代理对象。
```
**场景二多例的 setter 注入**

这种注入方法偶然会有，特别是在多线程的场景下，具体代码如下：

很多人说这种情况 spring 容器启动会报错，其实是不对的，我非常负责任的告诉你程序能够正常启动。

**为什么呢？**

其实在 AbstractApplicationContext 类的 refresh 方法中告诉了我们答案，

它会调用 finishBeanFactoryInitialization 方法，该方法的作用是为了 spring 容器启动的时候提前

初始化一些 bean。该方法的内部又调用了 preInstantiateSingletons 方法

```
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
public void test1() {
}
}
```
```
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
publicclass TestService2 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
23
```

标红的地方明显能够看出： **非抽象、单例** 并且 **非懒加载的类** 才能被提前初始 bean。

而多例即 SCOPE_PROTOTYPE 类型的类，非单例，不会被提前初始化 bean，所以程序能够正常启动。

如何让他提前初始化 bean 呢？

只需要再定义一个单例的类，在它里面注入 TestService 1

重新启动程序，执行结果：

果然出现了循环依赖。

注意：这种循环依赖问题是无法解决的，因为它没有用缓存，每次都会生成一个新对象。

**场景三：构造器注入**

这种注入方式现在其实用的已经非常少了，但是我们还是有必要了解一下，看看如下代码：

运行结果：

出现了循环依赖，为什么呢？

```
@Service
publicclass TestService3 {
```
```
@Autowired
private TestService1 testService1;
}
```
```
1 2 3 4 5 6
```
```
Requested bean is currently in creation: Is there an unresolvable circular
reference?
```
```
1
```
```
@Service
publicclass TestService1 {
```
```
public TestService1(TestService2 testService2) {
}
}
```
```
@Service
publicclass TestService2 {
```
```
public TestService2(TestService1 testService1) {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
```
```
Requested bean is currently in creation: Is there an unresolvable circular
reference?
```
```
1
```

从图中的流程看出构造器注入没能添加到三级缓存，也没有使用缓存，所以也无法解决循环依赖问题。

**场景四：单例的代理对象 setter 注入**

这种注入方式其实也比较常用，比如平时使用：@Async 注解的场景，会通过 AOP 自动生成代理对象。

我那位同事的问题也是这种情况。

从前面得知程序启动会报错，出现了循环依赖：

为什么会循环依赖呢？

```
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
@Async
public void test1() {
}
}
@Service
publicclass TestService2 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
```
```
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error
creating bean with name 'testService1': Bean with name 'testService1' has
been injected into other beans [testService2] in its raw version as part of
a circular reference, but has eventually been wrapped. This means that said
other beans do not use the final version of the bean. This is often the
result of over-eager type matching - consider using 'getBeanNamesOfType' with
the 'allowEagerInit' flag turned off, for example.
```
```
1
```

答案就在下面这张图中：

说白了，bean 初始化完成之后，后面还有一步去检查：第二级缓存和原始对象是否相等。由于它对前
面流程来说无关紧要，所以前面的流程图中省略了，但是在这里是关键点，我们重点说说：

那位同事的问题正好是走到这段代码，发现第二级缓存和原始对象不相等，所以抛出了循环依赖的异
常。

如果这时候把 TestService 1 改个名字，改成：TestService 6，其他的都不变。

再重新启动一下程序，神奇般的好了。

```
@Service
publicclass TestService6 {
```
```
@Autowired
private TestService2 testService2;
```
```
@Async
public void test1() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```

what？ 这又是为什么？

这就要从 spring 的 bean 加载顺序说起了，默认情况下，spring 是按照文件完整路径递归查找的，按路径
+文件名排序，排在前面的先加载。所以 TestService 1 比 TestService 2 先加载，而改了文件名称之后，
TestService 2 比 TestService 6 先加载。

为什么 TestService 2 比 TestService 6 先加载就没问题呢？

答案在下面这张图中：

这种情况 testService 6 中其实第二级缓存是空的，不需要跟原始对象判断，所以不会抛出循环依赖。

**场景 5 ：DependsOn 循环依赖**

还有一种有些特殊的场景，比如我们需要在实例化 Bean A 之前，先实例化 Bean B，这个时候就可以使
用@DependsOn 注解。

程序启动之后，执行结果：

```
@DependsOn(value = "testService2")
@Service
publicclass TestService1 {
```
```
@Autowired
private TestService2 testService2;
```
```
public void test1() {
}
}
@DependsOn(value = "testService1")
@Service
publicclass TestService2 {
```
```
@Autowired
private TestService1 testService1;
```
```
public void test2() {
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
```
```
1 Circular depends-on relationship between 'testService2' and 'testService1'
```

这个例子中本来如果 TestService 1 和 TestService 2 都没有加@DependsOn 注解是没问题的，反而加了这

个注解会出现循环依赖问题。

这又是为什么？

答案在 AbstractBeanFactory 类的 doGetBean 方法的这段代码中：

它会检查 dependsOn 的实例有没有循环依赖，如果有循环依赖则抛异常。

**总体策略：出现循环依赖如何解决？**

项目中如果出现循环依赖问题，说明是 spring 默认无法解决的循环依赖，要看项目的打印日志，属于哪
种循环依赖。目前包含下面几种情况：

**生成代理对象产生的循环依赖的解决方案：**

这类循环依赖问题解决方法很多，主要有：

```
1. 使用@Lazy注解，延迟加载
2. 使用@DependsOn注解，指定加载先后关系
3. 修改文件名称，改变循环依赖类的加载顺序
```
**使用@DependsOn 产生的循环依赖的解决方案：**

这类循环依赖问题要找到@DependsOn 注解循环依赖的地方，迫使它不循环依赖就可以解决问题。

**多例循环依赖的解决方案：**

这类循环依赖问题可以通过把 bean 改成单例的解决。

**构造器循环依赖的解决方案：**

这类循环依赖问题可以通过使用@Lazy 注解解决


**回答提要：**

按照上面的方式回答，起码 120 分。

```
但是答案太复杂， 如果上面的答案，记不住，就用下面的答案吧，至少也是 100 分。
```
###### 问题： Spring 是怎么解决循环依赖的？

首先，Spring 解决循环依赖有两个前提条件：

```
1. 不全是构造器方式的循环依赖
2. 必须是单例
```
基于上面的问题，我们知道 Bean 的生命周期，本质上解决循环依赖的问题就是三级缓存，通过三级缓
存提前拿到未初始化的对象。

第三级缓存：用来保存一个对象工厂，提供一个匿名内部类，用于创建二级缓存中的对象

第二级缓存：用来保存实例化完成，但是未初始化完成的对象

第一级缓存：用来保存实例化、初始化都完成的对象

假设一个简单的循环依赖场景，A、B 互相依赖。


A 对象的创建过程：

```
1. 创建对象A，实例化的时候把A对象工厂放入三级缓存
```
2 A 注入属性时，发现依赖 B，转而去实例化 B

3 同样创建对象 B，

注入属性时发现依赖 A，一次从一级到三级缓存查询 A，

从三级缓存通过对象工厂拿到 A，把 A 放入二级缓存，同时删除三级缓存中的 A，

然后，B 已经实例化并且初始化完成，把 B 放入一级缓存。

3 接着继续创建 A，顺利从一级缓存拿到实例化且初始化完成的 B 对象，A 对象创建也完成

删除二级缓存中的 A，同时把 A 放入一级缓存

4 最后，一级缓存中保存着实例化、初始化都完成的 A、B 对象


因此，由于把实例化和初始化的流程分开了，所以如果都是用构造器的话，就没法分离这个操作，

所以都是构造器的话就无法解决循环依赖的问题了。

###### 问题: 为什么要三级缓存？二级不行吗？ (重要)

不可以，主要是为了生成代理对象。

因为三级缓存中放的是生成具体对象的匿名内部类，他可以生成代理对象，也可以是普通的实例对象。

使用三级缓存主要是为了保证不管什么时候使用的都是一个对象。

假设只有二级缓存的情况，往二级缓存中放的显示一个普通的 Bean 对象，在 BeanPostProcessor 去生

成代理对象之后，覆盖掉二级缓存中的普通 Bean 对象，那么多线程环境下可能取到的对象就不一致
了。

#### Spring 注解（ 8 题目）

###### 什么是基于 Java 的 Spring 注解配置? 给一些注解的例子

基于 Java 的配置，允许你在少量的 Java 注解的帮助下，进行你的大部分 Spring 配置而非通过 XML 文件。

以@Configuration 注解为例，它用来标记类可以当做一个 bean 的定义，被 Spring IOC 容器使用。


另一个例子是@Bean 注解，它表示此方法将要返回一个对象，作为一个 bean 注册进 Spring 应用上下
文。

###### 怎样开启注解装配？

注解装配在默认情况下是不开启的，为了使用注解装配，我们必须在 Spring 配置文件中配置
<context:annotation-config/>元素。

###### @Component, @Controller, @Repository, @Service 有何区

###### 别？

@Component：这将 java 类标记为 bean。它是任何 Spring 管理组件的通用构造型。spring 的组件扫
描机制现在可以将其拾取并将其拉入应用程序环境中。

@Controller：这将一个类标记为 Spring Web MVC 控制器。标有它的 Bean 会自动导入到 IoC 容器
中。

@Service：此注解是组件注解的特化。它不会对 @Component 注解提供任何其他行为。您可以在服务
层类中使用 @Service 而不是 @Component，因为它以更好的方式指定了意图。

@Repository：这个注解是具有类似用途和功能的 @Component 注解的特化。它为 DAO 提供了额外
的好处。它将 DAO 导入 IoC 容器，并使未经检查的异常有资格转换为 Spring DataAccessException。

###### @Required 注解有什么作用

这个注解表明 bean 的属性必须在配置的时候设置，通过一个 bean 定义的显式的属性值或通过自动装
配，若@Required 注解的 bean 属性未被设置，容器将抛出 BeanInitializationException。示例：

###### @Autowired 注解有什么作用

@Autowired 默认是按照类型装配注入的，默认情况下它要求依赖对象必须存在（可以设置它 required
属性为 false）。@Autowired 注解提供了更细粒度的控制，包括在何处以及如何完成自动装配。它的
用法和@Required 一样，修饰 setter 方法、构造器、属性或者具有任意名称和/或多个参数的 PN 方法。

```
@Configuration
public class StudentConfig {
@Bean
public StudentBean myStudent() {
return new StudentBean();
}
}
```
```
1 2 3 4 5 6 7
```
```
public class Employee {
private String name;
@Required
public void setName(String name){
this.name=name;
}
public string getName(){
return name;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```

###### @Autowired 和@Resource 之间的区别

@Autowired 可用于：构造函数、成员变量、Setter 方法

@Autowired 和@Resource 之间的区别

```
@Autowired默认是按照类型装配注入的，默认情况下它要求依赖对象必须存在（可以设置它
required属性为false）。
@Resource默认是按照名称来装配注入的，只有当找不到与名称匹配的bean才会按照类型来装配
注入。
```
###### @Qualifier 注解有什么作用

当您创建多个相同类型的 bean 并希望仅使用属性装配其中一个 bean 时，您可以使用@Qualifier 注解
和 @Autowired 通过指定应该装配哪个确切的 bean 来消除歧义。

###### @RequestMapping 注解有什么用？

@RequestMapping 注解用于将特定 HTTP 请求方法映射到将处理相应请求的控制器中的特定类/方
法。此注释可应用于两个级别：

```
类级别：映射请求的 URL
方法级别：映射 URL 以及 HTTP 请求方法
```
#### Spring 数据访问（ 14 ）

###### 解释对象/关系映射集成模块

Spring 通过提供 ORM 模块，支持我们在直接 JDBC 之上使用一个对象/关系映射映射 (ORM) 工具，Spring
支持集成主流的 ORM 框架，如 Hiberate，JDO 和 iBATIS，JPA，TopLink，JDO，OJB 。Spring 的事务管
理同样支持以上所有 ORM 框架及 JDBC。

###### 在 Spring 框架中如何更有效地使用 JDBC？

使用 Spring JDBC 框架，资源管理和错误处理的代价都会被减轻。所以开发者只需写 statements 和
queries 从数据存取数据，JDBC 也可以在 Spring 框架提供的模板类的帮助下更有效地被使用，这个模板
叫 JdbcTemplate

```
public class Employee {
private String name;
@Autowired
public void setName(String name) {
this.name=name;
}
public string getName(){
return name;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```

###### 解释 JDBC 抽象和 DAO 模块

通过使用 JDBC 抽象和 DAO 模块，保证数据库代码的简洁，并能避免数据库资源错误关闭导致的问题，
它在各种不同的数据库的错误信息之上，提供了一个统一的异常访问层。它还利用 Spring 的 AOP 模块给
Spring 应用中的对象提供事务管理服务。

###### spring DAO 有什么用？

Spring DAO（数据访问对象） 使得 JDBC，Hibernate 或 JDO 这样的数据访问技术更容易以一种统一
的方式工作。这使得用户容易在持久性技术之间切换。它还允许您在编写代码时，无需考虑捕获每种技
术不同的异常。

###### spring JDBC API 中存在哪些类？

JdbcTemplate

SimpleJdbcTemplate

NamedParameterJdbcTemplate

SimpleJdbcInsert

SimpleJdbcCall

###### JdbcTemplate 是什么

JdbcTemplate 类提供了很多便利的方法解决诸如把数据库数据转变成基本数据类型或对象，执行写好
的或可调用的数据库操作语句，提供自定义的数据错误处理。

###### 使用 Spring 通过什么方式访问 Hibernate？使用 Spring 访问

###### Hibernate 的方法有哪些？

在 Spring 中有两种方式访问 Hibernate：

```
使用 Hibernate 模板和回调进行控制反转
扩展 HibernateDAOSupport 并应用 AOP 拦截器节点
```
###### 如何通过 HibernateDaoSupport 将 Spring 和 Hibernate 结合起来？

用 Spring 的 SessionFactory 调用 LocalSessionFactory。集成过程分三步：

```
配置the Hibernate SessionFactory
继承HibernateDaoSupport实现一个DAO
在AOP支持的事务中装配
```
###### Spring 支持的事务管理类型， spring 事务实现方式有哪些？

Spring 支持两种类型的事务管理：

**编程式事务管理** ：这意味你通过编程的方式管理事务，给你带来极大的灵活性，但是难维护。

**声明式事务管理** ：这意味着你可以将业务代码和事务管理分离，你只需用注解和 XML 配置来管理事务。

###### Spring 事务的实现方式和实现原理

Spring 事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring 是无法提供事务功能
的。真正的数据库层的事务提交和回滚是通过 binlog 或者 redo log 实现的。


###### 说一下 Spring 的事务传播行为

spring 事务的传播行为说的是，当多个事务同时存在的时候，spring 如何处理这些事务的行为。

```
① PROPAGATION_REQUIRED：如果当前没有事务，就创建一个新事务，如果当前存在事务，就
加入该事务，该设置是最常用的设置。
```
```
② PROPAGATION_SUPPORTS：支持当前事务，如果当前存在事务，就加入该事务，如果当前不
存在事务，就以非事务执行。
```
```
③ PROPAGATION_MANDATORY：支持当前事务，如果当前存在事务，就加入该事务，如果当前
不存在事务，就抛出异常。
```
```
④ PROPAGATION_REQUIRES_NEW：创建新事务，无论当前存不存在事务，都创建新事务。
```
```
⑤ PROPAGATION_NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前
事务挂起。
```
```
⑥ PROPAGATION_NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。
```
```
⑦ PROPAGATION_NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则
按REQUIRED属性执行。
```
###### 说一下 spring 的事务隔离？

spring 有五大隔离级别，默认值为 ISOLATION_DEFAULT（使用数据库的设置），其他四个隔离级别和
数据库的隔离级别一致：

```
1. ISOLATION_DEFAULT：用底层数据库的设置隔离级别，数据库设置的是什么我就用什么；
2. ISOLATION_READ_UNCOMMITTED：未提交读，最低隔离级别、事务未提交前，就可被其他事务
读取（会出现幻读、脏读、不可重复读）；
3. ISOLATION_READ_COMMITTED：提交读，一个事务提交后才能被其他事务读取到（会造成幻
读、不可重复读），SQL server 的默认级别；
4. ISOLATION_REPEATABLE_READ：可重复读，保证多次读取同一个数据时，其值都和事务开始时
候的内容是一致，禁止读取到别的事务未提交的数据（会造成幻读），MySQL 的默认级别；
5. ISOLATION_SERIALIZABLE：序列化，代价最高最可靠的隔离级别，该隔离级别能防止脏读、不可
重复读、幻读。
```
**脏读** ：表示一个事务能够读取另一个事务中还未提交的数据。比如，某个事务尝试插入记录 A，此时该
事务还未提交，然后另一个事务尝试读取到了记录 A。

**不可重复读** ：是指在一个事务内，多次读同一数据。

**幻读** ：指同一个事务内多次查询返回的结果集不一样。比如同一个事务 A 第一次查询时候有 n 条记
录，但是第二次同等条件下查询却有 n+1 条记录，这就好像产生了幻觉。发生幻读的原因也是另外一
个事务新增或者删除或者修改了第一个事务结果集里面的数据，同一个记录的数据内容被修改了，所有
数据行的记录就变多或者变少了。

###### Spring 框架的事务管理有哪些优点？

```
为不同的事务API 如 JTA，JDBC，Hibernate，JPA 和JDO，提供一个不变的编程模式。
为编程式事务管理提供了一套简单的API而不是一些复杂的事务API
支持声明式事务管理。
和Spring各种数据访问抽象层很好得集成。
```
###### 你更倾向用那种事务管理类型？


大多数 Spring 框架的用户选择声明式事务管理，因为它对应用代码的影响最小，因此更符合一个无侵入
的轻量级容器的思想。声明式事务管理要优于编程式事务管理，虽然比编程式事务管理（这种方式允许
你通过代码控制事务）少了一点灵活性。唯一不足地方是，最细粒度只能作用到方法级别，无法做到像
编程式事务那样可以作用到代码块级别。

#### Spring 面向切面编程 (AOP)（ 13 ）

###### 什么是 AOP

OOP (Object-Oriented Programming) 面向对象编程，允许开发者定义纵向的关系，但并适用于定义横
向的关系，导致了大量代码的重复，而不利于各个模块的重用。

AOP (Aspect-Oriented Programming)，一般称为面向切面编程，作为面向对象的一种补充，用于将那
些与业务无关，但却对多个对象产生影响的公共行为和逻辑，抽取并封装为一个可重用的模块，这个模
块被命名为“切面”（Aspect），减少系统中的重复代码，降低了模块间的耦合度，同时提高了系统的可
维护性。可用于权限认证、日志、事务处理等。

###### Spring AOP and AspectJ AOP 有什么区别？AOP 有哪些实现方

###### 式？

AOP 实现的关键在于代理模式，AOP 代理主要分为静态代理和动态代理。静态代理的代表为 AspectJ；
动态代理则以 Spring AOP 为代表。

（ 1 ）AspectJ 是静态代理的增强，所谓静态代理，就是 AOP 框架会在编译阶段生成 AOP 代理类，因此也
称为编译时增强，他会在编译阶段将 AspectJ (切面) 织入到 Java 字节码中，运行的时候就是增强之后的
AOP 对象。

（ 2 ）Spring AOP 使用的动态代理，所谓的动态代理就是说 AOP 框架不会去修改字节码，而是每次运行
时在内存中临时为方法生成一个 AOP 对象，这个 AOP 对象包含了目标对象的全部方法，并且在特定的切
点做了增强处理，并回调原对象的方法。

###### JDK 动态代理和 CGLIB 动态代理的区别

Spring AOP 中的动态代理主要有两种方式，JDK 动态代理和 CGLIB 动态代理：

```
JDK动态代理只提供接口的代理，不支持类的代理。核心InvocationHandler接口和Proxy类，
InvocationHandler 通过invoke()方法反射来调用目标类中的代码，动态地将横切逻辑和业务编织
在一起；接着，Proxy利用 InvocationHandler动态创建一个符合某一接口的的实例, 生成目标类
的代理对象。
如果代理类没有实现 InvocationHandler 接口，那么Spring AOP会选择使用CGLIB来动态代理目
标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成
指定类的一个子类对象，并覆盖其中特定方法并添加增强代码，从而实现AOP。CGLIB是通过继承
的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。
```
静态代理与动态代理区别在于生成 AOP 代理对象的时机不同，相对来说 AspectJ 的静态代理方式具有更
好的性能，但是 AspectJ 需要特定的编译器进行处理，而 Spring AOP 则无需特定的编译器处理。


```
InvocationHandler 的 invoke(Object proxy,Method method,Object[] args)：proxy是最终生成
的代理实例; method 是被代理目标实例的某个具体方法; args 是被代理目标实例某个方法的具体
入参, 在方法反射调用时使用。
```
###### 如何理解 Spring 中的代理？

将 Advice 应用于目标对象后创建的对象称为代理。在客户端对象的情况下，目标对象和代理对象是相
同的。

Advice + Target Object = Proxy

###### 解释一下 Spring AOP 里面的几个名词

（ 1 ）切面（Aspect）：切面是通知和切点的结合。通知和切点共同定义了切面的全部内容。在 Spring
AOP 中，切面可以使用通用类（基于模式的风格） 或者在普通类中以 @AspectJ 注解来实现。

（ 2 ）连接点（Join point）：指方法，在 Spring AOP 中，一个连接点总是代表一个方法的执行。应用
可能有数以千计的时机应用通知。这些时机被称为连接点。连接点是在应用执行过程中能够插入切面的
一个点。这个点可以是调用方法时、抛出异常时、甚至修改一个字段时。切面代码可以利用这些点插入
到应用的正常流程之中，并添加新的行为。

（ 3 ）通知（Advice）：在 AOP 术语中，切面的工作被称为通知。

（ 4 ）切入点（Pointcut）：切点的定义会匹配通知所要织入的一个或多个连接点。我们通常使用明确
的类和方法名称，或是利用正则表达式定义所匹配的类和方法名称来指定这些切点。

（ 5 ）引入（Introduction）：引入允许我们向现有类添加新方法或属性。

（ 6 ）目标对象（Target Object）： 被一个或者多个切面（aspect）所通知（advise）的对象。它通常
是一个代理对象。也有人把它叫做被通知（adviced） 对象。既然 Spring AOP 是通过运行时代理实现
的，这个对象永远是一个被代理（proxied） 对象。

（ 7 ）织入（Weaving）：织入是把切面应用到目标对象并创建新的代理对象的过程。在目标对象的生
命周期里有多少个点可以进行织入：

```
编译期：切面在目标类编译时被织入。AspectJ的织入编译器是以这种方式织入切面的。
类加载期：切面在目标类加载到JVM时被织入。需要特殊的类加载器，它可以在目标类被引入应用
之前增强该目标类的字节码。AspectJ5的加载时织入就支持以这种方式织入切面。
运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象
动态地创建一个代理对象。SpringAOP就是以这种方式织入切面。
```
###### Spring 在运行时通知对象

通过在代理类中包裹切面，Spring 在运行期把切面织入到 Spring 管理的 bean 中。代理封装了目标类，
并拦截被通知方法的调用，再把调用转发给真正的目标 bean。当代理拦截到方法调用时，在调用目标
bean 方法之前，会执行切面逻辑。

直到应用需要被代理的 bean 时，Spring 才创建代理对象。如果使用的是 ApplicationContext 的话，在
ApplicationContext 从 BeanFactory 中加载所有 bean 的时候，Spring 才会创建被代理的对象。因为
Spring 运行时才创建代理对象，所以我们不需要特殊的编译器来织入 SpringAOP 的切面。

###### Spring 只支持方法级别的连接点

因为 Spring 基于动态代理，所以 Spring 只支持方法连接点。Spring 缺少对字段连接点的支持，而且它不
支持构造器连接点。方法之外的连接点拦截功能，我们可以利用 Aspect 来补充。


###### 在 Spring AOP 中，关注点和横切关注的区别是什么？在 spring

###### aop 中 concern 和 cross-cutting concern 的不同之处

关注点（concern）是应用中一个模块的行为，一个关注点可能会被定义成一个我们想实现的一个功
能。

横切关注点（cross-cutting concern）是一个关注点，此关注点是整个应用都会使用的功能，并影响整
个应用，比如日志，安全和数据传输，几乎应用的每个模块都需要的功能。因此这些都属于横切关注
点。

###### Spring 通知有哪些类型？

在 AOP 术语中，切面的工作被称为通知，实际上是程序执行时要通过 SpringAOP 框架触发的代码段。

Spring 切面可以应用 5 种类型的通知：

```
1. 前置通知（Before）：在目标方法被调用之前调用通知功能；
2. 后置通知（After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；
3. 返回通知（After-returning ）：在目标方法成功执行之后调用通知；
4. 异常通知（After-throwing）：在目标方法抛出异常后调用通知；
5. 环绕通知（Around）：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定
义的行为。
```
```
同一个aspect，不同advice的执行顺序：
```
```
①没有异常情况下的执行顺序：
```

```
around before advice
before advice
target method 执行
around after advice
after advice
afterReturning
```
```
②有异常情况下的执行顺序：
```
```
around before advice
before advice
target method 执行
around after advice
after advice
afterThrowing:异常发生
java.lang.RuntimeException: 异常发生
```
###### 什么是切面 Aspect？

aspect 由 pointcount 和 advice 组成，切面是通知和切点的结合。它既包含了横切逻辑的定义, 也包括
了连接点的定义. Spring AOP 就是负责实施切面的框架, 它将切面所定义的横切逻辑编织到切面所指定
的连接点中.
AOP 的工作重心在于如何将增强编织目标对象的连接点上, 这里包含两个工作:

```
如何通过 pointcut 和 advice 定位到特定的 joinpoint 上
如何在 advice 中编写切面代码.
```
可以简单地认为, 使用 @Aspect 注解的类就是切面.

###### 解释基于 XML Schema 方式的切面实现


在这种情况下，切面由常规类以及基于 XML 的配置实现。

###### 解释基于注解的切面实现

在这种情况下 (基于@AspectJ 的实现)，涉及到的切面声明的风格与带有 java 5 标注的普通 java 类一致。

###### 有几种不同类型的自动代理？

BeanNameAutoProxyCreator

DefaultAdvisorAutoProxyCreator

Metadata autoproxying

#### 大厂面试题：读过 Spring 源码吗？说说 Spring 事务是怎么

#### 实现的？

###### 说在前面

在 40 岁老架构师尼恩的 **读者交流群** (50+) 中，最近有小伙伴拿到了一线互联网企业如阿里、美团、极
兔、有赞、希音的面试资格，Spring 事务源码的面试题，经常遇到：

```
(1）spring什么情况下进行事务回滚？
```
```
(2) spring 事务的传播行为有哪些 ？
```
与之类似的、其他小伙伴遇到过的问题还有：

```
读过Spring源码吗？说说Spring事务是怎么实现的？
```
Spring 事务的面试题，绝对是面试的核心重点，也是核心难点。

这里尼恩给大家做一下系统化、体系化梳理，使得大家可以充分展示一下大家雄厚的 “技术肌肉”， **让面
试官爱到 “不能自已、口水直流”** 。

也一并把这个题目以及参考答案，收入咱们的《尼恩 Java 面试宝典》V 60 版本，供后面的小伙伴参考，
提升大家的 3 高架构、设计、开发水平。

```
注：本文以 PDF 持续更新，最新尼恩 架构笔记、面试题 的PDF文件，请从这里获取：[码云](）
```
###### 什么是数据库事务

**数据库事务** (Database Transaction) ，是指作为单个逻辑工作单元执行的一系列操作，要么完全地执
行，要么完全地不执行。

**简单来说： 事务是逻辑上的一组操作，要么都执行，要么都不执行。**

通过事务，至少可以实现 2 点：

（ 1 ） 操作的原子性


（ 2 ）数据一致性。

我们系统的每个业务方法可能包括了多个原子性的数据库操作，比如下面的 savePerson () 方法中就

有两个原子性的数据库操作。

这些原子性的数据库操作是有依赖的，它们要么都执行，要不就都不执行。

事务就是保证这两个关键操作要么都成功，要么都要失败。最经典也经常被拿出来说例子就是 **转账** 了。

假如小明要给小红转账 1000 元，这个转账会涉及到两个关键操作就是：

```
1. 将小明的余额减少 1000 元。
2. 将小红的余额增加 1000 元。
```
万一在这两个操作之间突然出现错误比如银行系统崩溃或者网络故障，导致小明余额减少而小红的余额
没有增加，这样就不对了。

###### 数据库事务的 ACID 四大特性

数据库事务的 ACID 四大特性是事务的基础，下面简单来了解一下。

事务的执行具备四大特征：

**1 、Atomic 原子性**

事务必须是一个原子的操作序列单元，事务中包含的各项操作在一次执行过程中，要么全部执行成功，
要么全部不执行，任何一项失败，整个事务回滚，只有全部都执行成功，整个事务才算成功。

**2 、Consistency 一致性**

```
public void savePerson() {
personDao.save(person);
personDetailDao.save(personDetail);
}
```
```
1
2
3
4
```
```
public class OrdersService {
private AccountDao accountDao;
```
```
public void setOrdersDao(AccountDao accountDao) {
this.accountDao = accountDao;
}
```
```
@Transactional(propagation = Propagation.REQUIRED,
isolation = Isolation.DEFAULT, readOnly = false, timeout =
```
- 1 )
public void accountMoney () {
//小红账户多 1000
accountDao.addMoney ( 1000 ,xiaohong);
//模拟突然出现的异常，比如银行中可能为突然停电等等
//如果没有配置事务管理的话会造成，小红账户多了 1000 而小明账户没有少钱
int i = 10 / 0 ;
//小王账户少 1000
accountDao.reduceMoney ( 1000 ,xiaoming);
}
}

```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
```

```
隔离级别
脏读（Dirty
Read）
```
```
不可重复读
（NonRepeatable Read）
```
```
幻读（Phantom
Read）
```
```
读未提交(Read
uncommitted）
可能 可能 可能
```
```
读已提交（Read
committed）
不可能 可能 可能
```
```
可重复读
（Repeatable
read）
```
```
不可 能 不可 能 可能
```
```
可串行化
不可能 不可能 不可能
```
事务的执行不能破坏数据库数据的完整性和一致性，事务在执行之前和之后，数据库都必须处于一致性
状态。

**3 、Isolation 隔离性**

在并发环境中，并发的事务是相互隔离的，一个事务的执行不能被其他事务干扰。

即不同的事务并发操纵相同的数据时，每个事务都有各自完整的数据空间，即一个事务内部的操作及使
用的数据对其他并发事务是隔离的，并发执行的各个事务之间不能相互干扰。

**4 、Durability 持久性**

持久性（durability）：持久性也称永久性（permanence），指一个事务一旦提交，它对数据库中对应
数据的状态变更就应该是永久性的。

即使发生系统崩溃或机器宕机，只要数据库能够重新启动，那么一定能够将其恢复到事务成功结束时的
状态。

```
比方说：一个人买东西的时候需要记录在账本上，即使老板忘记了那也有据可查。
```
以上的事务特性是由 Innodb 引擎提供，为实现这些特性，使用到了数据库锁、WAL、MVCC 等技术。

###### 事务的并发执行

并发场景、高并发场景下，事务都是并发执行的。

多个事务，如果都是一个一个串行，想想数据库的性能会有多低下。

但是，事务的并发执行，可能会带来很多问题, 比如脏读、不可重复读、幻读等问题。

如果不对事务进行并发控制，我们看看数据库并发操作是会有那些异常情形

```
（ 1 ）一类丢失更新：两个事物读同一数据，一个修改字段 1 ，一个修改字段 2 ，后提交的恢复了先
提交修改的字段。
（ 2 ）二类丢失更新：两个事物读同一数据，都修改同一字段，后提交的覆盖了先提交的修改。
（ 3 ）脏读：读到了未提交的值，万一该事物回滚，则产生脏读。
（ 4 ）不可重复读：两个查询之间，被另外一个事务修改（update）了数据的内容，产生内容的不
一致。
（ 5 ）幻读：两个查询之间，被另外一个事务插入或删除了（insert、delete）记录，产生结果集
的不一致。
```
在数据库操作中，为了有效保证并发读取数据的正确性，提出的事务 **隔离级别** 。我们的数据库锁，也是
为了构建这些隔离级别存在的。


```
隔离级别
脏读（Dirty
Read）
```
```
不可重复读
（NonRepeatable Read）
```
```
幻读（Phantom
Read）
```
```
（Serializable ）
```
```
不可能 不可能 不可能
```
###### （ 1 ）读未提交

**允许脏读。** 在 **读未提交** 隔离级别下，允许 **脏读** 的情况发生。

脏读指的是读到了其他事务未提交的数据，

未提交意味着这些数据可能会回滚，也就是可能最终不会存到数据库中，也就是不存在的数据。

读到了并一定最终存在的数据，这就是脏读。

脏读最大的问题就是可能会读到不存在的数据。

比如在上图中，事务 B 的更新数据被事务 A 读取，但是事务 B 回滚了，更新数据全部还原，也就是说事务
A 刚刚读到的数据并没有存在于数据库中。

###### （ 2 ）读已提交

在 **读已提交** 隔离级别下， **禁止了脏读** ，但是允许 **不可重复读** 的情况发生

**不可重复读** 指的是在一个事务内，最开始读到的数据和事务结束前的任意时刻读到的同一批数据出现不
一致的情况。

```
如果一个事务正在处理某一数据，并对其进行了更新，
但同时尚未完成事务，或者说事务没有提交，
与此同时，允许另一个事务也能够访问该数据。
例如A将变量n从 0 累加到 10 才提交事务，此时B可能读到n变量从 0 到 10 之间的所有中间值。
```
```
1
2
3
4
```
```
只允许读到已经提交的数据。
即事务A在将n从 0 累加到 10 的过程中，B无法看到n的中间值，之中只能看到 10 。
```
```
1
2
```
```
事务A在将n从 0 累加到 10 的过程中，B无法看到n的中间值，之中只能看到 10 。
同时，
有事务C进行从 10 到 20 的累加，此时B在同一个事务内再次读时，读到的是 20 。
```
```
1
2
3
```

**事务 A 多次读取同一数据，但事务 B 在事务 A 多次读取的过程中，对数据作了更新并提交，导致事务 A
多次读取同一数据时，结果不一致。**

不可重复读一词，有点反人类，不好记忆。是从 **Nonrepeatable read** 翻译过来的，感觉英文的，好
记忆一点。

###### （ 3 ）可重复读

在 **可重复读** 隔离级别下，禁止了： **脏读、不可重复读** 。

但是，允许 **幻读** 。

在可重复读中，该 sql 第一次读取到数据后，就将这些数据加锁（悲观锁），其它事务无法修改这些数
据，就可以实现可重复读了。

但这种方法却无法锁住 insert 的数据，所以当事务 A 先前读取了数据，或者修改了全部数据，事务 B 还是
可以 insert 数据提交，

这时事务 A 就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。

###### （ 4 ）串行化

**事务的隔离级别总结**

隔离级别有串行化读、可重复读、读已提交、读未提交四种级别。

```
串行化读级别下的事务并发度太低，原因是锁的粒度太大，基本没有场景可以被使用。
读未提交级别允许脏读，可以使用的场景并不多。
读已提交和可重复读是大多数数据库也是大多数项目会采用的数据库隔离级别。
```
```
1 保证在事务处理过程中，多次读取同一个数据时，其值都和事务开始时刻时是一致的。
```
```
1 最严格的事务，要求所有事务被串行执行，不能并发执行。
```

**读已提交隔离级别** 下由于读到其他事务已提交的数据，所以不会出现脏读，在普通读取时，使用到
MVCC 的快照读机制解决幻读问题；若在查询语句后增加 for update，标识当前查询是当前读，当前读
并不能解决幻读问题；允许不可重复读。

**读已提交隔离级别** 并发度比较高，互联网行业需要数据库较高的事务并发度，一般会选择此种隔离级
别。也是 Oracle 数据库的默认隔离级别。在使用锁方面，查询时对结果中数据的索引加共享锁，数据读
取结束就会释放，更新时对操作的数据索引加排他锁，需要等到事务结束才会释放。

**可重复读** 隔离级别是 Mysql 的默认隔离级别， **事务并发度仅次于读已提交** ，普通读取的幻读问题与读已
提交的解决方式一致，不过当前读可以使用 Gap Lock + Record Lock 解决。不可重复读的问题解决办
法就是查询时对结果中数据的索引加锁共享锁，再事务结束时才会释放锁。解决不可重读的代价会牺牲
掉并发度。

###### JDBC 的转账事务案例

为了深入 Spring 事务源码，先看一下在 JDBC 中对事务的操作处理.

还是以经典的转账事务为例。

```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
```
```
public class TransactionExample {
```
```
public static void main(String[] args) {
Connection conn = null;
PreparedStatement pstmt1 = null, pstmt2 = null;
try {
```
```
// 加载数据库驱动并建立连接
Class.forName("com.mysql.jdbc.Driver");
conn =
DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root",
"password");
```
```
// 关闭自动提交，开启事务
conn.setAutoCommit(false);
```
```
// 创建SQL语句
String sql1 = "UPDATE account SET balance = balance -? WHERE
id = ?";
String sql2 = "UPDATE account SET balance = balance +? WHERE
id = ?";
```
```
// 创建PreparedStatement对象
pstmt1 = conn.prepareStatement(sql1);
pstmt2 = conn.prepareStatement(sql2);
```
```
// 设置参数
pstmt1.setDouble( 1 , 1000 );
pstmt1.setInt( 2 , 1 );
pstmt2.setDouble( 1 , 1000 );
pstmt2.setInt( 2 , 2 );
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
```
```
17
18
19
20
21
22
```
```
23
```
```
24
25
26
27
28
29
30
31
32
33
```

java 代码中要使用事务，就是三个步骤：

mysql 中事务默认是自动提交, 一条 sql 语句就是一个事务.

所以，上面的代码，第一步，首先关闭自动提交，开启手动事务方式

```
// 执行更新操作
int count1 = pstmt1.executeUpdate();
int count2 = pstmt2.executeUpdate();
```
```
if (count1 > 0 && count2 > 0 ) {
System.out.println("转账成功");
// 提交事务
conn.commit();
} else {
System.out.println("转账失败");
// 回滚事务
conn.rollback();
}
} catch (ClassNotFoundException e) {
e.printStackTrace();
} catch (SQLException e) {
try {
if (conn != null) {
// 回滚事务
conn.rollback();
}
} catch (SQLException e1) {
e1.printStackTrace();
}
e.printStackTrace();
} finally {
try {
if (pstmt1 != null) {
pstmt1.close();
}
if (pstmt2 != null) {
pstmt2.close();
}
if (conn != null) {
conn.close();
}
} catch (SQLException e) {
e.printStackTrace();
}
}
}
}
```
```
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
```
```
start transaction;-- 开启一个事务
commit;-- 事务提交
rollback;-- 事务回滚
```
```
1
2
3
4
```

然后，第 2 步，如果顺利的话，就提交事务

如果发生异常的话，第 3 步，就回滚事务

这里总结一下，上面代码中的 JDBC 的转账事务案例几个重点步骤：

```
获取连接：
```
```
关闭自动提交，开启事务：conn.setAutoCommit(false)
提交事务：conn.commit()
回滚事务：conn.rollback()
```
#### Spring 转账事务案例

###### Spring 两种事务管理方式

Spring 支持两种事务管理方式：编程式事务和声明式事务。

```
// 关闭自动提交，开启事务
conn.setAutoCommit(false);
```
```
1
2
3
4
```
```
// 提交事务
conn.commit();
```
```
1
2
3
```
```
// 回滚事务
conn.rollback();
```
```
1
2
3
```
```
Connection conn =
DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root",
"password")
```
```
1
```
```
2
```

编程式事务是指在代码中显式地开启、提交或回滚事务。这种方式需要在代码中编写事务管理的相关逻
辑，比较繁琐，但是灵活性较高，可以根据具体的业务需要进行定制。

声明式事务是通过配置来实现的，不需要在代码中显式地管理事务。这种方式需要在配置文件中声明事
务的属性，比如事务的传播行为、隔离级别等。声明式事务的好处是可以将事务管理的逻辑与业务逻辑
分离，使得代码更加简洁、清晰，同时也方便了事务管理的统一配置和维护。

在 Spring 中，声明式事务主要是通过 AOP 实现的。Spring 提供了两种声明式事务的方式：基于 XML
配置和基于注解配置。基于 XML 配置的方式需要在 Spring 配置文件中声明事务管理器和事务通知等相
关信息，而基于注解配置的方式则可以在代码中通过注解来声明事务的属性，比如 @Transactional。

**基于 XML 配置文件进行配置**

```
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:p="http://www.springframework.org/schema/p"
xmlns:context="http://www.springframework.org/schema/context"
xmlns:aop="http://www.springframework.org/schema/aop"
xmlns:tx="http://www.springframework.org/schema/tx"
xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-4.3.xsd
http://www.springframework.org/schema/aop
http://www.springframework.org/schema/aop/spring-aop-4.3.xsd
http://www.springframework.org/schema/tx
http://www.springframework.org/schema/tx/spring-tx-4.1.xsd">
<!-- 开启扫描 -->
<context:component-scan base-package="com.dpb.*"></context:component-
scan>
```
```
<!-- 配置数据源 -->
<bean
class="org.springframework.jdbc.datasource.DriverManagerDataSource"
id="dataSource">
<property name="url"
value="jdbc:oracle:thin:@localhost:1521:orcl"/>
<property name="driverClassName"
value="oracle.jdbc.driver.OracleDriver"/>
<property name="username" value="pms"/>
<property name="password" value="pms"/>
</bean>
```
```
<!-- 配置JdbcTemplate -->
<bean class="org.springframework.jdbc.core.JdbcTemplate" >
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```
```
11
12
```
```
13
14
15
```
```
16
```
```
17
```
```
18
19
20
21
22
23
```

**基于注解的声明式配置**

一般来说，更加推荐声明式事务比编程式事务，因为它可以使代码更加简洁、清晰，同时也方便了事务
管理的统一配置和维护。

所以，这里使用声明式事务进行演示，并且是使用基于注解配置的声明式事务。

首先必须要添加 @EnableTransactionManagement 注解，保证事务注解生效

其次，在方法上添加 @Transactional 代表注解生效

下面的案例，用到基于注解的声明式配置，具体的注解是 @Transactional。

```
<constructor-arg name="dataSource" ref="dataSource"/>
</bean>
```
```
<!--
Spring中，使用XML配置事务三大步骤：
```
1. 创建事务管理器
2. 配置事务方法
3. 配置 AOP
-->
<bean
class="org. springframework. jdbc. datasource. DataSourceTransactionManager"
id="transactionManager">
<property name="dataSource" ref="dataSource"/>
</bean>
<tx:advice id="advice" transaction-manager="transactionManager">
<tx:attributes>
<tx:method name="fun*" propagation="REQUIRED"/>
</tx:attributes>
</tx:advice>
<!-- aop配置 -->
<aop:config>
<aop: pointcut expression="execution (* *.. service.*.*(..))"
id="tx"/>
<aop:advisor advice-ref="advice" pointcut-ref="tx"/>
</aop:config>
</beans>

```
24
25
26
27
28
29
30
31
32
33
```
```
34
35
36
37
38
39
40
41
42
43
```
```
44
45
46
47
```
```
@EnableTransactionManagement
public class AnnotationMain {
public static void main(String[] args) {
}
}
```
```
1 2 3 4 5 6
```
```
@Transactional
public int insertUser(User user) {
userDao.insertUser();
userDao.insertLog();
return 1 ;
}
```
```
1 2 3 4 5 6 7
```

###### 转账案例的三层架构

首先，看看日常项目中，转账案例的三层架构。

我们在日常生产项目中，项目由 Controller、Serivce、Dao 三层进行构建。

对于服务层的 transfer 方法，实际对于 DAO 调用来说，分别调用了两次 DAO 的 upate 方法，更新了两
个账号的 amount 金额，也就是说，对数据库的操作为两次。

所以，我们要保证 transfer 方法是符合事务定义的，具备事务的四大特性：ACID。

###### 一个简单的 Spring 转账事务案例

好的，下面是一个简单的 Spring 转账事务案例，一共 5 步骤：

```
1. 定义账户实体类
2. 定义转账服务接口
3. 实现转账服务接口
4. 定义账户数据访问对象（DAO）
5. 配置事务管理器
```
接下来，咱们 step by step，一点点揭秘一下这个 5 步骤

```
1. 定义账户实体类
```
```
2. 定义转账服务接口
```
```
public class Account {
```
```
private Long id;
private String accountNumber;
private double balance;
```
```
// 省略 getter 和 setter 方法
}
```
```
1 2 3 4 5 6 7 8 9
```

```
3. 实现转账服务接口
```
```
4. 实现账户数据访问对象（DAO）
```
```
public interface TransferService {
void transfer(String fromAccount, String toAccount, double amount);
}
```
1
2
3

```
@Service
public class TransferServiceImpl implements TransferService {
```
```
@Autowired
private AccountDao accountDao;
```
```
@Transactional
public void transfer(String fromAccount, String toAccount, double
amount) {
Account from = accountDao.findByAccountNumber(fromAccount);
Account to = accountDao.findByAccountNumber(toAccount);
from.setBalance(from.getBalance() - amount);
to.setBalance(to.getBalance() + amount);
accountDao.update(from);
accountDao.update(to);
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
```
```
@Repository
public class AccountDaoImpl implements AccountDao {
```
```
@Autowired
private JdbcTemplate jdbcTemplate;
```
```
public Account findByAccountNumber(String accountNumber) {
String sql = "SELECT * FROM account WHERE account_number = ?";
return jdbcTemplate.queryForObject(sql, new Object[]
{accountNumber}, new AccountRowMapper());
}
```
```
public void update(Account account) {
String sql = "UPDATE account SET balance =? WHERE id = ?";
jdbcTemplate.update(sql, account.getBalance(), account.getId());
}
}
```
```
class AccountRowMapper implements RowMapper<Account> {
public Account mapRow(ResultSet rs, int rowNum) throws SQLException {
Account account = new Account();
account.setId(rs.getLong("id"));
account.setAccountNumber(rs.getString("account_number"));
account.setBalance(rs.getDouble("balance"));
return account;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
```

```
5. 配置事务管理器
```
在上面的代码中，我们使用了 Spring 的声明式事务管理。

在第 3 步，也就是服务层的实现方法上，添加 @Transactional 注解，告诉 Spring 这是一个需要进行事
务管理的方法。

这里的事务方法，是 transfer () 方法。

当 transfer () 方法被调用时，如果发生异常，事务管理器会自动回滚事务，保证数据库的一致性。

#### 详解：事务注解 @Transactional

@Transactional 是 Spring Framework 中常用的注解之一，它可以被用于管理事务。通过使用这个注
解，我们可以方便地管理事务，保证数据的一致性和完整性。

在 Spring 应用中，当我们需要对数据库进行操作时，通常需要使用事务来保证数据的一致性和完整
性。

@Transactional 注解可以被用于类或方法上，用于指定事务的管理方式。当它被用于类上时，它表示
该类中所有的方法都将被包含在同一个事务中。当它被用于方法上时，它表示该方法将被包含在一个新
的事务中。

@Transactional 注解有多个属性，其中最常用的是 propagation 和 isolation。

```
propagation 属性用于指定事务的传播行为，它决定了当前方法执行时，如何处理已经存在的事务
isolation 属性用于指定事务的隔离级别，它决定了当前事务与其他事务之间的隔离程度。
```
除了 propagation 和 isolation 属性外，@Transactional 还支持其他属性，如 readOnly、timeout、
rollbackFor、noRollbackFor 等，这些属性可以用于进一步细化事务的行为。

总之，@Transactional 注解是 Spring 应用中常用的事务管理注解。

```
@Configuration
@EnableTransactionManagement
public class AppConfig {
```
```
@Bean
public DataSource dataSource() {
// 配置数据源
}
```
```
@Bean
public JdbcTemplate jdbcTemplate() {
return new JdbcTemplate(dataSource());
}
```
```
@Bean
public PlatformTransactionManager transactionManager() {
return new DataSourceTransactionManager(dataSource());
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
```

###### @Tranasctional 注解的使用注意事项

@Tranasctional 注解是 Spring 框架提供的声明式注解事务解决方案，在使用@Transactional 注解时需
要注意以下问题:

```
1. @Transactional 注解只能用在public 方法上，如果用在protected或者private的方法上，不会报
错，但是该注解不会生效。
2. 默认情况下，@Transactional注解只能回滚 非检查型异常，具体为RuntimeException及其子类和
Error子类
```
```
非检查型异常指（Unchecked Exception）的是程序在编译时不会提示需要处理该异常，而
是在运行时才会出现异常。在 Java 中，非检查型异常指的是继承自 RuntimeException 类
的异常，例如 NullPointerException、ArrayIndexOutOfBoundsException 等。这些异常通
常是由程序员的代码错误引起的，因此应该尽可能避免它们的发生，但是在代码中并不需要
显式地处理它们。
检查型异常（Checked Exception）是指在 Java 中，编译器会强制要求对可能会抛出这些异
常的代码进行异常处理，否则代码将无法通过编译。这些异常包括 IOException、
SQLException 等等，它们通常表示一些外部因素导致的异常情况，比如文件读写错误、数
据库连接失败等等。
在编写代码时应该尽量避免抛出非检查型异常，因为这些异常的发生通常意味着程序存在严
重的逻辑问题。
```
默认情况下，@Transactional 注解只能回滚非检查型异常，为啥呢?

可以从 Spring 源码的 DefaultTransactionAttribute 类里找到判断方法 rollbackOn。

```
3. 如果需要对检查型异常（Checked Exception）进行回滚，可以使用rollbackFor 属性来定义回滚
的异常类型，使用 propagation 属性定义事务的传播行为。
```
下面是一个例子：

上面的例子中: 指定了回滚 Exception 类的异常为 Exception 类型或者其子类型检查型异常（Checked
Exception），另外，配置类事务的传播行为支持当前事务，当前如果没有事务，那么会创建一个事
务。

```
4. @Transactional注解不能回滚被try{}catch() 捕获的异常。
5. @Transactional注解只能对在被Spring 容器扫描到的类下的方法生效。
```
其实 Spring 事务的创建也是有一定的规则，对于一个方法里已经存在的事务，Spring 也提供了解决方案
去进一步处理存在事务，通过设置@Tranasctional 的 propagation 属性定义 Spring 事务的传播规则。

###### Spring 事务的传播规则

Spring 事务的传播规则是指在多个事务方法相互调用的情况下，事务应该如何进行传播和管理。

Spring 事务的传播行为一共有 7 种，定义在 spring-tx 模块的 Propagation 枚举类里，对应的常量值定义
在 TransactionDefinition 接口里, 值为 int 类型的 0-6。

```
@Override
public boolean rollbackOn(Throwable ex) {
return (ex instanceof RuntimeException || ex instanceof Error);
}
```
```
1
2
3
4
```
```
@Transactional(rollbackFor = Exception.class,propagation =
Propagation.REQUIRED)
```
```
1
```

```
PROPAGATION_REQUIRED
```
```
支持当前事务，如果当前没有事务，则创建一个事务，这
是最常见的选择。
```
```
PROPAGATION_SUPPORTS 支持当前事务，如果当前没有事务，就以非事务来执行
```
```
PROPAGATION_MANDATORY 支持当前事务，如果没有当前事务，就抛出异常。
```
```
PROPAGATION_REQUIRES_NEW 新建事务，如果当前存在事务，就把当前事务挂起。
```
```
PROPAGATION_NOT_SUPPORTED
```
```
以非事务执行操作，如果当前存在事务，则当前事务挂
起。
```
```
PROPAGATION_NEVER 以非事务方式执行，如果当前存在事务，则抛出异常。
```
```
PROPAGATION_NESTED
如果当前存在事务，则在嵌套事务内执行。如果当前没有
事务，则进行与PROPAGATION_REQUIRED 类似的操作。
```
稍后一点，结合源码介绍。

#### Spring 事务总体架构

40 岁老架构尼恩的建议： 要学源码，先梳理架构。

pring 事务总体架构，尼恩给大家梳理了两个要点：

```
二大核心模式
三大核心流程
```
###### 二大核心模式

Spring 事务架构，用到了二大核心模式

**模式一： 策略模式**


设计了统一的事务管理器超级接口， **PlatformTransactionManager** 是 Spring 事务管理器设计的基
类。

**PlatformTransactionManager** 基类，实现了事务执行设计到的三个核心方法：

不同的数据管理组件 JDBC、Hibernate 等，为 Spring 都提供了对应的事务管理器实现类。

既然是策略模式, 如何进行动态的决策呢？

在动态代理模式的切面的支撑类 TransactionAspectSupport 中，会有一个
determineTransactionManager 方法，动态的从 IOC 容器，加载注册了的
**PlatformTransactionManager** 实现类。


尼恩的脚手架，用到的 orm 框架是 jpa。在 spring 的 bean factory 里边注册了的
**PlatformTransactionManager** 实现类是 JpaTransactionManager 。

所以，这里的 JpaTransactionManager 作为事务管理器。

**模式二： 动态代理模式**

动态代理有 JDK 的动态代理、 Aspectj 的动态代理，SpringAOP 动态代理，很多种。

SpringAOP 和 Aspectj 的动态代理有和联系呢？ 本质上 Aspectj 是一种静态代理，而 SpringAOP 是动态
代理。但 Aspectj 的一套定义 AOP 的 API 非常好，直观易用。

但是，AOP 联盟的关键词有 Advice (顶级的通知类/拦截器)、MethodInvocation (方法连接点)、
MethodInterceptor (方法拦截器)

SpringAOP 与 AOP 联盟关系

SpringAOP 在 AOP 联盟基础上又增加了几个类，丰富了 AOP 定义及使用概念，SpringAOP 的核心名称和
概念包括：

```
Advisor ：包含通知(拦截器)，Spring内部使用的AOP顶级接口，还需要包含一个aop适用判断的
过滤器，考虑到通用性， 过滤规则由其子接口定义，例如IntroductionAdvisor和
PointcutAdvisor ，过滤器用于判断bean是否需要被代理
Pointcut : 切点，属于过滤器的一种实现，匹配过滤哪些类哪些方法需要被切面处理，包含一个
ClassFilter和一个MethodMatcher，使用PointcutAdvisor定义时需要
ClassFilter ：限制切入点或引入点与给定目标类集的匹配的筛选器，属于过滤器的一种实现。过
滤筛选合适的类，有些类不需要被处理
MethodMatcher ：方法匹配器，定义方法匹配规则，属于过滤器的一种实现，哪些方法需要使
用AOP
```
SpringAOP 实现的大致思路：

1. 配置获取 Advisor (顾问)：拦截器+AOP 匹配过滤器，生成 Advisor
2. 生成代理：根据 Advisor 生成代理对象，会生成 JdkDynamicAopProxy 或 CglibAopProxy
3. 执行代理：代理类执行代理时，从 Advisor 取出拦截器，生成 MethodInvocation (连接点) 并执行代理
过程。

如果对在 Spring AOP 中，Advisor 和 Interceptor 是两个重要的概念。


Advisor 是用于定义切面的对象，它包含了切点和通知两个部分。

```
切点指定了哪些方法需要被拦截 ， 使用@Pointcut 注解
通知则指定了拦截后需要执行的逻辑。
```
Interceptor 是用于拦截后，执行拦截逻辑的对象，它实现了 Spring 的 MethodInterceptor 接口，负
责拦截方法的执行，并在方法执行前后执行一些操作。

下面是一个使用 Advisor 和 Interceptor 的示例：

在上面的代码中，我们使用了 Advisor 和 Interceptor 来实现对 com. example. service 包中所有方法的
拦截。

首先，我们定义了一个切点 pointcut ()，它指定了需要拦截的方法。

然后，我们定义了一个 around () 方法，并使用 @Around 注解将它与切点关联起来。在 around () 方法
中，我们实现了拦截逻辑，即在方法执行前后分别输出 "before method" 和 "after method"。最后，
我们通过 pjp.proceed () 方法执行了被拦截的方法，并返回了它的执行结果。

需要注意的是，为了让 Spring 能够识别 MyInterceptor 类并将它作为 Advisor 使用，我们需要在它上
面加上 @Aspect 和 @Component 注解。

注意 Advisor 是动态生成的，但是 Spring 也会实现了一个类似于 PointcutAdvisor 的接口，可以根

据配置的切入点和目标方法生成代理对象，调用咱们定义的 MyInterceptor 实现拦截和 AOP 增强功能。

```
@Aspect
@Component
public class MyInterceptor {
```
```
//切点
@Pointcut("execution(* com.example.service.*.*(..))")
public void pointcut() {}
```
```
@Around("pointcut()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
// 在方法执行前执行的逻辑
System.out.println("before method");
```
```
// 执行被拦截的方法
Object result = pjp.proceed();
```
```
// 在方法执行后执行的逻辑
System.out.println("after method");
```
```
return result;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
```

了解了 SpringAOP 基本原理之后，我们来看看 SpringAOP 实现事务的动态代理模式的架构。

SpringAOP 实现事务的动态代理模式的架构图如下：

动态代理模式的两个核心角色是：

```
Advisor： BeanFactoryTransactionAttributeSourceAdvisor
Interceptor： TransactionInterceptor
```
Spring 事务进行了 Advisor 的扩展， BeanFactoryTransactionAttributeSourceAdvisor 是 Spring

AOP 框架中一个用于事务管理的类。

该 Advisor 它实现了 PointcutAdvisor 接口，可以根据配置的切入点和事务属性来为目标方法生成代

理对象，从而实现事务管理的功能。

在 Spring 中，事务管理是通过 TransactionInterceptor 实现的。该 Advisor 的作用就是为
Interceptor 提供事务属性信息。

该 Advisor 通过 TransactionAttributeSource 接口来获取事务属性，而
TransactionAttributeSource 可以从配置文件、注解等多种途径获取事务属性信息。该 Advisor 是

从 Spring 容器中获取 TransactionAttributeSource 的实现类，并将其设置给

```
TransactionInterceptor，从而实现事务管理的功能。
```
###### 三大核心流程


```
AOP 动态代理装配流程
AOP 动态代理执行流程
事务执行流程
```
**核心流程一：AOP 动态代理装配流程**

AOP 动态代理装配流程，大致如下：

咱们关注的是 Advisor 和 Interceptor 的装配。

**核心流程二：AOP 动态代理执行流程**

AOP 动态代理执行流程，大致如下：

执行被@Transactional 注解的方法时，首先会被 AOP 切面代理拦截，先执行事务的动态代理，再执行
业务方法。

在事务的动态代理动态代理对象中，进行事务的开启、提交、或者回滚。

**核心流程三：事务执行流程**


AOP 事务执行流程，大致如下：

这个使用 Spring 事务管理器实现。

所以，学习 Spring 源码，咱们先从 Spring 事务管理器源码开始。

#### Spring 事务管理器架构和源码学习

Spring 事务管理的实现有许多细节，如果对整个模块架构有个大体了解，会非常有利于我们理解事务，

###### Spring 事务管理模块架构：

Spring 事务管理模块架构，如下：

Spring 并不直接管理事务，而是提供了多种事务管理器。


通过事务管理器，Spring 将事务管理的职责委托给 ORM 框架，比如 Hibernate 或者 JTA 等，由 ORM 框架
的持久化机制所提供的相关平台框架的事务来实现。

###### 事务管理的超级接口

Spring 事务管理器的超级接口是 PlatformTransactionManager，

PlatformTransactionManager 超级接口的内容如下：

通过这个超级接口，Spring 为各个平台如 JDBC、Hibernate 等都提供了对应的事务管理器。

从这里可知：PlatformTransactionManager 具体的实现就是各个平台自己的事情了。

所以说，具体的事务管理机制对 Spring 来说是透明的，Spring 并不关心那些，那些是对应各个平台需要
关心的。

所以，Spring 事务管理模块架构的一个优点就是：为不同的事务 API 提供一致的编程模型，如 JTA、
JDBC、Hibernate、JPA。

下面分别介绍各个平台框架实现事务管理的机制。

###### JDBC 事务管理器

如果应用程序中直接使用 JDBC 来进行 ORM 持久化，DataSourceTransactionManager 会为你处理事
务，提供事务管理的的具体实现。

```
DataSourceTransactionManager 是 Spring Framework 中用于管理数据库事务的类，它是基于
```
JDBC 的事务管理器。

为了使用 DataSourceTransactionManager，如果使用 XML 定义 Bean，需要使用如下的 XML 将其装配到
应用程序的上下文定义中：

在 Spring Boot 中，如果你使用了 Spring Data JPA 或者 MyBatis 等 ORM 框架，那么默认会使用
DataSourceTransactionManager 进行事务管理。

SpringBoot 在使用事物 Transactional 的时候，要在 main 方法上加上
@EnableTransactionManagement 注解开发事物声明，在使用的 service 层的公共方法加上
@Transactional (spring) 注解。

```
Public interface PlatformTransactionManager()...{
// 由TransactionDefinition得到TransactionStatus对象
TransactionStatus getTransaction(TransactionDefinition definition) throws
TransactionException;
// 提交
Void commit(TransactionStatus status) throws TransactionException;
// 回滚
Void rollback(TransactionStatus status) throws TransactionException;
}
```
```
1 2 3 4 5 6 7 8 9
```
```
<bean id="transactionManager"
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource" />
</bean>
```
```
1
```
```
2
3
4
```

如果我们不想使用事物 @Transactional 注解，想自己进行事物控制 (编程事物管理)，控制某一段的代

码事务生效或者回滚，但是又不想自己去编写那么多的代码，怎么办呢？

可以使用 springboot 中的 DataSourceTransactionManager 和 TransactionDefinition 这两个类来
结合使用，能够达到手动控制事物的提交回滚。

代码示例:

```
@Autowired
private DataSourceTransactionManager dataSourceTransactionManager;
@Autowired
private TransactionDefinition transactionDefinition;
```
```
public boolean transferMoney(User user) {
/*
* 手动进行事物控制
*/
TransactionStatus transactionStatus=null;
boolean isCommit = false;
try {
//开启事务
transactionStatus =
```
```
dataSourceTransactionManager.getTransaction(transactionDefinition);
System.out.println("查询的数据1:" + udao.findById(user.getId()));
// 进行新增/修改
udao.insert(user);
System.out.println("查询的数据2:" + udao.findById(user.getId()));
if(user.getAge()< 20 ) {
user.setAge(user.getAge()+ 2 );
udao.update(user);
System.out.println("查询的数据3:" +
udao.findById(user.getId()));
}else {
throw new Exception("模拟一个异常!");
}
```
```
//手动提交
dataSourceTransactionManager.commit(transactionStatus);
```
```
isCommit= true;
System.out.println("手动提交事物成功!");
throw new Exception("模拟第二个异常!");
```
```
} catch (Exception e) {
//如果未提交就进行回滚
if(!isCommit){
System.out.println("发生异常,进行手动回滚！");
//手动回滚事物
dataSourceTransactionManager.rollback(transactionStatus);
}
e.printStackTrace();
}
return false;
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
```
```
16
17
18
19
20
21
22
23
```
```
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
```

特别说明：

```
在进行使用的时候，需要注意在回滚的时候，要确保开启了事物但是未提交，如果未开启或已提
交的时候进行回滚是会在catch里面发生异常的！
```
实际上，DataSourceTransactionManager 是通过调用 java. sql. Connection 来管理事务，而后者是通过
DataSource 获取到的。通过调用连接的 commit () 方法来提交事务，同样，事务失败则通过调用
rollback () 方法进行回滚。

###### Hibernate 事务管理器

如果应用程序的持久化是通过 Hibernate 实现的，那么你需要使用 HibernateTransactionManager。对
于 Hibernate 3，需要在 Spring 上下文定义中添加如下的<bean>声明：

sessionFactory 属性需要装配一个 Hibernate 的 session 工厂，HibernateTransactionManager 的实现细
节是它将事务管理的职责委托给 org. hibernate. Transaction 对象，而后者是从 Hibernate Session 中获
取到的。

当事务成功完成时，HibernateTransactionManager 将会调用 Transaction 对象的 commit () 方法，反
之，将会调用 rollback () 方法。

###### Java 持久化 API（JPA）事务管理器

Hibernate 多年来一直是事实上的 Java 持久化标准，但是现在 Java 持久化 API 作为真正的 Java 持久化标准
进入大家的视野。

如果你计划使用 JPA 的话，那你需要使用 Spring 的 JpaTransactionManager 来处理事务。

你需要在 Spring 中这样配置 JpaTransactionManager：

JpaTransactionManager 只需要装配一个 JPA 实体管理工厂（javax. persistence. EntityManagerFactory
接口的任意实现）。

JpaTransactionManager 将与由工厂所产生的 JPA EntityManager 合作来构建事务。

###### Java 原生 API 事务管理器

如果你没有使用以上所述的事务管理，或者是跨越了多个事务管理源（比如两个或者是多个不同的数据
源），你就需要使用 JtaTransactionManager：

```
<bean id="transactionManager"
class="org. springframework. orm. hibernate 3. HibernateTransactionManager">
<property name="sessionFactory" ref="sessionFactory" />
</bean>
```
```
1
```
```
2
3
```
```
<bean id="transactionManager"
class="org. springframework. orm. jpa. JpaTransactionManager">
<property name="sessionFactory" ref="sessionFactory" />
</bean>
```
```
1
```
```
2
3
```
```
<bean id="transactionManager"
class="org. springframework. transaction. jta. JtaTransactionManager">
<property name="transactionManagerName"
value="java:/TransactionManager" />
</bean>
```
```
1
```
```
2
```
```
3
```

JtaTransactionManager 将事务管理的责任委托给 javax. transaction. UserTransaction 和
javax. transaction. TransactionManager 对象，其中事务成功完成通过 UserTransaction.commit () 方法
提交，事务失败通过 UserTransaction.rollback () 方法回滚。

#### TransactionDefinition 基本事务属性定义

如何开启事务？

事务管理器接口 PlatformTransactionManager 通过 getTransaction (TransactionDefinition definition)
方法来开启一个事务，这个方法里面的参数是 TransactionDefinition 类，这个类就定义了一些基本的事
务属性。

那么什么是事务属性呢？

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。

总体来说，事务属性包含了 5 个方面，如图所示：

TransactionDefinition 接口内容如下：

我们可以发现 TransactionDefinition 正好用来定义事务属性，下面详细介绍一下各个事务属性。

###### 核心事务属性 1 ：传播行为属性

```
public interface TransactionDefinition {
int getPropagationBehavior (); // 返回事务的传播行为
int getIsolationLevel (); // 返回事务的隔离级别，事务管理器根据它来控制另外一
个事务可以看到本事务内的哪些数据
int getTimeout ();  // 返回事务必须在多少秒内完成
boolean isReadOnly (); // 事务是否只读，事务管理器能够根据这个返回值进行优化，确保
事务是只读的
}
```
```
1 2 3 4 5 6
```

```
传播行为含义
```
```
PROPAGATION_REQUIRED
表示当前方法必须运行在事务中。如果当前事务存在，方
法将会在该事务中运行。否则，会启动一个新的事务
```
```
PROPAGATION_SUPPORTS
```
```
表示当前方法不需要事务上下文，但是如果存在当前事务
的话，那么该方法会在这个事务中运行
```
```
PROPAGATION_MANDATORY
```
```
表示该方法必须在事务中运行，如果当前事务不存在，则
会抛出一个异常
```
```
PROPAGATION_REQUIRED_NEW
```
```
表示当前方法必须运行在它自己的事务中。一个新的事务
将被启动。如果存在当前事务，在该方法执行期间，当前
事务会被挂起。如果使用 JTATransactionManager 的话，
则需要访问 TransactionManager
```
```
PROPAGATION_NOT_SUPPORTED
```
```
表示该方法不应该运行在事务中。如果存在当前事务，在
该方法运行期间，当前事务将被挂起。如果使用
JTATransactionManager 的话，则需要访问
TransactionManager
```
```
PROPAGATION_NEVER
表示当前方法不应该运行在事务上下文中。如果当前正有
一个事务在运行，则会抛出异常
```
```
PROPAGATION_NESTED
```
```
表示如果当前已经存在一个事务，那么该方法将会在嵌套
事务中运行。嵌套的事务可以独立于当前事务进行单独地
提交或回滚。如果当前事务不存在，那么其行为与
PROPAGATION_REQUIRED 一样。注意各厂商对这种传播
行为的支持是有所差异的。可以参考资源管理器的文档来
确认它们是否支持嵌套事务
```
事务的第一个方面是传播行为（propagation behavior）。

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。

例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。

Spring 定义了七种传播行为：

使用 spring 声明式事务，spring 使用 AOP 来支持声明式事务，会根据事务属性，自动在方法调用之前决
定是否开一个事务，并在方法执行之后决定事务提交或回滚事务。

（ 1 ）PROPAGATION_REQUIRED

PROPAGATION_REQUIRED 功能说明：

```
如果存在一个事务，则支持当前事务。
如果没有事务，则开启一个新的事务。
```
methodA 调用 methodB


methodB () 的代码

如果单独调用 methodB 方法：

Spring 保证在 methodB 方法中所有的调用，都获得到一个相同的连接。在调用 methodB 时，没有一个
存在的事务，所以获得一个新的连接，开启了一个新的事务。

相当于

如果是先调用 MethodA 时，在 MethodA 内又会调用 MethodB. 执行效果相当于：

```
//事务属性 PROPAGATION_REQUIRED
methodA{
......
methodB ();
......
}
```
```
1 2 3 4 5 6
```
```
//事务属性 PROPAGATION_REQUIRED
methodB{
......
}
```
```
1
2
3
4
```
```
main{
metodB ();
}
```
```
1
2
3
```
```
Main{
Connection con=null;
try{
con = getConnection ();
con.setAutoCommit (false);
```
```
//方法调用
methodB ();
```
```
//提交事务
con.commit ();
} Catch (RuntimeException ex) {
//回滚事务
con.rollback ();
} finally {
//释放资源
closeCon ();
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
```

当在 MethodA 中调用 MethodB 时，环境中已经有了一个事务，所以 methodB 就加入当前事务。

（ 2 ）PROPAGATION_SUPPORTS

功能说明：

```
如果存在一个事务，支持当前事务。
如果没有事务，则非事务的执行。
```
但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS 与不使用事务有少许不同。

单纯的调用 methodB 时，methodB 方法是非事务的执行的。

当调用 methdA 时, methodB 则加入了 methodA 的事务中, 事务地执行。

（ 3 ）PROPAGATION_MANDATORY

功能说明：

```
如果已经存在一个事务，支持当前事务。
如果没有一个活动的事务，则抛出异常。
```
当单独调用 methodB 时，因为当前没有一个活动的事务，

```
main{
Connection con = null;
try{
con = getConnection ();
methodA ();
con.commit ();
} catch (RuntimeException ex) {
con.rollback ();
} finally {
closeCon ();
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
```
```
//事务属性 PROPAGATION_REQUIRED
methodA (){
methodB ();
}
```
```
//事务属性 PROPAGATION_SUPPORTS
methodB (){
......
}
```
```
1 2 3 4 5 6 7 8 9
```
```
//事务属性 PROPAGATION_REQUIRED
methodA (){
methodB ();
}
```
```
//事务属性 PROPAGATION_MANDATORY
methodB (){
......
}
```
```
1 2 3 4 5 6 7 8 9
```

则会抛出异常

当调用 methodA 时，methodB 则加入到 methodA 的事务中，事务地执行。

（ 4 ）PROPAGATION_REQUIRES_NEW

```
总是开启一个新的事务。
如果一个事务已经存在，则将这个存在的事务挂起。
```
下面是一个例子

调用 A 方法：

相当于

```
throw new IllegalTransactionStateException (“Transaction propagation
‘mandatory’ but no existing transaction found”);
```
```
1
```
```
2
```
```
//事务属性 PROPAGATION_REQUIRED
methodA (){
doSomeThingA ();
methodB ();
doSomeThingB ();
}
```
```
//事务属性 PROPAGATION_REQUIRES_NEW
methodB (){
......
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
main (){
methodA ();
}
```
```
1
2
3
```
```
main (){
TransactionManager tm = null;
try{
//获得一个 JTA 事务管理器
tm = getTransactionManager ();
tm.begin ();//开启一个新的事务
Transaction ts 1 = tm.getTransaction ();
doSomeThing ();
tm.suspend ();//挂起当前事务
try{
tm.begin ();//重新开启第二个事务
Transaction ts 2 = tm.getTransaction ();
methodB ();
ts 2.commit ();//提交第二个事务
} Catch (RunTimeException ex) {
ts 2.rollback ();//回滚第二个事务
} finally {
//释放资源
}
//methodB 执行完后，恢复第一个事务
tm.resume (ts 1);
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
```

在这里，我把 ts 1 称为外层事务，ts 2 称为内层事务。

从上面的代码可以看出，ts 2 与 ts 1 是两个独立的事务，互不相干。

Ts 2 是否成功并不依赖于 ts 1。

如果 methodA 方法在调用 methodB 方法后的 doSomeThingB 方法失败了，而 methodB 方法所做的结果
依然被提交。而除了 methodB 之外的其它代码导致的结果却被回滚了。

另外，使用 PROPAGATION_REQUIRES_NEW, 需要使用 JtaTransactionManager 作为事务管理器。

（ 5 ）PROPAGATION_NOT_SUPPORTED

功能说明：

```
总是非事务地执行，并挂起任何存在的事务。
```
使用 PROPAGATION_NOT_SUPPORTED, 也需要使用 JtaTransactionManager 作为事务管理器。

（ 6 ）PROPAGATION_NEVER

功能说明：

```
总是非事务地执行
如果存在一个活动事务，则抛出异常。
```
（ 7 ）PROPAGATION_NESTED

功能说明：

```
如果一个活动的事务存在，则运行在一个嵌套的事务中.
如果没有活动事务, 则按 TransactionDefinition. PROPAGATION_REQUIRED 属性执行。
```
这是一个嵌套事务, 使用 JDBC 3.0 驱动时, 仅仅支持 DataSourceTransactionManager 作为事务管理器。

需要 JDBC 驱动的 java. sql. Savepoint 类。有一些 JTA 的事务管理器实现可能也提供了同样的功能。

使用 PROPAGATION_NESTED，还需要把 PlatformTransactionManager 的 nestedTransactionAllowed
属性设为 true; 而 nestedTransactionAllowed 属性值默认为 false。

```
doSomeThingB ();
ts 1.commit ();//提交第一个事务
} catch (RunTimeException ex) {
ts 1.rollback ();//回滚第一个事务
} finally {
//释放资源
}
}
```
```
22
23
24
25
26
27
28
29
```
```
//事务属性 PROPAGATION_REQUIRED
methodA (){
doSomeThingA ();
methodB ();
doSomeThingB ();
}
```
```
//事务属性 PROPAGATION_NESTED
methodB (){
......
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```

如果单独调用 methodB 方法，则按 REQUIRED 属性执行。如果调用 methodA 方法，相当于下面的效
果：

当 methodB 方法调用之前，调用 setSavepoint 方法，保存当前的状态到 savepoint。

如果 methodB 方法调用失败，则恢复到之前保存的状态。

但是需要注意的是，这时的事务并没有进行提交，如果后续的代码 (doSomeThingB () 方法) 调用失败，则
回滚包括 methodB 方法的所有操作。

嵌套事务一个非常重要的概念就是内层事务依赖于外层事务。外层事务失败时，会回滚内层事务所做的
动作。而内层事务操作失败并不会引起外层事务的回滚。

PROPAGATION_NESTED 与 PROPAGATION_REQUIRES_NEW 的区别: 它们非常类似, 都像一个嵌套事
务，如果不存在一个活动的事务，都会开启一个新的事务。使用 PROPAGATION_REQUIRES_NEW 时，
内层事务与外层事务就像两个独立的事务一样，一旦内层事务进行了提交后，外层事务不能对其进行回
滚。两个事务互不影响。两个事务不是一个真正的嵌套事务。同时它需要 JTA 事务管理器的支持。

使用 PROPAGATION_NESTED 时，外层事务的回滚可以引起内层事务的回滚。而内层事务的异常并不会
导致外层事务的回滚，它是一个真正的嵌套事务。DataSourceTransactionManager 使用 savepoint 支
持 PROPAGATION_NESTED 时，需要 JDBC 3.0 以上驱动及 1.4 以上的 JDK 版本支持。其它的 JTA
TrasactionManager 实现可能有不同的支持方式。

PROPAGATION_REQUIRES_NEW 启动一个新的, 不依赖于环境的 “内部” 事务. 这个事务将被完全
commited 或 rolled back 而不依赖于外部事务, 它拥有自己的隔离范围, 自己的锁, 等等. 当内部事务开
始执行时, 外部事务将被挂起, 内务事务结束时, 外部事务将继续执行。

另一方面, PROPAGATION_NESTED 开始一个 “嵌套的” 事务, 它是已经存在事务的一个真正的子事务. 潜
套事务开始执行时, 它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套
事务是外部事务的一部分, 只有外部事务结束后它才会被提交。

由此可见, PROPAGATION_REQUIRES_NEW 和 PROPAGATION_NESTED 的最大区别在于,

```
main (){
Connection con = null;
Savepoint savepoint = null;
try{
con = getConnection ();
con.setAutoCommit (false);
doSomeThingA ();
savepoint = con 2.setSavepoint ();
try{
methodB ();
} catch (RuntimeException ex) {
con.rollback (savepoint);
} finally {
//释放资源
}
doSomeThingB ();
con.commit ();
} catch (RuntimeException ex) {
con.rollback ();
} finally {
//释放资源
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
19
20
21
22
23
```

```
PROPAGATION_REQUIRES_NEW 完全是一个新的事务, 而 PROPAGATION_NESTED 则是外部事
务的子事务, 如果外部事务 commit, 嵌套事务也会被 commit, 这个规则同样适用于 roll back.
```
事务的传播属性，当如何选择？

一般情况下，PROPAGATION_REQUIRED 应该是我们首先的事务传播行为。它能够满足我们大多数的事
务需求。

###### 核心事务属性 2 ：隔离级别属性

事务的第二个维度就是隔离级别（isolation level）。

隔离级别定义了一个事务可能受其他并发事务影响的程度。

（ 1 ）并发事务引起的问题

在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务。并发虽然是必须
的，但可能会导致一下的问题。

```
脏读（Dirty reads）——脏读发生在一个事务读取了另一个事务改写但尚未提交的数据时。
如果改写在稍后被回滚了，那么第一个事务获取的数据就是无效的。
不可重复读（Nonrepeatable read）——不可重复读发生在一个事务执行相同的查询两次或
两次以上，但是每次都得到不同的数据时。这通常是因为另一个并发事务在两次查询期间进
行了更新。
幻读（Phantom read）——幻读与不可重复读类似。它发生在一个事务（T 1）读取了几行
数据，接着另一个并发事务（T 2）插入了一些数据时。在随后的查询中，第一个事务（T 1）
就会发现多了一些原本不存在的记录。
```
**不可重复读与幻读的区别**

```
（ 1 ）不可重复读的重点是修改
（ 2 ）幻读的重点在于新增或者删除：
```
（ 1 ）不可重复读的重点是修改:

同样的条件, 你读取过的数据, 再次读取出来发现值不一样了
例如：在事务 1 中，Mary 读取了自己的工资为 1000, 操作并没有完成

在事务 2 中，这时财务人员修改了 Mary 的工资为 2000, 并提交了事务.

在事务 1 中，Mary 再次读取自己的工资时，工资变为了 2000

在一个事务中前后两次读取的结果并不一致，导致了不可重复读。

（ 2 ）幻读的重点在于新增或者删除：

```
con 1 = getConnection ();
select salary from employee empId ="Mary";
```
```
1
2
```
```
con 2 = getConnection ();
update employee set salary = 2000 ;
con 2.commit ();
```
```
1
2
3
```
```
//con 1
select salary from employee empId ="Mary";
```
```
1
2
```

```
隔离级别含义
```
```
ISOLATION_DEFAULT 使用后端数据库默认的隔离级别
```
```
ISOLATION_READ_UNCOMMITTED
```
```
最低的隔离级别，允许读取尚未提交的数据变更，可能会
导致脏读、幻读或不可重复读
```
```
ISOLATION_READ_COMMITTED
```
```
允许读取并发事务已经提交的数据，可以阻止脏读，但是
幻读或不可重复读仍有可能发生
```
```
ISOLATION_REPEATABLE_READ
```
```
对同一字段的多次读取结果都是一致的，除非数据是被本
身事务自己所修改，可以阻止脏读和不可重复读，但幻读
仍有可能发生
```
```
ISOLATION_SERIALIZABLE
```
```
最高的隔离级别，完全服从 ACID 的隔离级别，确保阻止
脏读、不可重复读以及幻读，也是最慢的事务隔离级别，
因为它通常是通过完全锁定事务相关的数据库表来实现的
```
同样的条件, 第 1 次和第 2 次读出来的记录数不一样
例如：目前工资为 1000 的员工有 10 人。

事务 1, 读取所有工资为 1000 的员工。

共读取 10 条记录

这时另一个事务向 employee 表插入了一条员工记录，工资也为 1000

事务 1 再次读取所有工资为 1000 的员工

共读取到了 11 条记录，这就产生了幻像读。

从总的结果来看, 似乎不可重复读和幻读都表现为两次读取的结果不一致。

但如果你从控制的角度来看, 两者的区别就比较大。

```
对于不可重复读, 只需要锁住满足条件的记录。
对于幻读, 要锁住满足条件及其相近的记录。
```
**Spring 隔离级别的定义**

数据库有自己的隔离级别的定义，Spring 也有自己的隔离级别的定义

###### 核心属性 3 ： 事务只读属性

事务只读属性: 表示这个事务只读取数据但不更新数据, 这样可以帮助数据库引擎优化事务。

```
con 1 = getConnection ();
Select * from employee where salary = 1000 ; 12
```
```
1
2
```
```
con 2 = getConnection ();
Insert into employee (empId, salary) values ("Lili", 1000 );
con 2.commit ();
```
```
1
2
3
```
```
//con 1
select * from employee where salary = 1000 ;
```
```
1
2
```

具体来说： 如果事务只对后端的数据库进行读操作，数据库可以利用事务的只读特性来进行一些特定的
优化。通过将事务设置为只读，你就可以给数据库一个机会，让它应用它认为合适的优化措施。

当使用 @Transaction 注解时，可以通过设置 readOnly=true 来指定这是一个只读事务，这样在事务

执行期间就不会对数据进行修改，只会进行查询操作。

以下是一个使用 @Transaction 只读示例的代码片段：

在上面的示例中，getUserById 方法被标记为只读事务，因此在执行期间只会进行查询操作。

如果在方法中尝试进行修改操作，将会抛出异常。

###### 核心属性 4 ：事务超时属性

事务超时属性：事务在强制回滚之前可以保持多久。这样可以防止长期运行的事务占用资源。

为了使应用程序很好地运行，事务不能运行太长的时间。

因为事务可能涉及对后端数据库的锁定，所以长时间的事务会不必要的占用数据库资源。

事务超时就是事务的一个定时器，在特定时间内事务如果没有执行完毕，那么就会自动回滚，而不是一
直等待其结束。

```
@Transactional 注解中的 timeout 属性用于设置事务的超时时间，单位为秒。如果事务在超过指定
```
时间后仍未完成，则会被强制回滚。

以下是一个示例：

```
@Service
public class UserService {
```
```
@Autowired
private UserRepository userRepository;
```
```
@Transactional (readOnly = true)
public User getUserById (Long id) {
return userRepository.findById (id). orElse (null);
}
```
```
// 其他方法...
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
```
```
@Service
public class UserService {
@Autowired
private UserRepository userRepository;
```
```
@Transactional (timeout = 5 )
public void updateUser (User user) {
userRepository.save (user);
// 执行一些其他操作
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```

在上面的示例中，updateUser 方法使用了 @Transactional 注解，并设置了 timeout 属性为 5

秒。如果该方法执行时间超过 5 秒，事务将会被强制回滚。

需要注意的是，如果在事务内部调用了其他带有事务注解的方法，那么这些方法的超时时间也会受到影
响。

因此，在设置事务超时时间时，需要考虑到事务内部所涉及的所有方法的执行时间。

###### 核心属性 5 ：Spring 事务的回滚规则

事务五边形的最后一个方面是一组规则，这些规则定义了哪些异常会导致事务回滚而哪些不会。

好的，下面是一个简单的 Java 代码示例，演示了 @Transactional 注解的使用以及如何在出现异常时

回滚事务：

在上面的示例中，createUser 方法上使用了 @Transactional 注解，表示这个方法需要在事务管理

下执行。如果在方法执行过程中发生异常，事务会自动回滚，保证数据的一致性。

在这个例子中，如果用户创建失败，createUser 方法会抛出一个 RuntimeException 异常，这会导

致事务回滚，用户创建操作会被撤销。

需要注意的是，@Transactional 注解只能应用于公共方法，因为只有公共方法才能被代理，从而实

现事务管理。

此外，@Transactional 注解默认只对受检查异常进行回滚，而对非受检查异常不进行回滚。

```
非检查型异常指（Unchecked Exception）的是程序在编译时不会提示需要处理该异常，而是在
运行时才会出现异常。
```
```
检查型异常（Checked Exception）是指在 Java 中，编译器会强制要求对可能会抛出这些异常的
代码进行异常处理，否则代码将无法通过编译。
```
一般来说，在编写代码时应该尽量避免抛出非检查型异常，因为这些异常的发生通常意味着程序存在严
重的逻辑问题。

如果是 **检查型异常（Checked Exception）** , 可以声明事务在遇到特定的检查型异常时像遇到运行期异
常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。

如果需要对非受检查异常也进行回滚，可以在 @Transactional 注解中指定 rollbackFor 属性，例

如

```
@Service
public class UserService {
```
```
@Autowired
private UserRepository userRepository;
```
```
@Transactional
public void createUser (User user) {
userRepository.save (user);
if (user.getId () == null) {
throw new RuntimeException ("Failed to create user");
}
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
```

#### 核心流程一：AOP 动态代理装配流程源码剖析

###### 注解@EnableTransactionManagement

我们知道了使用 AOP 技术实现，那到底是如何实现的呢？

我们从 @EnableTransactionManagement 注解聊起，我们点进该注解：

很明显，TransactionManagementConfigurationSelector 类是我们主要关注的内容

一共注册了两个：

```
AutoProxyRegistrar. class：注册 AOP 处理器
ProxyTransactionManagementConfiguration. class：代理事务配置，
```
注册事务需要用的一些类，而且 Role=ROLE_INFRASTRUCTURE 都是属于内部级别的

```
@Transactional (rollbackFor = Exception. class)
public void createUser (User user) {
userRepository.save (user);
if (user.getId () == null) {
throw new RuntimeException ("Failed to create user");
}
}
```
```
1 2 3 4 5 6 7
```
```
@Import (TransactionManagementConfigurationSelector. class)
public @interface EnableTransactionManagement {
```
```
1
2
3
```
```
public class TransactionManagementConfigurationSelector extends
AdviceModeImportSelector<EnableTransactionManagement> {
```
```
/**
* 此处是 AdviceMode 的作用，默认是用代理，另外一个是 ASPECTJ
*/
@Override
protected String[] selectImports (AdviceMode adviceMode) {
switch (adviceMode) {
case PROXY:
return new String[] {AutoProxyRegistrar.class.getName (),
ProxyTransactionManagementConfiguration.class.getName ()};
case ASPECTJ:
return new String[] {determineTransactionAspectClass ()};
default:
return null;
}
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
14
15
16
17
18
```
```
@Configuration (proxyBeanMethods = false)
@Role (BeanDefinition. ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends
AbstractTransactionManagementConfiguration {
```
```
1
2
3
```

到这里，看到 BeanFactoryTransactionAttributeSourceAdvisor 以 advisor 结尾的类，这就是进行创建
AOP 动态代理对象的核心类，其 Interceptor 为 **transactionInterceptor**

到这里，我们思考一下，利用我们之前学习到的 AOP 的源码，猜测其运行逻辑：

```
我们在方法上写上 @EnableTransactionManagement 注解，Spring 会注册一个
BeanFactoryTransactionAttributeSourceAdvisor 的类
创建对应的方法 Bean 时，会和 AOP 一样，利用该 Advisor 类生成对应的代理对象
最终调用方法时，会调用代理对象，并通过环绕增强来达到事务的功能
```
#### 核心流程二：AOP 动态代理执行流程源码剖析

动态代理对象的代理方法被执行后，会执行到 Interceptor 拦截器。

```
@Bean (name =
TransactionManagementConfigUtils. TRANSACTION_ADVISOR_BEAN_NAME)
@Role (BeanDefinition. ROLE_INFRASTRUCTURE)
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor (
TransactionAttributeSource transactionAttributeSource,
TransactionInterceptor transactionInterceptor) {
// 【重点】注册了 BeanFactoryTransactionAttributeSourceAdvisor 的
advisor
// 其 advice 为 transactionInterceptor
BeanFactoryTransactionAttributeSourceAdvisor advisor = new
BeanFactoryTransactionAttributeSourceAdvisor ();
advisor.setTransactionAttributeSource (transactionAttributeSource);
advisor.setAdvice (transactionInterceptor);
if (this. enableTx != null) {
advisor.setOrder (this. enableTx.<Integer>getNumber ("order"));
}
return advisor;
}
```
```
@Bean
@Role (BeanDefinition. ROLE_INFRASTRUCTURE)
public TransactionAttributeSource transactionAttributeSource () {
return new AnnotationTransactionAttributeSource ();
}
```
```
@Bean
@Role (BeanDefinition. ROLE_INFRASTRUCTURE)
public TransactionInterceptor
transactionInterceptor (TransactionAttributeSource
transactionAttributeSource) {
TransactionInterceptor interceptor = new TransactionInterceptor ();
```
```
interceptor.setTransactionAttributeSource (transactionAttributeSource);
if (this. txManager != null) {
interceptor.setTransactionManager (this. txManager);
}
return interceptor;
}
}
```
```
4 5 6 7 8 9
```
```
10
11
```
```
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
```
```
29
30
```
```
31
32
33
34
35
36
```

我们从上面看到其 Interceptor 正是 TransactionInterceptor。代理对象运行时，会拿到所有的
Interceptor 并进行排序，责任链递归运行

这里先看一下 TransactionInterceptor 类图

然后看其源码内容：

这里的 invoke 方法执行的时候，会执行 invokeWithinTransaction ，具体看代码

```
public class TransactionInterceptor extends TransactionAspectSupport
implements MethodInterceptor {
@Override
@Nullable
public Object invoke (MethodInvocation invocation) throws Throwable {
// 获取我们的代理对象的 class 属性
Class<?> targetClass = (invocation.getThis () != null?
AopUtils.getTargetClass (invocation.getThis ()) : null);
```
```
// Adapt to TransactionAspectSupport's invokeWithinTransaction...
/**
* 以事务的方式调用目标方法
* 在这埋了一个钩子函数用来回调目标方法的
*/
return invokeWithinTransaction (invocation.getMethod (), targetClass,
invocation::proceed);
}
}
```
```
@Nullable
protected Object invokeWithinTransaction (Method method, @Nullable Class<?>
targetClass, final InvocationCallback invocation){
// 获取我们的事务属性源对象
TransactionAttributeSource tas = getTransactionAttributeSource ();
// 通过事务属性源对象获取到当前方法的事务属性信息
final TransactionAttribute txAttr = (tas != null?
tas.getTransactionAttribute (method, targetClass) : null);
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
```
```
14
15
16
17
18
```
```
19
20
21
22
```

```
// 获取我们配置的事务管理器对象
final TransactionManager tm = determineTransactionManager (txAttr);
```
```
if (txAttr == null || !(ptm instanceof
CallbackPreferringPlatformTransactionManager)) {
// 【重点】创建 TransactionInfo
TransactionInfo txInfo = createTransactionIfNecessary (ptm,
txAttr, joinpointIdentification);
try {
// 执行被增强方法, 调用具体的处理逻辑
// 我们实际的业务方法
retVal = invocation.proceedWithInvocation ();
}
catch (Throwable ex) {
// 异常回滚，事务回滚
completeTransactionAfterThrowing (txInfo, ex);
throw ex;
}
finally {
//清除事务信息，恢复线程私有的老的事务信息
cleanupTransactionInfo (txInfo);
}
// 提交事务
//成功后提交，会进行资源储量，连接释放，恢复挂起事务等操作
commitTransactionAfterReturning (txInfo);
return retVal;
}
}
```
```
// 创建连接 + 开启事务
protected TransactionInfo createTransactionIfNecessary (@Nullable
PlatformTransactionManager tm,
@Nullable TransactionAttribute txAttr, final String
joinpointIdentification) {
// 获取 TransactionStatus 事务状态信息
status = tm.getTransaction (txAttr);
```
```
// 根据指定的属性与 status 准备一个 TransactionInfo，
return prepareTransactionInfo (tm, txAttr, joinpointIdentification,
status);
}
```
```
// 存在异常时回滚事务
protected void completeTransactionAfterThrowing (@Nullable TransactionInfo
txInfo, Throwable ex) {
// 进行回滚
txInfo.getTransactionManager (). rollback (txInfo.getTransactionStatus ());
}
```
```
// 调用事务管理器的提交方法
protected void commitTransactionAfterReturning (@Nullable TransactionInfo
txInfo){
txInfo.getTransactionManager (). commit (txInfo.getTransactionStatus ());
}
```
23
24
25
26
27

28
29

30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52

53

54
55
56
57
58

59
60
61
62

63
64
65
66
67
68

69
70
71


从上面的拦截器可以看到

```
在业务方法前面，开启事务
业务方法后面，捕获异常，执行事务回滚
如果没有异常，提交事务
```
#### 核心流程三：事务执行流程源码剖析

Spring 事务设计了统一的事务管理器超级接口， **PlatformTransactionManager** 是 Spring 事务管理
器设计的基类。

先看看看 PlatformTransactionManager 的实现，其实现一共三个方法：

```
开启事务：TransactionStatus getTransaction (@Nullable TransactionDefinition definition)
提交事务：void commit (TransactionStatus status)
回滚事务：void rollback (TransactionStatus status)
```
接下来，我们分别看一下其如何实现的

**开启事务源码分析**

我们想一下，在开启事务这一阶段，我们会做什么功能呢？ 参考 DataSourceTransactionManager，这
个阶段应该会 **创建连接并且开启事务**

```
public final TransactionStatus getTransaction (@Nullable
TransactionDefinition definition){
```
```
// PROPAGATION_REQUIRED，PROPAGATION_REQUIRES_NEW，PROPAGATION_NESTED 都需
要新建事务
```
```
1
```
```
2
3
```

```
if (def.getPropagationBehavior () ==
TransactionDefinition. PROPAGATION_REQUIRED ||
def.getPropagationBehavior () ==
TransactionDefinition. PROPAGATION_REQUIRES_NEW ||
def.getPropagationBehavior () ==
TransactionDefinition. PROPAGATION_NESTED) {
//没有当前事务的话，REQUIRED，REQUIRES_NEW，NESTED 挂起的是空事务，然后创建一
个新事务
SuspendedResourcesHolder suspendedResources = suspend (null);
try {
// 看这里重点：开始事务
return startTransaction (def, transaction, debugEnabled,
suspendedResources);
}
catch (RuntimeException | Error ex) {
// 恢复挂起的事务
resume (null, suspendedResources);
throw ex;
}
}
}
```
```
private TransactionStatus startTransaction (TransactionDefinition
definition, Object transaction, boolean debugEnabled, @Nullable
SuspendedResourcesHolder suspendedResources) {
// 是否需要新同步
boolean newSynchronization = (getTransactionSynchronization () !=
SYNCHRONIZATION_NEVER);
// 创建新的事务
DefaultTransactionStatus status = newTransactionStatus ( definition,
transaction, true, newSynchronization, debugEnabled, suspendedResources);
// 【重点】开启事务和连接
doBegin (transaction, definition);
// 新同步事务的设置，针对于当前线程的设置
prepareSynchronization (status, definition);
return status;
}
```
```
protected void doBegin (Object transaction, TransactionDefinition
definition) {
// 判断事务对象没有数据库连接持有器
if (! txObject.hasConnectionHolder () ||
txObject.getConnectionHolder (). isSynchronizedWithTransaction ()) {
// 【重点】通过数据源获取一个数据库连接对象
Connection newCon = obtainDataSource (). getConnection ();
// 把我们的数据库连接包装成一个 ConnectionHolder 对象然后设置到我们的 txObject
对象中去
// 再次进来时，该 txObject 就已经有事务配置了
txObject.setConnectionHolder (new ConnectionHolder (newCon), true);
}
```
```
// 【重点】获取连接
con = txObject.getConnectionHolder (). getConnection ();
```
```
// 为当前的事务设置隔离级别【数据库的隔离级别】
Integer previousIsolationLevel =
DataSourceUtils.prepareConnectionForTransaction (con, definition);
```
```
4 5 6 7 8 9
```
10
11

12
13
14
15
16
17
18
19
20
21

22
23

24
25

26
27
28
29
30
31
32
33

34
35
36
37
38
39

40
41
42
43
44
45
46
47
48
49


到这里，我们的获取事务接口完成了数据库连接的创建和关闭自动提交（开启事务），将
Connection 注册到了缓存（resources）当中，便于获取。

**提交事务源码分析**

```
// 设置先前隔离级别
txObject.setPreviousIsolationLevel (previousIsolationLevel);
// 设置是否只读
txObject.setReadOnly (definition.isReadOnly ());
```
```
// 关闭自动提交
if (con.getAutoCommit ()) {
//设置需要恢复自动提交
txObject.setMustRestoreAutoCommit (true);
// 【重点】关闭自动提交
con.setAutoCommit (false);
}
```
```
// 判断事务是否需要设置为只读事务
prepareTransactionalConnection (con, definition);
// 标记激活事务
txObject.getConnectionHolder (). setTransactionActive (true);
```
```
// 设置事务超时时间
int timeout = determineTimeout (definition);
if (timeout != TransactionDefinition. TIMEOUT_DEFAULT) {
txObject.getConnectionHolder (). setTimeoutInSeconds (timeout);
}
```
```
// 绑定我们的数据源和连接到我们的同步管理器上，把数据源作为 key, 数据库连接作为 value
设置到线程变量中
if (txObject.isNewConnectionHolder ()) {
// 将当前获取到的连接绑定到当前线程
TransactionSynchronizationManager.bindResource (obtainDataSource (),
txObject.getConnectionHolder ());
}
}
}
```
```
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
```
```
75
76
77
```
```
78
79
80
```
```
public final void commit (TransactionStatus status) throws
TransactionException {
DefaultTransactionStatus defStatus = (DefaultTransactionStatus)
status;
// 如果在事务链中已经被标记回滚，那么不会尝试提交事务，直接回滚
if (defStatus.isLocalRollbackOnly ()) {
// 不可预期的回滚
processRollback (defStatus, false);
return;
}
```
```
// 设置了全局回滚
if (! shouldCommitOnGlobalRollbackOnly () &&
defStatus.isGlobalRollbackOnly ()) {
// 可预期的回滚，可能会报异常
processRollback (defStatus, true);
return;
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
12
13
14
15
```

这里比较重要的有两个步骤：

doCommit：提交事务（直接使用 JDBC 提交即可）

cleanupAfterCompletion：数据清除，与线程中的私有资源解绑，方便释放

```
// 【重点】处理事务提交
processCommit (defStatus);
}
```
```
// 处理提交，先处理保存点，然后处理新事务，如果不是新事务不会真正提交，要等外层是新事务的
才提交，
// 最后根据条件执行数据清除, 线程的私有资源解绑，重置连接自动提交，隔离级别，是否只读，释放
连接，恢复挂起事务等
private void processCommit (DefaultTransactionStatus status) throws
TransactionException {;
// 如果是独立的事务则直接提交
doCommit (status);
```
```
//根据条件，完成后数据清除, 和线程的私有资源解绑，重置连接自动提交，隔离级别，是否只读，
释放连接，恢复挂起事务等
cleanupAfterCompletion (status);
}
```
```
16
17
18
19
20
21
```
```
22
```
```
23
```
```
24
25
26
```
```
27
28
```
```
protected void doCommit (DefaultTransactionStatus status) {
DataSourceTransactionObject txObject = (DataSourceTransactionObject)
status.getTransaction ();
Connection con = txObject.getConnectionHolder (). getConnection ();
try {
// JDBC 连接提交
con.commit ();
}
catch (SQLException ex) {
throw new TransactionSystemException ("Could not commit JDBC
transaction", ex);
}
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
// 线程同步状态清除
TransactionSynchronizationManager.clear ();
```
```
// 清除同步状态【这些都是线程的缓存，使用 ThreadLocal 的】
public static void clear () {
synchronizations.remove ();
currentTransactionName.remove ();
currentTransactionReadOnly.remove ();
currentTransactionIsolationLevel.remove ();
actualTransactionActive.remove ();
}
// 如果是新事务的话，进行数据清除，线程的私有资源解绑，重置连接自动提交，隔离级别，是否只
读，释放连接等
doCleanupAfterCompletion (status.getTransaction ());
```
```
// 此方法做清除连接相关操作，比如重置自动提交啊，只读属性啊，解绑数据源啊，释放连接啊，清
除链接持有器属性
protected void doCleanupAfterCompletion (Object transaction) {
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
```
```
13
14
15
```
```
16
```

这就是我们提交事务的操作了，总之来说，主要就是调用 JDBC 的 commit 提交和清除一系列的线程内
部数据和配置

**回滚事务源码分析**

回滚事务，简单来说调用 JDBC 的 rollback 和清除数据

```
DataSourceTransactionObject txObject = (DataSourceTransactionObject)
transaction;
// 将数据库连接从当前线程中解除绑定
TransactionSynchronizationManager.unbindResource (obtainDataSource ());
```
```
// 释放连接
Connection con = txObject.getConnectionHolder (). getConnection ();
```
```
// 恢复数据库连接的自动提交属性
con.setAutoCommit (true);
// 重置数据库连接
DataSourceUtils.resetConnectionAfterTransaction (con,
txObject.getPreviousIsolationLevel (), txObject.isReadOnly ());
```
```
// 如果当前事务是独立的新创建的事务则在事务完成时释放数据库连接
DataSourceUtils.releaseConnection (con, this. dataSource);
```
```
// 连接持有器属性清除
txObject.getConnectionHolder (). clear ();
}
```
```
17
```
```
18
19
20
21
22
23
24
25
26
27
```
```
28
29
30
31
32
33
34
35
```
```
public final void rollback (TransactionStatus status) throws
TransactionException {
DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
processRollback (defStatus, false);
}
```
```
private void processRollback (DefaultTransactionStatus status, boolean
unexpected) {
// 回滚的擦欧洲哦
doRollback (status);
```
```
// 回滚完成后回调
triggerAfterCompletion (status,
TransactionSynchronization. STATUS_ROLLED_BACK);
```
```
// 根据事务状态信息，完成后数据清除，和线程的私有资源解绑，重置连接自动提交，隔离级
别，是否只读，释放连接，恢复挂起事务等
cleanupAfterCompletion (status);
}
```
```
protected void doRollback (DefaultTransactionStatus status) {
DataSourceTransactionObject txObject =
(DataSourceTransactionObject) status.getTransaction ();
Connection con = txObject.getConnectionHolder (). getConnection ();
// jdbc 的回滚
con.rollback ();
}
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
```
```
12
13
```
```
14
15
16
17
18
```
```
19
20
21
22
```

#### 尼恩总结

很多的小伙伴，当时对 Spring 懵懂无知，或者，看完源码，转眼间忘记了

非常痛苦

看了此文，记住尼恩梳理的 2 大模式和三大核心流程

再也不会忘记 Spring 事务的核心源码了。

通过这篇顶奢好文，99.99% 的人应该都可以理解了 Spring 事务的来龙去脉.

如果做不到，可以看 10 遍，一定就可以做到了。

## 专题 6 ：Spring MVC 面试题专题部分

###### 什么是 Spring MVC？简单介绍下你对 Spring MVC 的理解？

Spring MVC 是一个基于 Java 的实现了 MVC 设计模式的请求驱动类型的轻量级 Web 框架，通过把模型-视
图-控制器分离，将 web 层进行职责解耦，把复杂的 web 应用分成逻辑清晰的几部分，简化开发，减少出
错，方便组内开发人员之间的配合。

###### Spring MVC 的优点

（ 1 ）可以支持各种视图技术, 而不仅仅局限于 JSP；

（ 2 ）与 Spring 框架集成（如 IoC 容器、AOP 等）；

（ 3 ）清晰的角色分配：前端控制器 (dispatcherServlet) , 请求到处理器映射（handlerMapping), 处理
器适配器（HandlerAdapter), 视图解析器（ViewResolver）。

（ 4 ） 支持各种请求资源的映射策略。

#### 核心组件

###### Spring MVC 的主要组件？

（ 1 ）前端控制器 DispatcherServlet（不需要程序员开发）

作用：接收请求、响应结果，相当于转发器，有了 DispatcherServlet 就减少了其它组件之间的耦合
度。

（ 2 ）处理器映射器 HandlerMapping（不需要程序员开发）

作用：根据请求的 URL 来查找 Handler

（ 3 ）处理器适配器 HandlerAdapter


注意：在编写 Handler 的时候要按照 HandlerAdapter 要求的规则去编写，这样适配器 HandlerAdapter
才可以正确的去执行 Handler。

（ 4 ）处理器 Handler（需要程序员开发）

（ 5 ）视图解析器 ViewResolver（不需要程序员开发）

作用：进行视图的解析，根据视图逻辑名解析成真正的视图（view）

（ 6 ）视图 View（需要程序员开发 jsp）

View 是一个接口，它的实现类支持不同的视图类型（jsp，freemarker，pdf 等等）

###### 什么是 DispatcherServlet

Spring 的 MVC 框架是围绕 DispatcherServlet 来设计的，它用来处理所有的 HTTP 请求和响应。

###### 什么是 Spring MVC 框架的控制器？

控制器提供一个访问应用程序的行为，此行为通常通过服务接口实现。控制器解析用户输入并将其转换
为一个由视图呈现给用户的模型。Spring 用一个非常抽象的方式实现了一个控制层，允许用户创建多种
用途的控制器。

###### Spring MVC 的控制器是不是单例模式, 如果是, 有什么问题, 怎么解

###### 决？

答：是单例模式, 所以在多线程访问的时候有线程安全问题, 不要用同步, 会影响性能的, 解决方案是在控制
器里面不能写字段。

#### 工作原理

###### 请描述 Spring MVC 的工作流程？描述一下 DispatcherServlet 的工

###### 作流程？

（ 1 ）用户发送请求至前端控制器 DispatcherServlet；
（ 2 ） DispatcherServlet 收到请求后，调用 HandlerMapping 处理器映射器，请求获取 Handle；
（ 3 ）处理器映射器根据请求 url 找到具体的处理器，生成处理器对象及处理器拦截器 (如果有则生成) 一
并返回给 DispatcherServlet；
（ 4 ）DispatcherServlet 调用 HandlerAdapter 处理器适配器；
（ 5 ）HandlerAdapter 经过适配调用具体处理器 (Handler，也叫后端控制器)；
（ 6 ）Handler 执行完成返回 ModelAndView；
（ 7 ）HandlerAdapter 将 Handler 执行结果 ModelAndView 返回给 DispatcherServlet；
（ 8 ）DispatcherServlet 将 ModelAndView 传给 ViewResolver 视图解析器进行解析；
（ 9 ）ViewResolver 解析后返回具体 View；
（ 10 ）DispatcherServlet 对 View 进行渲染视图（即将模型数据填充至视图中）
（ 11 ）DispatcherServlet 响应用户。


#### MVC 框架

###### MVC 是什么？MVC 设计模式的好处有哪些

mvc 是一种设计模式（设计模式就是日常开发中编写代码的一种好的方法和经验的总结）。模型
（model）-视图（view）-控制器（controller），三层架构的设计模式。用于实现前端页面的展现与后
端业务数据处理的分离。

mvc 设计模式的好处

1. 分层设计，实现了业务系统各个组件之间的解耦，有利于业务系统的可扩展性，可维护性。

2. 有利于系统的并行开发，提升开发效率。

#### 常用注解

###### 注解原理是什么

注解本质是一个继承了 Annotation 的特殊接口，其具体实现类是 Java 运行时生成的动态代理类。我们通
过反射获取注解时，返回的是 Java 运行时生成的动态代理对象。通过代理对象调用自定义注解的方法，
会最终调用 AnnotationInvocationHandler 的 invoke 方法。该方法会从 memberValues 这个 Map 中索引
出对应的值。而 memberValues 的来源是 Java 常量池。

###### Spring MVC 常用的注解有哪些？

@RequestMapping：用于处理请求 url 映射的注解，可用于类或方法上。用于类上，则表示类中的所
有响应请求的方法都是以该地址作为父路径。

@RequestBody：注解实现接收 http 请求的 json 数据，将 json 转换为 java 对象。

@ResponseBody：注解实现将 conreoller 方法返回对象转化为 json 对象响应给客户。

###### SpingMvc 中的控制器的注解一般用哪个, 有没有别的注解可以替代？

答：一般用@Controller 注解, 也可以使用@RestController,@RestController 注解相当于
@ResponseBody ＋ @Controller, 表示是表现层, 除此之外，一般不用别的注解代替。

###### @Controller 注解的作用


在 Spring MVC 中，控制器 Controller 负责处理由 DispatcherServlet 分发的请求，它把用户请求的数据
经过业务处理层处理之后封装成一个 Model ，然后再把该 Model 返回给对应的 View 进行展示。在
Spring MVC 中提供了一个非常简便的定义 Controller 的方法，你无需继承特定的类或实现特定的接
口，只需使用@Controller 标记一个类是 Controller ，然后使用@RequestMapping 和
@RequestParam 等一些注解用以定义 URL 请求和 Controller 方法之间的映射，这样的 Controller 就能
被外界访问到。此外 Controller 不会直接依赖于 HttpServletRequest 和 HttpServletResponse 等
HttpServlet 对象，它们可以通过 Controller 的方法参数灵活的获取到。

@Controller 用于标记在一个类上，使用它标记的类就是一个 Spring MVC Controller 对象。分发处理
器将会扫描使用了该注解的类的方法，并检测该方法是否使用了@RequestMapping 注解。
@Controller 只是定义了一个控制器类，而使用@RequestMapping 注解的方法才是真正处理请求的处
理器。单单使用@Controller 标记在一个类上还不能真正意义上的说它就是 Spring MVC 的一个控制器
类，因为这个时候 Spring 还不认识它。那么要如何做 Spring 才能认识它呢？这个时候就需要我们把这
个控制器类交给 Spring 来管理。有两种方式：

```
在 Spring MVC 的配置文件中定义 MyController 的 bean 对象。
在 Spring MVC 的配置文件中告诉 Spring 该到哪里去找标记为@Controller 的 Controller 控制器。
```
###### @RequestMapping 注解的作用

RequestMapping 是一个用来处理请求地址映射的注解，可用于类或方法上。用于类上，表示类中的所
有响应请求的方法都是以该地址作为父路径。

RequestMapping 注解有六个属性，下面我们把她分成三类进行说明（下面有相应示例）。

**value， method**

value： 指定请求的实际地址，指定的地址可以是 URI Template 模式（后面将会说明）；

method： 指定请求的 method 类型， GET、POST、PUT、DELETE 等；

**consumes，produces**

consumes： 指定处理请求的提交内容类型（Content-Type），例如 application/json, text/html;

produces: 指定返回的内容类型，仅当 request 请求头中的 (Accept) 类型中包含该指定类型才返回；

**params，headers**

params： 指定 request 中必须包含某些参数值是，才让该方法处理。

headers： 指定 request 中必须包含某些指定的 header 值，才能让该方法处理请求。

###### @ResponseBody 注解的作用

作用： 该注解用于将 Controller 的方法返回的对象，通过适当的 HttpMessageConverter 转换为指定格
式后，写入到 Response 对象的 body 数据区。

使用时机：返回的数据不是 html 标签的页面，而是其他某种格式的数据时（如 json、xml 等）使用；

###### @PathVariable 和@RequestParam 的区别

请求路径上有个 id 的变量值，可以通过@PathVariable 来获取 @RequestMapping (value = “/page/{id}”,
method = RequestMethod. GET)

@RequestParam 用来获得静态的 URL 请求入参 spring 注解时 action 里用到。

#### 其他


###### Spring MVC 与 Struts 2 区别

相同点

都是基于 mvc 的表现层框架，都用于 web 项目的开发。

不同点

1. 前端控制器不一样。Spring MVC 的前端控制器是 servlet：DispatcherServlet。struts 2 的前端控制器
是 filter：StrutsPreparedAndExcutorFilter。

2. 请求参数的接收方式不一样。Spring MVC 是使用方法的形参接收请求的参数，基于方法的开发，线
程安全，可以设计为单例或者多例的开发，推荐使用单例模式的开发（执行效率更高），默认就是单例
开发模式。struts 2 是通过类的成员变量接收请求的参数，是基于类的开发，线程不安全，只能设计为多
例的开发。

3. Struts 采用值栈存储请求和响应的数据，通过 OGNL 存取数据，Spring MVC 通过参数解析器是将
request 请求内容解析，并给方法形参赋值，将数据和视图封装成 ModelAndView 对象，最后又将
ModelAndView 中的模型数据通过 reques 域传输到页面。Jsp 视图解析器默认使用 jstl。

4. 与 spring 整合不一样。Spring MVC 是 spring 框架的一部分，不需要整合。在企业项目中，Spring
MVC 使用更多一些。

###### Spring MVC 怎么样设定重定向和转发的？

（ 1 ）转发：在返回值前面加"forward: "，譬如"forward: user. do? name=method 4"

（ 2 ）重定向：在返回值前面加"redirect: "，譬如"redirect:http://www.baidu.com"

###### Spring MVC 怎么和 AJAX 相互调用的？

通过 Jackson 框架就可以把 Java 里面的对象直接转化成 Js 可以识别的 Json 对象。具体步骤如下 ：

（ 1 ）加入 Jackson. jar

（ 2 ）在配置文件中配置 json 的映射

（ 3 ）在接受 Ajax 方法里面可以直接返回 Object, List 等, 但方法前面要加上@ResponseBody 注解。

###### 如何解决 POST 请求中文乱码问题，GET 的又如何处理呢？

（ 1 ）解决 post 请求乱码问题：

在 web. xml 中配置一个 CharacterEncodingFilter 过滤器，设置成 utf-8；

```
<filter>
<filter-name>CharacterEncodingFilter</filter-name>
<filter-
class>org. springframework. web. filter. CharacterEncodingFilter</filter-class>
```
```
<init-param>
<param-name>encoding</param-name>
<param-value>utf-8</param-value>
</init-param>
</filter>
```
```
<filter-mapping>
<filter-name>CharacterEncodingFilter</filter-name>
<url-pattern>/*</url-pattern>
```
```
1 2 3 4 5 6 7 8 9
```
```
10
11
12
13
```

（ 2 ）get 请求中文参数出现乱码解决方法有两个：

①修改 tomcat 配置文件添加编码与工程编码一致，如下：

②另外一种方法对参数进行重新编码：

String userName = new String (request.getParamter (“userName”). getBytes (“ISO 8859-1”),“utf-8”)

ISO 8859-1 是 tomcat 默认编码，需要将 tomcat 编码后的内容按 utf-8 编码。

###### Spring MVC 的异常处理？

答：可以将异常抛给 Spring 框架，由 Spring 框架来处理；我们只需要配置简单的异常处理器，在异常处
理器中添视图页面即可。

###### 如果在拦截请求中，我想拦截 get 方式提交的方法, 怎么配置

答：可以在@RequestMapping 注解里面加上 method=RequestMethod. GET。

###### 怎样在方法里面得到 Request, 或者 Session？

答：直接在方法的形参中声明 request, Spring MVC 就自动把 request 对象传入。

###### 如果想在拦截的方法里面得到从前台传入的参数, 怎么得到？

答：直接在形参里面声明这个参数就可以, 但必须名字和传过来的参数一样。

###### 如果前台有很多个参数传入, 并且这些参数都是一个对象的, 那么怎么

###### 样快速得到这个对象？

答：直接在方法中声明这个对象, Spring MVC 就自动会把属性赋值到这个对象里面。

###### Spring MVC 中函数的返回值是什么？

答：返回值可以有很多类型, 有 String, ModelAndView。ModelAndView 类把视图和数据都合并的一起
的，但一般用 String 比较好。

###### Spring MVC 用什么对象从后台向前台传递数据的？

答：通过 ModelMap 对象, 可以在这个对象里面调用 put 方法, 把对象加到里面, 前台就可以通过 el 表达式拿
到。

###### 怎么样把 ModelMap 里面的数据放入 Session 里面？

答：可以在类上面加上@SessionAttributes 注解, 里面包含的字符串就是要放入 session 里面的 key。

###### Spring MVC 里面拦截器是怎么写的

有两种写法, 一种是实现 HandlerInterceptor 接口，另外一种是继承适配器类，接着在接口方法当中，实
现处理逻辑；然后在 Spring MVC 的配置文件中配置拦截器即可：

```
14 </filter-mapping>
```
```
<ConnectorURIEncoding="utf-8" connectionTimeout="20000" port="8080"
protocol="HTTP/1.1" redirectPort="8443"/>
```
```
1
```

###### 介绍一下 WebApplicationContext

WebApplicationContext 继承了 ApplicationContext 并增加了一些 WEB 应用必备的特有功能，它不同
于一般的 ApplicationContext ，因为它能处理主题，并找到被关联的 servlet。

## 专题 7 ：Tomcat 专题部分

#### Tomcat 是什么？

Tomcat 服务器 Apache 软件基金会项目中的一个核心项目，是一个免费的开放源代码的 Web 应用服务
器，属于轻量级应用服务器，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调
试 JSP 程序的首选。

#### Tomcat 的缺省端口是多少，怎么修改

```
1. 找到 Tomcat 目录下的 conf 文件夹
2. 进入 conf 文件夹里面找到 server. xml 文件
3. 打开 server. xml 文件
4. 在 server. xml 文件里面找到下列信息
5. 把 Connector 标签的 8080 端口改成你想要的端口
```
#### tomcat 有哪几种 Connector 运行模式 (优化)？

下面，我们先大致了解 Tomcat Connector 的三种运行模式。

```
BIO：同步并阻塞一个线程处理一个请求。缺点：并发量高时，线程数较多，浪费资源。
Tomcat 7 或以下，在 Linux 系统中默认使用这种方式。
```
**配制项** ：protocol=”HTTP/1.1”

```
<!-- 配置Spring MVC的拦截器 -->
<mvc:interceptors>
<!-- 配置一个拦截器的Bean就可以了 默认是对所有请求都拦截 -->
<bean id="myInterceptor" class="com.zwp.action.MyHandlerInterceptor">
</bean>
<!-- 只针对部分请求拦截 -->
<mvc:interceptor>
<mvc:mapping path="/modelMap.do" />
<bean class="com.zwp.action.MyHandlerInterceptorAdapter" />
</mvc:interceptor>
</mvc:interceptors>
```
```
1 2 3 4 5 6 7 8 9
```
```
10
```
```
<Service name="Catalina">
<Connector port="8080" protocol="HTTP/1.1"
connectionTimeout="20000"
redirectPort="8443" />
```
```
1
2
3
4
```

```
NIO：同步非阻塞 IO
利用 Java 的异步 IO 处理，可以通过少量的线程处理大量的请求，可以复用同一个线程处理多个
connection (多路复用)。
Tomcat 8 在 Linux 系统中默认使用这种方式。
Tomcat 7 必须修改 Connector 配置来启动。
配制项 ：protocol=”org. apache. coyote. http 11. Http 11 NioProtocol”
备注 ：我们常用的 Jetty，Mina，ZooKeeper 等都是基于 java nio 实现.
APR：即 Apache Portable Runtime，从操作系统层面解决 io 阻塞问题。 AIO 方式，* *异步非阻塞
IO**(Java NIO 2 又叫 AIO) 主要与 NIO 的区别主要是操作系统的底层区别. 可以做个比喻: 比作快递，
NIO 就是网购后要自己到官网查下快递是否已经到了 (可能是多次)，然后自己去取快递；AIO 就是
快递员送货上门了 (不用关注快递进度)。
配制项 ：protocol=”org. apache. coyote. http 11. Http 11 AprProtocol”
备注 ：需在本地服务器安装 APR 库。Tomcat 7 或 Tomcat 8 在 Win 7 或以上的系统中启动默认使用这
种方式。Linux 如果安装了 apr 和 native，Tomcat 直接启动就支持 apr。
```
#### Tomcat 有几种部署方式？

**在 Tomcat 中部署 Web 应用的方式主要有如下几种：**

```
1. 利用 Tomcat 的自动部署。
把 web 应用拷贝到 webapps 目录。Tomcat 在启动时会加载目录下的应用，并将编译后的结果放入
work 目录下。
2. 使用 Manager App 控制台部署。
在 tomcat 主页点击“Manager App” 进入应用管理控制台，可以指定一个 web 应用的路径或 war 文
件。
3. 修改 conf/server. xml 文件部署。
修改 conf/server. xml 文件，增加 Context 节点可以部署应用。
4. 增加自定义的 Web 部署文件。
在 conf/Catalina/localhost/ 路径下增加 xyz. xml 文件，内容是 Context 节点，可以部署应用。
```
#### tomcat 容器是如何创建 servlet 类实例？用到了什么原

#### 理？

```
1. 当容器启动时，会读取在 webapps 目录下所有的 web 应用中的 web. xml 文件，然后对 xml 文件进
行解析，并读取 servlet 注册信息。然后，将每个应用中注册的 servlet 类都进行加载，并通过反射
的方式实例化。（有时候也是在第一次请求时实例化）
2. 在 servlet 注册时加上 1 如果为正数，则在一开始就实例化，如果不写或为负数，则第一次请求实例
化。
```
#### Tomcat 工作模式

Tomcat 作为 servlet 容器，有三种工作模式：

```
1 、独立的 servlet 容器，servlet 容器是 web 服务器的一部分；
2 、进程内的 servlet 容器，servlet 容器是作为 web 服务器的插件和 java 容器的实现，web 服务器插
件在内部地址空间打开一个 jvm 使得 java 容器在内部得以运行。反应速度快但伸缩性不足；
```

```
3 、进程外的 servlet 容器，servlet 容器运行于 web 服务器之外的地址空间，并作为 web 服务器的插
件和 java 容器实现的结合。反应时间不如进程内但伸缩性和稳定性比进程内优；
```
进入 Tomcat 的请求可以根据 Tomcat 的工作模式分为如下两类：

```
Tomcat 作为应用程序服务器：请求来自于前端的 web 服务器，这可能是 Apache, IIS, Nginx 等；
Tomcat 作为独立服务器：请求来自于 web 浏览器；
```
面试时问到 Tomcat 相关问题的几率并不高，正式因为如此，很多人忽略了对 Tomcat 相关技能的掌握，
下面这一篇文章整理了 Tomcat 相关的系统架构，介绍了 Server、Service、Connector、Container 之间
的关系，各个模块的功能，可以说把这几个掌握住了，Tomcat 相关的面试题你就不会有任何问题了！
另外，在面试的时候你还要有意识无意识的往 Tomcat 这个地方引，就比如说常见的 Spring MVC 的执行
流程，一个 URL 的完整调用链路，这些相关的题目你是可以往 Tomcat 处理请求的这个过程去说的！掌握
了 Tomcat 这些技能，面试官一定会佩服你的！

学了本章之后你应该明白的是：

```
Server、Service、Connector、Container 四大组件之间的关系和联系，以及他们的主要功能点；
Tomcat 执行的整体架构，请求是如何被一步步处理的；
Engine、Host、Context、Wrapper 相关的概念关系；
Container 是如何处理请求的；
Tomcat 用到的相关设计模式；
```
#### Tomcat 顶层架构

俗话说，站在巨人的肩膀上看世界，一般学习的时候也是先总览一下整体，然后逐个部分个个击破，最
后形成思路，了解具体细节，Tomcat 的结构很复杂，但是 Tomcat 非常的模块化，找到了 Tomcat 最
核心的模块，问题才可以游刃而解，了解了 Tomcat 的整体架构对以后深入了解 Tomcat 来说至关重
要！

先上一张 Tomcat 的顶层结构图（图 A），如下：


Tomcat 中最顶层的容器是 Server，代表着整个服务器，从上图中可以看出，一个 Server 可以包含至少
一个 Service，即可以包含多个 Service，用于具体提供服务。

Service 主要包含两个部分：Connector 和 Container。从上图中可以看出 Tomcat 的心脏就是这两个组
件，他们的作用如下：

```
Connector 用于处理连接相关的事情，并提供 Socket 与 Request 请求和 Response 响应相关的转化;
Container 用于封装和管理 Servlet，以及具体处理 Request 请求；
```
一个 Tomcat 中只有一个 Server，一个 Server 可以包含多个 Service，一个 Service 只有一个 Container，
但是可以有多个 Connectors，这是因为一个服务可以有多个连接，如同时提供 Http 和 Https 链接，也可
以提供向相同协议不同端口的连接，示意图如下（Engine、Host、Context 下面会说到）：

多个 Connector 和一个 Container 就形成了一个 Service，有了 Service 就可以对外提供服务了，但是
Service 还要一个生存的环境，必须要有人能够给她生命、掌握其生死大权，那就非 Server 莫属了！所
以整个 Tomcat 的生命周期由 Server 控制。

另外，上述的包含关系或者说是父子关系，都可以在 tomcat 的 conf 目录下的 server. xml 配置文件中看
出，下图是删除了注释内容之后的一个完整的 server. xml 配置文件（Tomcat 版本为 8.0）


详细的配置文件内容可以到 Tomcat 官网查看：Tomcat 配置文件

上边的配置文件，还可以通过下边的一张结构图更清楚的理解：

Server 标签设置的端口号为 8005 ，shutdown=”SHUTDOWN” ，表示在 8005 端口监听“SHUTDOWN”命
令，如果接收到了就会关闭 Tomcat。一个 Server 有一个 Service，当然还可以进行配置，一个 Service 有
多个 Connector，Service 左边的内容都属于 Container 的，Service 下边是 Connector。

###### Tomcat 顶层架构小结

```
1. Tomcat 中只有一个 Server，一个 Server 可以有多个 Service，一个 Service 可以有多个 Connector 和
一个 Container；
2. Server 掌管着整个 Tomcat 的生死大权；
3. Service 是对外提供服务的；
4. Connector 用于接受请求并将请求封装成 Request 和 Response 来具体处理；
5. Container 用于封装和管理 Servlet，以及具体处理 request 请求；
```

知道了整个 Tomcat 顶层的分层架构和各个组件之间的关系以及作用，对于绝大多数的开发人员来说
Server 和 Service 对我们来说确实很远，而我们开发中绝大部分进行配置的内容是属于 Connector 和
Container 的，所以接下来介绍一下 Connector 和 Container。

#### Connector 和 Container 的微妙关系

由上述内容我们大致可以知道一个请求发送到 Tomcat 之后，首先经过 Service 然后会交给我们的
Connector，Connector 用于接收请求并将接收的请求封装为 Request 和 Response 来具体处理，
Request 和 Response 封装完之后再交由 Container 进行处理，Container 处理完请求之后再返回给
Connector，最后在由 Connector 通过 Socket 将处理的结果返回给客户端，这样整个请求的就处理完
了！

Connector 最底层使用的是 Socket 来进行连接的，Request 和 Response 是按照 HTTP 协议来封装的，所
以 Connector 同时需要实现 TCP/IP 协议和 HTTP 协议！

Tomcat 既然需要处理请求，那么肯定需要先接收到这个请求，接收请求这个东西我们首先就需要看一
下 Connector！

Connector 架构分析

Connector 用于接受请求并将请求封装成 Request 和 Response，然后交给 Container 进行处理，
Container 处理完之后在交给 Connector 返回给客户端。

因此，我们可以把 Connector 分为四个方面进行理解：

```
1. Connector 如何接受请求的？
2. 如何将请求封装成 Request 和 Response 的？
3. 封装完之后的 Request 和 Response 如何交给 Container 进行处理的？
4. Container 处理完之后如何交给 Connector 并返回给客户端的？
```
首先看一下 Connector 的结构图（图 B），如下所示：

Connector 就是使用 ProtocolHandler 来处理请求的，不同的 ProtocolHandler 代表不同的连接类型，比
如：Http 11 Protocol 使用的是普通 Socket 来连接的，Http 11 NioProtocol 使用的是 NioSocket 来连接
的。

其中 ProtocolHandler 由包含了三个部件：Endpoint、Processor、Adapter。

```
1. Endpoint 用来处理底层 Socket 的网络连接，Processor 用于将 Endpoint 接收到的 Socket 封装成
Request，Adapter 用于将 Request 交给 Container 进行具体的处理。
2. Endpoint 由于是处理底层的 Socket 网络连接，因此 Endpoint 是用来实现 TCP/IP 协议的，而
Processor 用来实现 HTTP 协议的，Adapter 将请求适配到 Servlet 容器进行具体的处理。
```

```
3. Endpoint 的抽象实现 AbstractEndpoint 里面定义的 Acceptor 和 AsyncTimeout 两个内部类和一个
Handler 接口。Acceptor 用于监听请求，AsyncTimeout 用于检查异步 Request 的超时，Handler 用
于处理接收到的 Socket，在内部调用 Processor 进行处理。
```
至此，我们应该很轻松的回答 1 ， 2 ， 3 的问题了，但是 4 还是不知道，那么我们就来看一下 Container 是
如何进行处理的以及处理完之后是如何将处理完的结果返回给 Connector 的？

#### Container 架构分析

Container 用于封装和管理 Servlet，以及具体处理 Request 请求，在 Container 内部包含了 4 个子容器，
结构图如下（图 C）：

4 个子容器的作用分别是：

```
1. Engine：引擎，用来管理多个站点，一个 Service 最多只能有一个 Engine；
2. Host：代表一个站点，也可以叫虚拟主机，通过配置 Host 就可以添加站点；
3. Context：代表一个应用程序，对应着平时开发的一套程序，或者一个 WEB-INF 目录以及下面的
web. xml 文件；
4. Wrapper：每一 Wrapper 封装着一个 Servlet；
```
下面找一个 Tomcat 的文件目录对照一下，如下图所示：


Context 和 Host 的区别是 Context 表示一个应用，我们的 Tomcat 中默认的配置下 webapps 下的每一个文
件夹目录都是一个 Context，其中 ROOT 目录中存放着主应用，其他目录存放着子应用，而整个
webapps 就是一个 Host 站点。

我们访问应用 Context 的时候，如果是 ROOT 下的则直接使用域名就可以访问，例如：
[http://www.baidu.com，如果是Host（webapps）下的其他应用，则可以使用www.baidu.com/docs进行访](http://www.baidu.com，如果是Host（webapps）下的其他应用，则可以使用www.baidu.com/docs进行访)
问，当然默认指定的根应用（ROOT）是可以进行设定的，只不过 Host 站点下默认的主应用是 ROOT 目
录下的。

看到这里我们知道 Container 是什么，但是还是不知道 Container 是如何进行请求处理的以及处理完之后
是如何将处理完的结果返回给 Connector 的？别急！下边就开始探讨一下 Container 是如何进行处理
的！

###### Container 如何处理请求的

Container 处理请求是使用 Pipeline-Valve 管道来处理的！（Valve 是阀门之意）

Pipeline-Valve 是 **责任链模式** ，责任链模式是指在一个请求处理的过程中有很多处理者依次对请求进行
处理，每个处理者负责做自己相应的处理，处理完之后将处理后的结果返回，再让下一个处理者继续处
理。

但是！Pipeline-Valve 使用的责任链模式和普通的责任链模式有些不同！区别主要有以下两点：


```
每个 Pipeline 都有特定的 Valve，而且是在管道的最后一个执行，这个 Valve 叫做 BaseValve，
BaseValve 是不可删除的；
在上层容器的管道的 BaseValve 中会调用下层容器的管道。
```
我们知道 Container 包含四个子容器，而这四个子容器对应的 BaseValve 分别在：
StandardEngineValve、StandardHostValve、StandardContextValve、StandardWrapperValve。

Pipeline 的处理流程图如下（图 D）：

```
Connector 在接收到请求后会首先调用最顶层容器的 Pipeline 来处理，这里的最顶层容器的
Pipeline 就是 EnginePipeline（Engine 的管道）；
在 Engine 的管道中依次会执行 EngineValve 1、EngineValve 2 等等，最后会执行
StandardEngineValve，在 StandardEngineValve 中会调用 Host 管道，然后再依次执行 Host 的
HostValve 1、HostValve 2 等，最后在执行 StandardHostValve，然后再依次调用 Context 的管道和
Wrapper 的管道，最后执行到 StandardWrapperValve。
当执行到 StandardWrapperValve 的时候，会在 StandardWrapperValve 中创建 FilterChain，并调
用其 doFilter 方法来处理请求，这个 FilterChain 包含着我们配置的与请求相匹配的 Filter 和
Servlet，其 doFilter 方法会依次调用所有的 Filter 的 doFilter 方法和 Servlet 的 service 方法，这样请
求就得到了处理！
当所有的 Pipeline-Valve 都执行完之后，并且处理完了具体的请求，这个时候就可以将返回的结果
交给 Connector 了，Connector 在通过 Socket 的方式将结果返回给客户端。
```
#### 参考文献：

https://www.cnblogs.com/leeego-123/p/12159574.html


https://www.jianshu.com/p/1dec08d290c1

https://blog.csdn.net/weixin_40006977/article/details/112711947


```
技术自由圈
```
## 未来职业，如何突围：三栖架构师


```
技术自由圈
```
### 成功案例： 2 年翻 3 倍， 35 岁卷王成功转型为架构师

详情：http://topcoder.cloud/forum.php?mod=forumdisplay&fid=43&page=1


技术自由圈


技术自由圈


技术自由圈


```
技术自由圈
```
### 硬核推荐：尼恩 Java 硬核架构班

详情：https://www.cnblogs.com/crazymakercircle/p/9904544.html


技术自由圈


```
技术自由圈
```
##### 架构班（社群 VIP）的起源：

最初的视频，主要是给读者加餐。很多的读者，需要一些高质量的实操、理论视频，所以，我就围绕书，和底层，做了几个
实操、理论视频，然后效果还不错，后面就做成迭代模式了。

##### 架构班（社群 VIP）的功能：^

提供高质量实操项目整刀真枪的架构指导、快速提升大家的:

⚫ 开发水平
⚫ 设计水平

⚫ 架构水平

弥补业务中 CRUD 开发短板，帮助大家尽早脱离具备 3 高能力，掌握：
⚫ 高性能

⚫ 高并发
⚫ 高可用

作为一个高质量的架构师成长、人脉社群，把所有的卷王聚焦起来，一起卷：

⚫ 卷高并发实操
⚫ 卷底层原理

⚫ 卷架构理论、架构哲学
⚫ 最终成为顶级架构师，实现人生理想，走向人生巅峰

##### 架构班（社群 VIP）的目的：^

⚫ 高质量的实操，大大提升简历的含金量，吸引力，增强面试的召唤率

⚫ 为大家提供九阳真经、葵花宝典，快速提升水平
⚫ 进大厂、拿高薪

⚫ 一路陪伴，提供助学视频和指导，辅导大家成为架构师
⚫ 自学为主，和其他卷王一起，卷高并发实操，卷底层原理、卷大厂面试题，争取狠卷 3 月成高手，狠卷 3 年成为顶级架

```
构师
```

```
技术自由圈
```
##### N 个超高并发实操项目：简历压轴、个顶个精彩


```
技术自由圈
```
【样章】第 17 章：横扫全网 Rocketmq 视频第 2 部曲: 工业级 rocketmq 高可用（HA）底层原
理和实操

工业级 rocketmq 高可用底层原理，包含：消息消费、同步消息、异步消息、单向消息等不同消息的底层原理和源码实现；
消息队列非常底层的主从复制、高可用、同步刷盘、异步刷盘等底层原理。

工业级 rocketmq 高可用底层原理和搭建实操，包含：高可用集群的搭建。
解决以下难题：

1 、技术难题：RocketMQ 如何最大限度的保证消息不丢失的呢？RocketMQ 消息如何做到高可靠投递？

2 、技术难题：基于消息的分布式事务，核心原理不理解
3 、选型难题： kafka or rocketmq ，该娶谁？

下图链接：https://www.processon.com/view/6178e8ae0e3e7416bde9da19


```
技术自由圈
```
### 简历优化后的成功涨薪案例（ VIP 含免费简历优化）


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


技术自由圈


```
技术自由圈
```
### 修改简历找尼恩（资深简历优化专家）

⚫ 如果面试表达不好，尼恩会提供简历优化指导

⚫ 如果项目没有亮点，尼恩会提供项目亮点指导

⚫ 如果面试表达不好，尼恩会提供面试表达指导

作为 40 岁老架构师，尼恩长期承担技术面试官的角色：

⚫ 从业以来，“阅历”无数，对简历有着点石成金、改头换面、脱胎换骨的指导能力。

⚫ 尼恩指导过刚刚就业的小白，也指导过 P 8 级的老专家，都指导他们上岸。

如何联系尼恩。尼恩微信，请参考下面的地址：

语雀：https://www.yuque.com/crazymakercircle/gkkw8s/khigna

码云：https://gitee.com/crazymaker/SimpleCrayIM/blob/master/疯狂创客圈总目录.md


