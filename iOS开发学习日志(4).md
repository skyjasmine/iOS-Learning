# iOS开发学习日志(4)

## 1.Protocol(协议)

### 1.1 协议的作用：

（1）用来==声明一大堆方法==（不能声明属性，也不能实现方法，只能用来写方法的声明）

（2）只要==某个类遵守了这个协议，就相当于拥有这个协议中的所有的方法声明==。这个类只是拥有了这个协议的声明而已，没有实现，所以==在类中要实现协议中的方法==。

（3）一个类只能单继承，但是==可以遵守多个协议==。当一个类遵守多个协议后，就相当于这个类拥有了所有协议中定义的方法的声明。就应该实现所有协议的方法。

语法：@interface 类名:父类名<协议>

​			@end

```objective-c
//-----------------MyProtocol.h-----------------
#import <Foundation/Foundation.h>
@protocol MyProtocol <NSObject>      //协议的声明格式
-(void)run;
-(void)sleep;
@end
  
//-------------------Dog.h---------------------
#import <Foundation/Foundation.h>
#import "MyProtocol.h"
@interface Dog : NSObject<MyProtocol>    //类可以遵守多个协议

@end
  
//--------------------Dog.m--------------------
#import "Dog.h"
@implementation Dog
-(void)run
{
    NSLog(@"🐶在跑步");
}

-(void)sleep
{
    NSLog(@"🐶在睡觉");

}
@end
  
//--------------------main.m---------------------
#import <Foundation/Foundation.h>
#import "Dog.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Dog *wangCai = [[Dog alloc] init];
        [wangCai run];
        [wangCai sleep];
    }
    return 0;
}
```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/1.png)

从上面的执行结果可以看出，Dog类遵守协议MyProtocol，并在Dog.m中实现了协议中的方法，然后在mian程序中Dog类对象就可以调用协议中的这些方法了。

### 1.2  @required与@optional

我们可以使用@required与@optional来修饰协议中的方法。

在协议中，被@required修饰的方法，遵守这个协议的类必须实现它，否则编译器发出警告。被@optional修饰的方法，遵守这个协议的类可以实现它，也可以不实现它，不实现时编译器不会发出警告。系统默认@required。

但是如果调用了协议中的方法，而这个方法没有在遵守协议的类中被实现的话，就会报错。

这两个关键字的作用在于：告诉遵守协议的类，哪些方法必须实现，因为这些方法将会被调用。

```objective-c
//----------------Sport.h-----------------
#import <Foundation/Foundation.h>

@protocol SportProtocol <NSObject>
@required
-(void)climb;

@optional
-(void)swim;
@end
```

### 1.4 NSObject协议

协议间是有继承关系的：

语法：@protocol 协议名称<父协议名称>

​			@end

子协议中不仅有自己的方法声明，还有父协议中的所有方法声明。如果某个类遵守了某个协议，那么这个类就拥有了该协议和其父协议的所有方法声明。

在Foundation框架中，有一个类NSObject，是OC所有类的基类。Foundation框架中同样存在一个协议，也叫做NSOject。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/2.png)

NSObject协议被NSObject类遵守，所以NSObject协议中的所有方法，全部的OC类都拥有了。即：所有的类都遵守的NSObject协议。==NSObject协议叫做基协议==。

类的名称可以和协议的名称一致。

写协议的规范：要求所有的协议都直接或者间接继承自NSObject协议。

### 1.5 协议的类型限制

NSObject<协议名称> *指针名称

指针可以指向遵守了指定协议的任意对象，**否则就会发出警告**。

```objective-c
//------------Student.h------------
@interface Student : NSObject<StudyProtocol>

@end
  
//-------------main.m--------------
NSObject<StudyProtocol> *obj1 = [Student new];

```

用处在于：当我要调用对象中的协议方法时，可以通过是否有警告来推断类是否遵守了协议。只有遵守了该协议，才能拥有协议方法。
更新：在对协议方法调用时（特别是optional），一般都会用**responseToSelector:**来判断对象是否真正实现了这个方法。

### 1.6 代理模式

代理模式是iOS开发中常用的一种设计模式。

举个例子解释一下：一个小宝宝几个月大了，但是妈妈要去上班没有办法照顾他，于是找了一个保姆，委托保姆来照顾👶，这就是代理模式。

代理模式的实用场景：iOS中通常使用代理做消息的传递，比如UITableView就是使用代理来创建cell，点击cell等一系列操作。

