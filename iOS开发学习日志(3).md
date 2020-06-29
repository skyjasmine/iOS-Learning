# iOS开发学习日志(3)

## 1.第二期交流问题解答

### 1.1 Properties

1️⃣对自定义对象使用copy修饰时，需要注意什么？copy的默认行为是什么？

（1）对自定义对象使用copy修饰时，首先要让自定义类**遵循NSCopying协议**。

```objective-c
//----------Person.h----------
@interface Person:NSObject <NSCopying>
@end
```

然后在类的实现文件中实现**copyWithZone**方法：

```objective-c
//---------Person.m----------
#import "Person.h"
@implementation Person
-(id)copyWithZone:(NSZone *)zone
{
    Person *copy = [[[self class] allocWithZone:zone]init];
    copy.name = self.name;
    return copy;
}
@end
  
//---------main.m----------
#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p1 = [[Person alloc] init];
        p1.name = @"jasmine";
        Person *p2 = [p1 copy];
        NSLog(@"\n%@-----%@\n%@-----%@",p1.name,p1,p2.name,p2);
    }
    return 0;
}
```

执行结果：

![](https://github.com/skyjasmine/iOS-/blob/master/images/1.png)
可以看出实现了自定义对象的深拷贝。

（2）copy的默认行为：赋值给带有copy修饰的属性时，不保留新值，而是将重新拷贝一份对象在堆中。并且拷贝后返回的对象是一个**不可变对象**。

对**NSString、NSArray、NSDictionary**建议使用copy修饰，因为它们都有对应的Mutable类型，如果新值是可变对象，而不拷贝，那么该对象的值就可能在不知情的情况下被人修改。block也建议copy修饰。

对**可变属性，不建议**使用copy修饰。

2️⃣在对象dealloc时，可以使用weak指针指向对象吗？

不可以，dealloc时使用__weak指向自己，会发生闪退。
### 1.2 load & initialize

1️⃣load和initialize的标准写法

```objective-c
//load方法用来实现Method Swizzle
+(void)load
{
  	Method originalFunc = class_getInstanceMethod([self class],@selector(originalFunc));
  	Method SwizzledFunc = class_getInstanceMethod([self class],@selector(SwizzledFunc));
  
    method_exchangeImplementations(originalFunc, swizzledFunc);
}

//initialize方法主要用来对一些不方便在编译期初始化的对象进行赋值。比如NSMutableArray这种类型的实例化依赖于runtime的消息发送（是否是因为其能动态添加对象？？？），无法在编译期初始化。
+(void)initialize
{
  	if(self == [People class])
    {
      	//to do something......
		}
}
```

⚠️此处是没有灵魂的搜索博客的结果，本人目前对Method Swizzle不甚了解。

2️⃣load方法中调用了某些类方法，会触发initialize吗？

直觉告诉我不会。😌

### 1.3 Block

1️⃣调用block前养成判空的习惯

调用block前，如果block为nil，程序会崩溃。因此使用block前首先判断block是否为空。

```objective-c
!block? :block();
```

2️⃣block的使用带来的循环引用怎么解决？如果block中继续嵌套block的场景下应该怎么解决循环引用的问题？

```objective-c
//---------MyObect.h----------
#import <Foundation/Foundation.h>
typedef void(^blk_t) (void);
@interface MyObject : NSObject
{
    blk_t blk_;
}
@end
  
//----------MyObject.m---------
#import "MyObject.h"
@implementation MyObject
-(id)init
{
    self = [super init];
    blk_ = ^{NSLog(@"self = %@",self);};
    return self;
}

-(void)dealloc
{
    NSLog(@"dealloc");
}
@end
  
//--------------main.m------------
  int main(int argc, const char * argv[]) {
  @autoreleasepool {
    id o = [[MyObject alloc]init];
    NSLog(@"%@",o);
  }
  return 0;
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/3.png)

可以看出**dealloc方法没有被调用**。因为MyObject类对象持有Block对象blk_的强引用，而init实例方法中执行的Block语法使用了self。由于Block语法赋值在成员变量blk中，因此通过Block语法生成在栈上的Block此时由栈复制到堆，并持有self。此时发生了循环引用。

要避免循环应用，可以**声明附有__weak修饰符的变量**，并将self赋值使用。

```objective-c
-(id)init
{
    self = [super init];
  	id __weak weakSelf = self;
    blk_ = ^{NSLog(@"self = %@",weakSelf);};
    return self;
}
```

修改后执行结果：

![](https://github.com/skyjasmine/iOS-/blob/master/images/4.png)

可见能够正确dealloc。

## 2.Category类别

### 2.1 为什么要使用类别

(1)当一个类方法多且复杂时（应用较少）；

(2) ==为一个已经存在的类添加方法==。支持在**没有源代码时，基于某些特定场合，为一个类增加功能**。可以添加类方法、实例方法、重写本类方法；**不能添加属性，实例变量**（使用类别的大多数情况）。

类别不是子类，添加的方法实际上加到了直接操作的类的实现中，只要简单地声明类别，应用中该类的任何用户都可以在类的实例上访问到那些方法。

### 2.2 类别实现

1️⃣将一个臃肿的类分为多个分类

首先写一个Student类作为主类：

```objective-c
//--------Student.h---------
/*
	主类中实现学生类的基本行为方法：吃饭、睡觉
*/
#import <Foundation/Foundation.h>
@interface Student : NSObject
-(void)eat;
-(void)sleep;
@end
  
//--------Student.m--------
#import "Student.h"
@implementation Student
-(void)eat
{
    NSLog(@"我在吃饭");
}
-(void)sleep
{
    NSLog(@"我在睡觉");
}
@end 
  
```

接着为Student类添加分类，类名为：**本类名+分类名**。如：**Student+itcast**，即为Student类添加一个itcast分类，分类模块如下：

```objective-c
//--------Student+itcast.m---------
/*
	分类中实现学生类的娱乐行为方法
*/
#import "Student+itcast.h"
@implementation Student (itcast)
-(void)playLOL
{
    NSLog(@"我在打LOL");
}
-(void)sing
{
    NSLog(@"我在唱歌");
}
@end
  
//----------main.m----------
#import <Foundation/Foundation.h>
#import "Student.h"
#import "Student+itcast.h"                     //要访问分类的成员，需要将分类的头文件添加
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Student *s1 = [Student new];
        [s1 eat];
        [s1 sleep];
        [s1 playLOL];
        [s1 playLOL];
    }
    return 0;
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/5.png)

