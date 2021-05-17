# iOS开发学习报告(6)

## 《Effective Objective-C》

学习《Effective Objective-C》第41-52条，总结重点如下：

* 如果对象中存在一个属性可能会出现线程安全问题，可以在该属性的getter方法中使用**dispatch_sync**进行读操作，setter方法中使用**disptach_barrier_async**进行写操作。这样比使用锁🔐来实现同步更加高效；
* 通过GCD提供的**dispatch_once**函数编写只需要执行一次的代码（如创建单例对象的方法sharedInstance），可以**保证线程安全**，为了保证每次调用时都使用相同的标记，标记要声明在==static==或==global==作用域；
* 遍历collection最好使用"块枚举"enumerateObjectsUsingBlock，修改块签名，指出对象具体类型，可使编译器在调用错误方法时报错

## 第四期交流问题解决

dispatch_semaphore_t的信号值是否可以为负数？如果被析构时，内部信号值和初始化的值不一致会遇到什么问题？如果信号量析构时，有地方还在wait会发生什么？

* dispatch_semaphore_t的信号值必须大于等于0。
* 信号量销毁时会调用dispatch_semaphore_dispose函数，而此函数会执行信号当前值与初始值的比较，如果小于初始值，则直接抛出崩溃，即：signal的调用次数一定要大于等于wait的调用次数，否则就会导致崩溃。



## 《Header First设计模式》

#### 1 策略模式

其中重点描述了几个设计原则：

* 找出应用中可能需要变化的地方，独立出来；
* 针对接口编程，而不是针对实现编程：可以编写一组其他类专门用于实现对象的一些行为；
* 多用组合，少用继承：组合具有更大弹性，

策略模式：定义了**算法族**（不同行为），分别封装起来，互相之间可以替换。

根据书中的例子做了有关策略模式的应用练习：https://github.com/skyjasmine/ZLDuck/tree/zl

#### 2 观察者模式

观察者模式：定义了对象之前的一对多依赖关系，当被观察对象状态改变时，所有依赖者都会收到通知并自动更新。OC中的**NSNotificationCenter**基于这种模式。

遵守的设计原则：

* 交互对象之间松耦合，不需要知道对方的实现细节；

#### 3 装饰者模式

定义：动态地将责任附加到对象上。要扩展功能，装饰者模式比继承更具有弹性。

⚠️装饰者与被装饰者必须是**一样的类型**，即有同一个超类/遵循同一个接口。

设计原则：

* **开放-关闭原则**：类应该对扩展开放，对修改关闭。当然并不是每个地方都遵循这个原则，而是在很有可能改变/扩展的地方，应用该原则。

装饰者模式应用练习：https://github.com/skyjasmine/ZLDecorator/tree/zl

#### 4 工厂模式

定义：定义一个**创建对象**的接口（抽象基类），工厂方法把类的实例化推迟到具体==子类==决定。将创建对象的代码封装起来，实例化对象时，只需依赖接口，而非具体类。通过**继承**的方式实现。

设计原则：

* 依赖倒置原则：要**依赖抽象**，不要依赖具体类。应用工厂模式，高层组件和低层组件会依赖相同的抽象。

如何避免违反依赖倒置原则：

* 变量不可以持有具体类的引用（不要new）；
* 不要让类派生自具体类（类要派生自抽象：接口或抽象类）；
* 不要覆盖基类中已经实现的方法（所有子类要共享该方法）。

抽象工厂模式：提供一个==接口==（protocol），用于创建相关产品，不需要明确具体类。通过**组合**形式实现。

工厂模式应用练习：https://github.com/skyjasmine/ZLFactory

#### 5 单例模式

定义：确保一个类**只有一个实例**，并且提供一个全局访问点。

通过GCD提供的**dispatch_once**函数编写创建单例对象的方法，可以**保证线程安全**，为了保证每次调用时都使用相同的标记，标记要声明在==static==或==global==作用域.

#### 6命令模式

定义：将请求封装成对象，以便使用不同的请求、队列或者日志来**参数化**其他对象（将请求command作为参数传入control对象中）。该模式 也支持可撤销的操作。

**发出**请求的对象和**执行**请求的对象间，通过**命令对象**进行沟通。

命令模式应用练习：https://github.com/skyjasmine/ZLCommand

#### 7 适配器模式与外观模式

这两种模式的目的都是改变接口，方便客户端调用。

（1）OO适配器：客户---->适配器---->被适配者

适配器模式：将一个类的接口，转换成客户期望的另一个接口。适配器让原本接口不兼容的类可以产生联系。

（2）外观模式：提供一个统一的接口，用来访问子系统中的一群接口。外观定义了一个高层接口，让子系统更容易使用。

设计原则：

* 最少知识原则：减少对象之间的交互

#### 8 模块方法模式

定义：在一个方法中定义一个算法（一组步骤）的骨架，而将一些步骤延迟到子类中。使得子类可以在不改变算法结构的情况下，重新定义算法的某些步骤。

设计原则：

* 好莱坞原则，允许低层组件将自己挂钩到系统上，但是高层组件会决定什么时候和怎样使用这些底层组件。即，高层组件对待低层组件的方式“别调用我，我会调用你”。

#### 9 迭代器模式

（1）迭代器模式：提供一种方法顺序访问一个聚合对象内的各个元素，而又不暴露内部的表示。（C++STL库中集合都有iterator）

设计原则：

* 一个类应该只有一个引起变化的原因。即：尽量让每个类保持单一责任（高内聚）。

内聚：当一个模块或一个类被设计成只支持一组相关的功能，具有高内聚；当设计成支持一组不相关的功能时，低内聚。