```objective-c
//--------------------Children.h------------------
#import <Cocoa/Cocoa.h>

NS_ASSUME_NONNULL_BEGIN

@class Children;

@protocol ChildrenDelegate <NSObject>

- (void)wash:(Children*)children;
- (void)play:(Children*)children;

@end

@interface Children:NSObject
//Children的代理对象delegate
@property (nonatomic,weak) id<ChildrenDelegate> delegate;    //20200702update

@end
```
**更新：一般delegate都是使用weak来修饰，以避免一些不必要的循环引用**。 
上面代码中我们定义了一个协议：ChildrenDelegate，其中含有两个必要方法：wash和play。

还定义了一个非常重要的属性delegate，因为id是不确定类型，所以__delegate可以被赋值为的类型是：**只要实现了ChildrenDelegate协议的类就可以**。格式为id<协议名>。

```objective-c
//-----------------------Children.m----------------------------
#import "Children.h"

@implementation Children

@end
  
//---------------------Nure.h-------------------
#import <Foundation/Foundation.h>
#import "Children.h"

@interface Nure : NSObject<ChildrenDelegate>

@end
  
//-----------------------Nure.m-----------------------
#import "Nure.h"

@implementation Nure
- (void)wash:(Children*)children
{
    NSLog(@"保姆帮👶洗澡");
}
- (void)play:(Children*)children
{
    NSLog(@"保姆和👶玩耍");
}
@end
```

在保姆类中遵守了ChildrenDelegate协议，并且在类中实现了wash和play方法。

```objective-c
//---------------------main.m-------------------
#import <Foundation/Foundation.h>
#import "Children.h"
#import "Nure.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Children *child = [[Children alloc]init];
        Nure *nure = [[Nure alloc]init];
        child.delegate = nure;
        
        [child.delegate wash:child];
        [child.delegate play:child];
    }
    return 0;
}
```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/3.png)

从以上代码可以看出代理模式的好处，如果此时又多了一个人照顾👶：妈妈类，那么只要让妈妈类实现ChildrenDelegate协议即可。因此实现了被委托对象和委托对象之间的解藕，具有很高的扩展性。

## 2.GCD

Grand Central Dispatch(GCD)是异步执行技术之一，程序猿们只需要**定义想执行的任务并追加到适当的Dispatch Queue**中，GCD就能生成必要的线程并计划执行任务。

应用程序在启动时，通过最先执行的线程，即“**主线程**”来描绘用户界面、处理触摸屏幕等事件。如果在主线程中进行长时间的处理，如数据库访问，就会妨碍主线程的执行（阻塞）。因此，如果将长时间的处理事件放到**其他线程**中处理，就可以保证用户界面的响应性能。

### 2.1 Dispatch Queue

**定义想执行的任务并追加到适当的Dispatch Queue**中，可用源代码表示如下：

```objective-c
dispatch_async(queue,^{
  /*
   *要执行的任务
   */
});
```

该程序使用Block语法“定义想执行的任务”，通过dispatch_async函数追加到Dispatch Queue的变量queue中，这样就可以使Block在另一线程中执行。

Dispatch Queue是执行处理的等待队列。通过dispatch_async函数等API，在Block语法中记述想执行的处理并将其追加到Dispatch Queue。Dispatch Queue按照追加的顺序（先进先出FIFO）执行处理。

执行处理时存在两种Dispatch Queue，一种是**等待**现在处理的**Serial Dispatch Queue**(串行队列)，另一种是**不等待**现在执行处理的**Concurrent Dispatch Queue**(并行队列)。

要比较这两种Dispatch Queue，先看下一段代码：

```objective-c
dispatch_async(queue,blk0);
dispatch_async(queue,blk1);
dispatch_async(queue,blk2);
dispatch_async(queue,blk3);
```

当变量queue是Serial Dispatch Queue时，因为要等到现在执行中的处理结束，所以这几个任务将会按照追加到queue中的顺序执行，**同时执行的任务只能有一个**，所以上述源代码的执行顺序如下：blk0➡️blk1➡️blk2➡️blk3.

当变量queue是Concurrent Dispatch Queue时，不用等待现在执行中的处理结束，所以首先执行blk0，不论blk0结束与否，都开始执行后面的blk1；然后不论blk1是否结束，开始执行blk2。。。。。以此类推。

这样可以并行执行多个处理，但是并行执行的处理数量由XNU决定，当处理结束，XNU会结束不在需要的线程。