可以看出s1对象可以直接调用主类和分类中的所有方法。

2️⃣无类的源代码时，为类添加方法。

**非正式协议**：为系统自带的类写分类，叫做非正式协议。

```objective-c
//---------NSString+itcast.h----------
/*
  为NSString类添加一个方法numberCount：计算当前字符串中有多少阿拉伯数字
 */

#import <Foundation/Foundation.h>
@interface NSString (itcast)
//求当前字符串的阿拉伯数字个数
-(int)numberCount;
@end
  
//---------NSString+itcast.m----------
#import "NSString+itcast.h"
@implementation NSString (itcast)
-(int)numberCount
{
    int count = 0;
    for(int i=0;i<self.length;i++)
    {
        char ch = [self characterAtIndex:i];
        if(ch >= '0' && ch <= '9')
        {
            count++;
        }
    }
    return count;
}
@end

//-----------main.m-----------
#import <Foundation/Foundation.h>
#import "NSString+itcast.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSString *str = @"abc123edf456euf7eihfe890";
        int count = [str numberCount];
        NSLog(@"numberCount=%d",count);
    }
    return 0;
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/6.png)

可以看出为NSString添加类别后，所有的NSString对象都可以调用类别中的方法。

⚠️使用类别的注意点：

* 类别中只能添加方法，不能增加属性。
* 分类中可以存在与本类同名方法，优先调用类别方法，即使没有引入类别的头文件。
  如果多个分类中都有多个同名方法，优先调用最后编译的分类。

### 2.3 Extension延展

Extension用于**有源代码时，向类添加功能**，Extension是一个特殊的类别，所以Extension也是**类的一部分**。Extension的特殊之处在于：

（1）Extension这个特殊的分类**没有名字**

（2）**只有声明没有实现，和本类共享一个实现**

Extension不会独占一个文件，将Extension写在类的实现文件中，这个写在Extension中的成员，相当于类的私有成员，**只能类内访问**。类似于C++中的private成员。因此如果有一些不想暴露的成员，那么这个时候就可以使用延展来达到目的。

```objective-c
//-------------Student.m----------
#import "Student.h"
//将延展放在类的实现中
@interface Student ()
@property (nonatomic,assign)int age;
-(void)study;
@end