（2）组合模式：将对象组合成树形结构来表现“整体/部分”层次结构。组合能让客户以一致的方式处理个别对象及对象组合。

#### 10 状态模式

定义：允许对象在内部状态改变时改变它的行为。该模式将状态封装成独立的类，并将动作委托到代表当前状态的对象。

#### 11 代理模式

定义：使用代理模式创建代表对象，让代表对象控制某对象的访问，被代理的可以使远程的对象、创建开销大的对象或者需要安全控制的对象。

#### 12 复合模式

MVC模式

Model：模型持有所有的数据、状态和逻辑

View：通常直接从模型中取得它需要显示的状态和数据，显示在UI界面上

Controller：转发请求，对请求进行处理

在Model和View之间加入Controller主要为了消除两者之间的耦合。我在搜索iOS MVC模式时，看到有些博客中的例子，将view和Controller分开了，在Controller中创建view对象，然后在view中通过委托给Controller来实现view中控件的响应。但是实际工程中似乎不会刻意地把view和Controller区分开？

#### 13 模式的使用

设计时，尽可能使用最简单的方式解决问题，只有在需要实践扩展的地方，才使用模式。过度使用设计模式可能导致代码过度工程化。

#### WWDC

* 观看了2020 WWDC，关于iOS14，印象比较深的两点：

  （1）App library ：不想显示在桌面的App可以将它移除放入App library，让桌面更加简洁，明了，想用的时候直接从App library搜索，很方便（自己体验了一下，针不戳😁）

  （2）App Clip让用户可以只使用某个App的一部分，省略了下载安装的步骤，省时快捷，而且使用后可以在App library 找到，然后下载。这样不常用的App就不用下载了，需要的时候也可以通过App Clip使用，节约内存（感觉有点像小程序的作用）

## NSURLSession

AFNetworking底层封装了NSURLSession。可以将NSURLSession类理解为七层网络协议中的会话层，用于管理网络接口的创建、维护、删除等工作，再底层的工作NSURLSession已经封装好了。

### 1.NSURLSessionConfiguration

通过NSURLSessionConfiguration对象可以对NSURLSession对象的超时时间、缓存策略、连接请求等进行配置。

### 2. NSURLSessionTask

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201107104405446.png)

* NSURLSessionDataTask：处理一般网络请求，GET|POST请求等；
* NSURLSessionUploadTask：处理上传请求
* NSURLSessionDownloadTask：处理下载请求

通过request对象或url创建task对象，可以通过代理或者block接收服务器数据。

* 代理方式

  遵守协议<NSURLSessionDelegate, NSURLSessionTaskDelegate>，NSURLSessionDelegate主要关于会话，比如会话关闭；NSURLSessionTaskDelegate主要关于任务，比如任务完成或失败。

##  AFNetworking

### 1.调试中出现的问题

#### 问题

* 使用CocoaPods安装AFNetworking并测试

  测试中出现了访问失败的情况：在访问https://www.baidu.com时，出现错误：

  ![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201024112335432.png)

  根据网上查找的两种方案：

  （1）在AFHTTPResponseSerializer实现中将acceptableContentTypes按如下方式修改，仍然没有解决问题😿。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201024114350183.png)
（2）然后我在main.m中按照下面的方式修改：

运行结果：![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201024113630611.png)

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201024113704451.png)

疑问：改变的都是AFHTTPResponseSerializer类的acceptableContentTypes属性，为什么前者不能解决问题，后者可以？

#### 解决方法

针对这个问题，我通过debug的方式观察了AFHTTPSessionManager的初始化过程，init函数中默认manager的responseSerializer是AFJSONResponseSerializer对象，而请求百度首页返回的是html网页，并非json文本。如果修改AFHTTPResponseSerializer中acceptaleContentTypes，该属性将会被AFJSONResponseSerializer的acceptaleContentTypes覆盖，依旧不能接收html。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201024113704451.png)

那如果将AFJSONResponseSerializer的acceptaleContentTypes也添加上html类型，那么则会出现如下问题：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201107192230622.png)

因为返回的根本不是JSON文本，而是html网页，所以无法按照JSON的方式解析。于是在发送请求前将manager的responseSerializer修改为AFHTTPResponseSerializer 对象，然后在该类的acceptaleContentTypes也添加上html类型，就能正确解析出html。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201107173323832.png)
![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/image-20201107192607963.png)

### 2. 底层原理

参考：https://blog.csdn.net/Forever_wj/article/details/108402416

#### 2.1 AFNetworking组成

AFNetworking中主要涉及的类有AFURLSessionManager、AFHTTPSessionManager、AFURLRequestSerialization、AFURLResponseSerialization、AFNetworkReachabilityManager、AFSecurityPolicy。

* AFURLSessionManager和AFHTTPSessionManager是父子类的关系，为网络管理模块，负责处理网络请求；
* AFURLRequestSerialization、AFURLResponseSerialization：请求和响应的序列化/反序列化模块；
* AFNetworkReachabilityManager：监控网络状态；
* AFSecurityPolicy：网络安全 。

#### 2.2 主要流程

* 创建sessionManager：[AFHTTPSessionManager manager]

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/sessionManagerInit.png)
从上面流程中可以 看出初始化manager的过程中主要进行了session设置、RequestSerializer和RequestSerializer的初始化。

* 与服务器通信： [manager GET:parameters:headers:progress:success:failure:]

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志6)/CommunicationWithServer.png)
AFHTTPSessionManager在创建GET、POST等请求时，过程基本如上图，方法内部会创建一个task对象，并调用 addDelegateForDataTask 将后面的处理交给 AFURLSessionManagerTaskDelegate 来完成。