### 2.2 dispatch_queue_create

通过==dispatch_queue_create==函数可生成Dispatch Queue。以下代码生成了一个 Dispatch Queue。

```objective-c
 //生成Serial Dispatch Queue 
dispatch_queue_t mySerialDispatchQueue = 
   dispatch_queue_create("com.example.gcd.MySerialDispatchQueue",NULL); 

 //生成Concurrent Dispatch Queue 
dispatch_queue_t myConcurrentDispatchQueue = 
  dispatch_queue_create("com.example.gcd.MyConcurrentDispatchQueue",DISPATCH_QUEUE_CONCURRENT); 
```

Dispatch Queue的名称推荐使用应用程序ID这种逆序全称域名（名称有点绕，但是看了上面的代码应该能大致理解什么意思了）。生成Serial Dispatch Queue 时，dispatch_queue_create的第二个参数设置为NULL；生成Concurrent Dispatch Queue 时，dispatch_queue_create的第二个参数设置为DISPATCH_QUEUE_CONCURRENT。

1️⃣Serial Dispatch Queue的适用情形：**多个线程更新相同资源导致数据竞争**时。

2️⃣Concurrent Dispatch Queue的适用情形：想**并行执行不发生数据竞争等问题的处理**时。

### 2.3 Main Dispatch Queue/Global Dispatch Queue

除了通过dispatch_queue_create函数可生成Dispatch Queue，还可以获取系统标准提供的Dispatch Queue。

1️⃣Main Dispatch Queue：是在主线程中执行的，因为主线程只有一个，因此Main Dispatch Queue就是Serial Dispatch Queue。追加到Main Dispatch Queue的处理在主线程的**RunLoop**中执行。一般会将一些用户界面更新等**必须在主线程中执行的处理**追加到Main Dispatch Queue中。可通过==dispatch_get_main_queue==函数获取。

2️⃣Global Dispatch Queue：所有应用程序都能够使用的Concurrent Dispatch Queue。有四个优先级：高优先级(High Priority)、默认优先级(Default Priority)、低优先级(Low Priority)、后台优先级(Background Priority)。但是通过XNU内核用于Global Dispatch Queue的线程并不能保证实时性，因此执行优先级只是大致判断。可通过==dispatch_get_global_queue==函数获取。

获取Dispatch Queue的源代码如下：

```objective-c
//获得Main Dispatch Queue
dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();

//获得Global Dispatch Queue（高优先级）
dispatch_queue_t globalDispatchQueueHigh = 					    
   dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);

//获得Global Dispatch Queue（默认优先级）
dispatch_queue_t globalDispatchQueueDefault = 
  dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

//获得Global Dispatch Queue（低优先级）
dispatch_queue_t globalDispatchQueueLow = 
  dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

//获得Global Dispatch Queue（后台优先级）
dispatch_queue_t globalDispatchQueueBackground = 
  dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);
```

### 2.4 dispatch_after

例如：3秒后将指定的Block追加到Main Dispatch Queue中：

```objective-c
//--------------dispatch_after--------------
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 3ull*NSEC_PER_SEC);  //现在开始3s后
dispatch_after(time, dispatch_get_main_queue(), ^{                           //追加到队列中
  NSLog(@"waited at least three seconds.");
});
```

==dispatch_after==函数并不是在指定时间后执行处理，而是在指定时间将处理追加到Dispatch Queue中。

dispatch_after的第一个参数是指定时间的dispatch_time_t类型，该值使用dispatch_time函数或dispatch_walltime函数生成。dispatch_time能从第一个参数指定的时间开始，到第二个参数指定的时间。上述代码中第一个参数DISPATCH_TIME_NOW表示现在的时间，3ull*NSEC_PER_SEC表示3秒。"ull"是C语言的数值字面量，显式表明类型"unsigned long long"。如果使用NSEC_PER_MSEC则以毫秒为单位计算。dispatch_time函数通常用于计算相对时间，dispatch_walltime函数用于计算绝对时间。

### 2.5 Dispatch Group

有时间我们可能会想要实现以下功能：在追加到Dispatch Queue中的**多个处理全部结束**后，想**执行结束处理**。如果实在Serial Dispatch Queue中，只需要将结束处理最后追加到队列中即可。但是在使用Concurrent Dispatch Queue时，就必须使用Dispatch Group。

例如要实现：追加3个Block到Global Dispatch Queue，这些Block如果全部执行完毕，就会执行Main Dispatch Queue中结束处理的Block。