@implementation Student
-(void)study
{
    NSLog(@"我在学习呢，不要打扰我QAQ");
}

-(void)haveto
{
    self.age = 18;
    NSLog(@"我今年%d啦",self.age);
    [self study];
}
@end
  
//------------mian.m----------
#import <Foundation/Foundation.h>
#import "Student.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Student *s1 = [[Student alloc]init];
        [s1 haveto];        
    }
    return 0;
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/7.png)

一开始我没有注意延展中成员只能类内访问，试图在main中通过Student对象调用study方法，会出现如下错误提示：

![](https://github.com/skyjasmine/iOS-/blob/master/images/8.png)

## 3.Operation Queue

在书籍《Objective-C 高级编程》一书中，我学习了有关多线程的GCD(Grand Central Dispatch)，它是异步执行任务的技术之一。程序员只需要定义想执行的任务并追加到适当的Dispatch Queue中，GCD就能生成必要的线程并计划执行任务。

而操作队列(Operation Queue)是由GCD提供的一个队列模型的Cocoa抽象，它在GCD之上实现了一些方便的功能，这些功能使得Operation Queue成为并发编程的首选工具。

### 3.1 NSOperation、NSOperationQueue操作和操作队列

（1）操作(Operation):就是在线程中执行的那段代码。GCD中操作是放在block中，而NSOperation中，使用NSOperation子类==NSInvocationOperation==**、==NSBlockOperation==，或者==自定义子类==来封装操作。

（2）操作队列(Operation Queues):

* 添加到NSOperationQueue队列中的操作，首先进入准备就绪的状态（就绪状态取决于操作之间的依赖关系），然后进入就绪状态的操作的**开始执行顺序**由操作之间的**优先级决定**（优先级是操作对象自身的属性）。
* Operation Queues通过设置**最大并发操作数**（MaxConcurrentOperationCount）来控制并发，串行。当

MaxConcurrentOperationCount=1时，串行执行操作。

* NSOperationQueue中有两种队列：主队列、自定义队列。**主队列**运行在**主线程**上，**自定义队列**在**后台**执行。

（3）NSOperation实现多线程的步骤：

1️⃣将需要执行的操作封装到NSOperation子类对象中；

2️⃣创建NSOperationQueue对象，并将NSOperation子类对象添加到NSOperationQueue对象中。

### 3.2 NSOperation、NSOperationQueue基本使用

#### 3.2.1 创建Operation

NSOperation是抽象类，不能用来封装操作，只能使用其子类封装操作。

1️⃣使用子类NSInvocationOperation

```objective-c
//-------------OperationQueue.m------------
#import "OperationQueue.h"
@implementation OperationQueue
  
  /*
  	使用子类NSInvocationOperation
  */
-(void)useInvocationOperation
{
    //创建NSInvocationOperation对象
    NSInvocationOperation *op = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(task1) object:nil];
    
    //调用start方法开始执行操作
    [op start];
}
-(void)task1
{
    for(int i=0;i<2;i++)
    {
        [NSThread sleepForTimeInterval:2];
        NSLog(@"1-----%@",[NSThread currentThread]);  //打印当前线程
    }
}
@end
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/9.png)
可以看出不使用NSOperationQueue，主线程中单独使用子类NSInvocationOperation时，操作就**在当前线程执行**，没有开启新线程。

2️⃣使用子类NSBlockOperation

```objective-c
//-------------OperationQueue.m------------
 /*
  	使用子类NSInvocationOperation
  */
-(void)useBlockOperation
{
    //1.创建NSBlockOperation对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^{
        for(int i=0;i<2;i++)
        {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    //2.调用start方法开始执行操作
    [op start];
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/10.png)
可以看出不使用NSOperationQueue，主线程中单独使用子类NSBlockOperation时，操作就**在当前线程执行**，没有开启新线程。

NSBlockOperation还提供了方法**addExecutionBlock:**，可以为NSBlockOperation添加额外操作。这些操作（包括b lockOperationWithBlock中的操作），可以在不同线程并发执行。只有当所有操作都完成执行时，才看作完成.

```objective-c
//-------------OperationQueue.m------------
#import "OperationQueue.h"
@implementation OperationQueue
   /*
  	使用子类NSInvocationOperation，并调用方法addExecutionBlock:
  */
-(void)useBlockOperationAddExecutionBlock
{
    //1.创建NSBlockOperation对象
    NSBlockOperation *op = [NSBlockOperation blockOperationWithBlock:^
    {
        for(int i=0;i<2;i++)
        {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    //2.添加额外操作
    [op addExecutionBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"2-----%@",[NSThread currentThread]);
        }
    }];
    
    [op addExecutionBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"3-----%@",[NSThread currentThread]);
        }
    }];
    
    [op addExecutionBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"4-----%@",[NSThread currentThread]);
        }
    }];
    
    [op addExecutionBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"5-----%@",[NSThread currentThread]);
        }
    }];
    
    //3.调用start方法开始执行操作
    [op start];
}
@end
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/11.png)
可见使用子类**NSBlockOperation**并调用方法**addExecutionBlock:**时，**blockOperationWithBlock:**中的操作和**addExecutionBlock:**添加的操作在不同线程中并发执行。并且**blockOperationWithBlock:**方法中的操作也不一定会在当前线程中执行。（这是博客中看到的结论，我自己做了几次blockOperationWithBlock:中的操作都是在main中执行😭）

如果一个NSBlockOperation对象封装了多个操作，是否开启新线程由操作个数决定，当操作个数较多时，就会自动开启新线程，线程个数由系统决定。

3️⃣使用继承自NSOperation的自定义子类

自定义NSOperation子类，通过重写main或者start方法定义NSOperation对象。重写main函数不需要管理操作的状态属性isExecuting和isFinished。当main执行完返回时，操作就结束了。

```objective-c
//------------MyOperation.h-----------
#import <Foundation/Foundation.h>
@interface MyOperation : NSOperation
-(void)useCustomOperation;
@end

//------------MyOperation.m-----------
#import "MyOperation.h"
@implementation MyOperation
-(void)main
{
    if(!self.isCancelled)
    {
        for(int i=0;i<2;i++)
        {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }
}

-(void)useCustomOperation
{
    //1.创建MyOperation对象
    MyOperation *op = [[MyOperation alloc]init];
    //2.调用start方法开始执行
    [op start];
}
@end
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/12.png)
可以看出不使用NSOperationQueue，主线程中单独使用自定义子类时，操作就**在当前线程执行**，没有开启新线程。

#### 3.2.1 创建Operation Queue

NSOperationQueue一共有两种队列：==主队列==和==自定义队列==。

1️⃣主队列

添加到==主队列==中的操作，都会放到==**主线程**==中执行，**不包括**addExecutionBlock添加的额外操作，额外操作可能在其他线程执行。

```objective-c
//主队列获取方法
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```

2️⃣自定义队列

添加到==自定义队列==的操作，自动放到==**子线程**==执行。包含**串行**、**并发**功能。

```objective-c
//自定义队列创建方法
NSOperationQueue *queue = [[NSOperationQueue alloc]init];
```

#### 3.2.3 将操作加入队列中

1️⃣先创建操作，再将创建好的操作加入到队列中。

```objective-c
//---------OperationQueue.m--------
/*
	使用addOperation:将操作加入队列中
*/
-(void)addOperationToQueue
{
    //1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
    
    //2.创建操作
    //使用NSInvocationOperation创建操作1.2
    NSInvocationOperation *op1 = [[NSInvocationOperation alloc]initWithTarget:self selector:@selector(task1) object:nil];
    NSInvocationOperation *op2 = [[NSInvocationOperation alloc]initWithTarget:self selector:@selector(task2) object:nil];

    //使用NSBlockOperation创建操作3
    NSBlockOperation *op3 = [NSBlockOperation blockOperationWithBlock:^{
        for(int i=0;i<2;i++)
       {
          [NSThread sleepForTimeInterval:2];
          NSLog(@"3-----%@",[NSThread currentThread]);
       }
    }];
    
    [op3 addExecutionBlock:^{
        for(int i=0;i<2;i++)
       {
          [NSThread sleepForTimeInterval:2];
          NSLog(@"4-----%@",[NSThread currentThread]);
       }
    }];
    
    //3.将操作加入队列中
    [queue addOperation:op1];    //相当于[op1 start]
    [queue addOperation:op2];    //相当于[op2 start]
    [queue addOperation:op3];    //相当于[op3 start]
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/13.png)
可以看出使用NSOperation子类创建操作，并使用**addOperation:**将操作加入队列后，系统**开启新线程，并发执行**。

2️⃣直接在Block中添加操作，然后将Block加入队列

```objective-c
/*
 使用addOperationWithBlock:将操作加入队列
 */
-(void)addOperationWithBlockToQueue
{
    //1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
    
    //使用addOperationWithBlock:将操作加入队列
    [queue addOperationWithBlock:^{
         for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    [queue addOperationWithBlock:^{
            for(int i=0;i<2;i++)
           {
              [NSThread sleepForTimeInterval:2];
              NSLog(@"2-----%@",[NSThread currentThread]);
           }
       }];
    
    [queue addOperationWithBlock:^{
            for(int i=0;i<2;i++)
           {
              [NSThread sleepForTimeInterval:2];
              NSLog(@"3-----%@",[NSThread currentThread]);
           }
       }];
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/14.png)
可以看出使用**addOperationWithBlock:**将操作加入队列后，系统同样会**开启新线程，并发执行**。

#### 3.2.4 NSOperationQueue控制串行执行、并发执行

NSOperationQueue有一个属性**maxConcurrentOperationCount**，可以控制一个队列中最多同时参与并发执行的操作数。maxConcurrentOperationCount不是控制并发线程数量，而是一个队列中能并发执行的最大操作个数，而且一个操作并非只能在一个线程中运行。

1️⃣maxConcurrentOperationCount默认下为-1，表示不限制同时并发执行的操作数，可并发执行。

2️⃣maxConcurrentOperationCount为==1==时，队列为==串行==队列，串行执行操作。

3️⃣maxConcurrentOperationCount大于1时，队列为并发队列，并发执行操作。注意maxConcurrentOperationCount值不能超过系统限制。

```objective-c
/*
 设置maxConcurrentOperationCount
 */
-(void)setMaxConcurrentOperationCount
{
    //1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
    
    //2.设置maxConcurrentOperationCount
    queue.maxConcurrentOperationCount = 1;     //串行队列
 // queue.maxConcurrentOperationCount = 2;     //并发队列

    // 3.添加操作
    [queue addOperationWithBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    [queue addOperationWithBlock:^{
         for(int i=0;i<2;i++)
         {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"2-----%@",[NSThread currentThread]);
         }
     }];
    
    [queue addOperationWithBlock:^{
         for(int i=0;i<2;i++)
         {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"3-----%@",[NSThread currentThread]);
         }
     }];
    
    [queue addOperationWithBlock:^{
         for(int i=0;i<2;i++)
         {
            [NSThread sleepForTimeInterval:2];
            NSLog(@"4-----%@",[NSThread currentThread]);
         }
     }];
}
```

maxConcurrentOperationCount=1时，执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/15.png)
maxConcurrentOperationCount=2时，执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/16.png)
从两个执行结果中，可以看出最大并发操作数为1时，操作按照顺序串行执行，并且所有操作在两个线程中完成；最大并发操作数为2时，操作是并发执行的，可以同时执行两个操作。开启多少线程是由系统决定的。

### 3.3 NSOperation操作依赖

NSOperation、NSOperationQueue能够添加操作之间的依赖关系。通过操作依赖，可以控制操作之间的执行顺序。

现有以下需求：有A、B两个操作，要求A执行完操作后，B才能执行。让操作B依赖操作A：

```objective-c
/*
 使用addDependency:方法实现操作依赖
 */
-(void)addDependency
{
    //1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
    
    //2.创建操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"2-----%@",[NSThread currentThread]);
        }
    }];
    
    //3.添加依赖
    [op2 addDependency:op1];    //⚠️op2依赖于op1，即先执行op1，再执行op2
    
    //4.操作添加到队列中
    [queue addOperation:op1];
    [queue addOperation:op2];
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/17.png)
可以看出添加依赖后，能够实现让op1在op2之前执行。

### 3.4 线程间通信

iOS开发中，通常在主线程中进行UI刷新，如：点击、滚动、拖拽等操作；在后台线程进行一些比较耗时的操作，如：图片下载、文件上传等。当后台线程的耗时操作执行完成后，需要回到主线程，即向主线程发出通知。

```objective-c
/*
	线程间通信
*/
-(void)communication
{
    //1.创建队列
    NSOperationQueue *queue = [[NSOperationQueue alloc]init];
    
    //2.开辟后台线程，执行操作
    [queue addOperationWithBlock:^{
        //进行耗时操作
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"1-----%@",[NSThread currentThread]);
        }
    }];
    
    
    //3.回到主线程
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        //进行一些UI刷新等操作
        for(int i=0;i<2;i++)
        {
           [NSThread sleepForTimeInterval:2];
           NSLog(@"2-----%@",[NSThread currentThread]);
        }
    }];
}
```

如果后台线程中操作执行间隔为2s时，执行结果如下：
![](https://github.com/skyjasmine/iOS-/blob/master/images/18.png)

如果后台线程中相邻操作没有执行间隔时，执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/19.png)
从上述两张截图中可以看出，如果后台线程操作无延时，那么执行完所有操作才会返回主线程；如果两个操作间有延时，那么执行完当前操作后，立刻就回到主线程，然后再回后台线程执行操作。

### 3.5 线程安全

**线程安全**：当多个线程同时访问相同内存或数据，具体行为是不确定的。这会导致访问到的结果跟预期不同，这是在编写执行代码时未考虑线程安全。线程安全要求多线程运行和单线程运行的结果一致，获得变量结果也和预期一样。

如果每个线程对全局变量、静态变量**只有读**操作，一般是**线程安全**。因此线程安全的关键是确保在写入某个特定内存块时，没有其他线程可以在其完成前进行读取。即考虑**线程同步**问题。

**线程同步**：同步不是同时进行，而是**协同步调，按照预定的次序先后执行**。例如：两个人面对面聊天，两个人不能同时说话，会导致听不清（操作冲突）。一个人说完（线程结束操作），另一个再说（另一个线程开始操作）。

解决方案：给==线程加锁==，在一个线程中执行该操作时，不允许其他线程进行操作。iOS加锁有：@synchronized、NSLock、NSRecursiveLock、属性修饰符atomic等。

案例：共有10张火车票，两个售票窗口，一个A窗口，一个B窗口。生活经验告诉我们，两个人不可能买到同一张票，即有一个窗口在售某一张票时，另一个窗口是无法对其进行操作的。

针对这个问题，使用NSLock对象解决线程同步问题。NSLock对象可以通过进入锁时调用lock方法，解锁时调用unlock方法保证线程安全。

```objective-c
/*
   售卖火车票（线程安全）
 */