```objective-c
//-----------Dispatch Group---------
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

//监视group中处理执行的结束，一旦所有处理执行结束，就可将结束处理追加至Main Dispatch Queue
dispatch_group_notify(group, dispatch_get_main_queue(), ^{NSLog(@"done");});
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/4.png)

Global Dispatch Queue中的几个处理是并行执行的，所以最后打印的顺序不定。而执行结束处理的done一定是最后输出的。

1️⃣dispatch_group_create函数生成dispatch_group_t类型的Dispatch Group。

2️⃣dispatch_group_async与dispatch_async函数相同，都追加Block到指定的Dispatch Queue中，但是它的第一个参数是Dispatch Group。

 3️⃣在追加到**Global Dispatch Queue中的处理全部执行结束**时，使用dispatch_group_notify将执行结束处理的Block追加到指定的Dispatch Queue中。

也可以使用dispatch_group_wait函数等待全部处理执行结束：

```objective-c
//-----------Dispatch Group---------
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull*NSEC_PER_SEC);
//long result = dispatch_group_wait(group, DISPATCH_TIME_FOREVER);      //永久等待
long result = dispatch_group_wait(group, time);                         //等待time
if(result==0)    //此时group中处理全部结束
{
  NSLog(@"done");
}
else             //group中仍然有处理正在运行中
  NSLog(@"timeout");
```

dispatch_group_wait函数的第二个参数为指定的等待时间（超时）。DISPATCH_TIME_FOREVER表示永久等待。如果dispatch_group_wait的返回值为0，表示group中全部处理结束；返回值不为0，意味着超过指定时间，

一般这种情形下，推荐用dispatch_group_notify函数追加结束处理到Main Dispatch Queue中，因为这样既可以简化源代码，也不需要耗费多余的等待时间。

### 2.6 dispatch_barrier_async

写入处理不可与其他的写入处理以及包含读取处理的其他处理并行执行，但是如果读取处理只与读取处理并行执行，就不会出现数据竞争问题。

为了高效率地进行访问，可以将读取处理追加到Concurrent Dispatch Queue中，写入处理在任一个读取处理没有执行的状态下，追加到Serial Dispatch Queue中（写入处理结束之前，读取处理不可执行）。

GCD提供了==dispatch_barrier_async==函数，**该函数和dispatch_queue_create函数生成的Concurrent Dispatch Queue一起使用可实现高效率的数据库访问和文件访问**。

```objective-c
//------------dispatch_barrier_async----------------
//在blk0～blk3读取数据执行结束后，写入数据，然后恢复读取数据
dispatch_queue_t queue = 
  dispatch_queue_create("com.example.gcd.ForBarrier",DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, ^{NSLog(@"blk0-------reading");});
dispatch_async(queue, ^{NSLog(@"blk1-------reading");});
dispatch_async(queue, ^{NSLog(@"blk2-------reading");});
dispatch_async(queue, ^{NSLog(@"blk3-------reading");});

dispatch_barrier_async(queue,^{NSLog(@"blk-------writing");});

dispatch_async(queue, ^{NSLog(@"blk4-------reading");});
dispatch_async(queue, ^{NSLog(@"blk5-------reading");});
dispatch_async(queue, ^{NSLog(@"blk6-------reading");});
dispatch_async(queue, ^{NSLog(@"blk7-------reading");});
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/5.png)

可以发现dispatch_barrier_async函数会等到追加到Concurrent Dispatch Queue的并行执行的读取处理都结束后，才将写入处理追加到队列中。

### 2.7 dispatch_async/dispatch_sync

前几节中我们要将处理追加到队列中，都使用dispatch_async函数，其中"==async=="表示"==非同步==(asynchronous)"，就是将指定的Block“非同步”地追加到指定的Dispatch Queue中。dispatch_async不做任何等待。

系统还提供了dispatch_sync函数，其中"==sync=="表示"==同步==(synchronous)"，就是将指定的Block“同步”地追加到指定的Dispatch Queue中。在追加的Block执行结束之前，函数会一直等待。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/6.png)

1️⃣使用dispatch_async函数追加处理到Concurrent Dispatch Queue中，异步+并行

```objective-c
- (void)asyncConcurrent
{
    //创建并行队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
    
    //通过异步函数将任务加入队列
    dispatch_async(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"1-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    dispatch_async(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"2-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    dispatch_async(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"3-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    //证明：异步函数添加到任务队列中，任务不会立即执行
    NSLog(@"asyncConcurrent----------end");
}
```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/7.png)

从执行结果可以看出，使用**dispatch_async追加处理到Concurrent Dispatch Queue中，开启多条线程，并且执行顺序不定。**

2️⃣使用dispatch_sync函数追加处理到Concurrent Dispatch Queue中，同步+并行

```objective-c
//使用dispatch_sync函数追加处理到Concurrent Dispatch Queue中
-(void) syncConcurrent
{
    //创建并行队列
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);

    //通过同步函数将任务加入队列
    dispatch_sync(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"1-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    dispatch_sync(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"2-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    dispatch_sync(queue, ^{
        for(NSInteger i=0;i<10;i++)
        {
            NSLog(@"3-----%@",[NSThread currentThread]);  //显示当前线程信息
        }
    });
    
    NSLog(@"syncConcurrent----------end");
}
```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志4)/8.png)

从执行结果可以看出，使用**dispatch_sync追加处理到Concurrent Dispatch Queue中，只会开启一条线程，并且按照追加顺序执行处理。**

⚠️20200702更新：iOS中中使用dispatch_sync(dispatch_get_main_queue(),blk)未必发生死锁，要看在什么线程中执行这个语句。在主线程中调用dispatch_sync要注意可能会被子线程阻塞，导致用户界面卡顿。

### 2.8 Dispatch Semaphore(信号量)

当并行执行的处理更新数据时，会产生数据不一致的情况，有时应用程序还会异常结束。虽然使用Serial Dispatch Queue和dispatch_barrier_async函数可以避免这类问题，但是有必要进行**更细粒度**的排他控制，也就是实现了**线程同步**。

==Dispatch Semaphore==是持有计数的信号，该计数是多线程编程中的计数类型信号。**当计数为0时等待，计数大于等于1时，减1而不等待**。

```objective-c
//--------------Dispatch Semaphore--------------
//生成Dispatch Semaphore，初始值设为1
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);    

//指定等待时间
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 1ull*NSEC_PER_SEC);
//dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

//等待Dispatch Semaphore的计数值大于等于1
long result = dispatch_semaphore_wait(semaphore, time);
if(result==0)
{
  /*
   Dispatch Semaphore计数值大于等于1，所以将计数值-1
   可执行需要进行排他控制的处理
   */
}
else
{
  /*
    Dispatch Semaphore计数值等于0
    在达到指定时间之前一直阻塞
   */
}
```

1️⃣通过**dispatch_semaphore_create**函数生成dispatch_semaphore_t变量，参数表示计数的初始值。

2️⃣**dispatch_semaphore_wait**函数等待Dispatch Semaphore的计数值大于等于1.第二个参数指定等待时间，DISPATCH_TIME_FOREVER表示永久等待。

3️⃣当计数值大于等于1时，对该计数-1并从函数返回0，可安全地执行需要进行排他控制的处理；否则到达指定时间一直等待。**dispatch_semaphore_signal**函数可以将计数值+1.

```objective-c
//-------------------通过Dispatch Semaphore实现线程同步------------------
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

//Dispatch Semaphore的计数值设为1，保证可访问类对象的线程同时只能有1个
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

NSMutableArray *array = [[NSMutableArray alloc]init];

for(int i=0; i<10; i++)
{
  dispatch_async(queue, ^{
    //等待Dispatch Semaphore的计数值大于等于1
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);

    //Dispatch Semaphore的计数值大于等于1，计数自动-1，返回，可执行排他控制处理
    [array addObject:[NSNumber numberWithInt:i]];

    //排他控制处理结束，将Dispatch Semaphore的计数值手动加1
    dispatch_semaphore_signal(semaphore);
  });
}
```

以上代码实现了每次只能有一个线程可以向array中添加数据。

### 2.9 dispatch_once

==dispatch_once==函数保证在应用程序执行中只执行一次指定处理的API。先看下面几行经常用来进行初始化的代码：

```objective-c
static int initialized = NO;
if(initialized==NO)
{
		//初始化
  	initialized = YES;
}
```

如果使用dispatch_once函数，则可以将上述代码写为：

```objective-c
static dispatch_once_t pred;
dispatch_once(&pred,^{
  //初始化
});
```

通过dispatch_once函数，源代码即使在多线程环境下，也可保证线程安全。dispatch_once函数可用于单例模式，用于生成单例对象。