-(void)sellTicket
{
    while(1)
    {
        //加锁
        [self.lock lock];
        
        //有余票，继续售卖
        if(self.ticketSurplusCount>0)
        {
            self.ticketSurplusCount--;
            NSLog(@"%@",[NSString stringWithFormat:@"剩余票数：%d，窗口：%@",self.ticketSurplusCount,[NSThread currentThread]]);
            [NSThread sleepForTimeInterval:0.2];
        }
        
        //解锁
        [self.lock unlock];
        
        if(self.ticketSurplusCount<=0)
        {
            NSLog(@"所有火车票已售完");
            break;
        }
    }
}

-(void)initTicketStatus
{
    NSLog(@"currentThread------%@",[NSThread currentThread]);
    
    self.ticketSurplusCount = 50;
    self.lock = [[NSLock alloc]init];        //初始化NSLock对象
    
    //1.创建queue1、queue2代表A、B两个售票窗口
    NSOperationQueue *queue1 = [[NSOperationQueue alloc]init];
    queue1.maxConcurrentOperationCount = 1;
    
    NSOperationQueue *queue2 = [[NSOperationQueue alloc]init];
    queue2.maxConcurrentOperationCount = 1;
    
    //2.创建售票操作
    NSBlockOperation *op1 = [NSBlockOperation blockOperationWithBlock:^{
        [self sellTicket];
    }];
    
    NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
        [self sellTicket];
    }];
    
    [queue1 addOperation:op1];
    [queue2 addOperation:op2];
}
```

执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/20.png)
从结果可以看出，使用NSLock加锁、解锁得到正确的票数，解决了线程安全问题。

### 3.6 NSOperation、NSOperationQueue常用方法

#### 3.6.1 Operation

1️⃣判断操作状态方法：

```objective-c
-(BOOL)isFinished;      //判断操作是否已经结束
-(BOOL)isCancelled;     //判断操作是否已经取消
-(BOOL)isExecutin;      //判断操作是否正在运行
-(BOOL)isReady;         //判断操作是否处于准备就绪状态，与操作的依赖关系相关
```

2️⃣操作同步：

```objective-c
-(void)waitUntilFinished;                   //阻塞当前线程，直到操作结束。可用于线程执行顺序的同步
-(void)addDependency:(NSOperation*)op;      //添加依赖
@property (readonly,copy)NSArray<NSOperation*> *dependencies;     //在当前操作开始执行之前完成执行的所有操作对象数组
```

#### 3.6.2 OperationQueue

1️⃣取消、暂停、恢复操作

```objective-c
-(void)cancelAllOperations;      //取消队列的所有操作
-(BOOL)isSuspended;              //判断队列是否处于暂停状态。YES：暂停，NO：恢复
-(void)setSuspended:(BOOL)b;     //设置队列暂停或恢复。YES：暂停，NO：恢复
```

2️⃣操作同步

```objective-c
-(void)waitUntilAllOperationsAreFinished;     //阻塞当前线程，直到队列中的操作全部执行完毕
```

3️⃣添加、获取操作

```objective-c
-(void)addOperationWithBlock:(void (^)(void))block;          //向队列中添加NSBlockOperation对象
-(void)addOpertions:(NSArray*)ops  waitUntilFinished:(BOOL)wait;      //向队列添加操作数组，wait
标记是否阻塞当前线程直到所有操作结束
-(NSArray*)operations;                 //当前队列中的操作数组，操作执行结束后会自动从数组清除
-(NSUInteger)operationCount;           //当前队列的操作数
```

操作、队列的暂停和取消，不是说执行后将当前操作立即暂停/取消运行，而是在**当前操作执行完成后不再执行新的操作。**
