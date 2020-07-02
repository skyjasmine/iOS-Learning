## 1.Runtime（运行时系统）

Objective-C不能直接编译为汇编语言，而是要先转为C语言再进行编译和汇编的操作，从OC到C语言的过渡就是由==runtime==实现。

### 1.1 obj_msgSend

OC中的实例对象调用一个方法称作消息传递。消息传递采用动态绑定机制来决定具体调用哪个方法，OC的实例方法转写为C语言后实际上就是一个函数，但是OC并不是在编译期决定调用哪个函数，而是在运行期决定，因为编译期无法确定最终会调用哪个函数，这是由于运行期是可以修改方法的实现的。Objective-C中方法调用，本质是发送消息。即：[receiver message]会被编译器转化为：objc_msgSend(receiver, selector)。**objc_msgSend**函数原型如下：

```objective-c
OBJC_EXPORT void objc_msgSend(void /* id self, SEL op, ... */ )
```

该函数有两个参数，一个**id**类型，一个**SEL**类型，它们分别表示方法的调用者和方法选择器selector。

SEL被定义在objc/objc.h中：

```objective-c
typedef struct objc_selector *SEL;
```

selector可以理解为一个字符串类型的名称，用于查找对应的函数实现。

其中id是一个结构体指针，可以指向Objective-C的任何对象。

```objective-c
typedef struct objc_object *id;
```

其中objc_object结构体定义如下：

```objective-c
struct objc_object { Class isa OBJC_ISA_AVAILABILITY;};
```

这就是**通常所说的对象**，这个结构体只有一个isa变量，对象可以通过isa指针找到其所属的类。isa是一个Class类型的成员变量，而Class也是一个结构体指针类型：

```objective-c
typedef struct objc_class *Class;
```

它的定义如下：

```objective-c
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class   OBJC2_UNAVAILABLE;
    const char *name   OBJC2_UNAVAILABLE;
    long version     OBJC2_UNAVAILABLE;
    long info     OBJC2_UNAVAILABLE;
    long instance_size   OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars  OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists  OBJC2_UNAVAILABLE;
    struct objc_cache *cache    OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols   OBJC2_UNAVAILABLE;
#endif
} OBJC2_UNAVAILABLE;
```

这就是通常所说的类：

* isa指针：指向所属**元类**（meta），元类中保存了创建**类对象**和**类方法**（+）所需的信息。
* super_class：指向超类
* name：类名
* version：类的版本信息
* info：类的详情
* instance_size：该类实例对象的大小
* ivars：指向类的实例变量列表
* methodLists：指向该类的实例方法（-）列表，它将方法选择器和实现的地址联系起来。
* cache：Runtime系统会把被调用的方法存到cache中（一个方法被调用，今后很可能也会被调用，我的理解有点程序局部性原理的意思），下次查找更快。
* protocols：指向该类的协议列表。

实例对象是一个结构体，该结构体只有一个成员变量，指向构造它的类对象，这个类对象中存储了一切实例对象需要的信息，类对象是通过元类创建的。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/1.png)

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/2.png)
**==综上，当调用一个方法时，运行过程大致如下：==**

1️⃣Runtime会把方法调用转化为消息发送，即obj_msgSend，并把**方法的调用者**和**方法选择器**当作参数传递过去；

2️⃣方法调用者会通过isa指针找到其所属类，然后在cache或者methodLists中查找该方法，如果找到就跳到对应的方法去执行；

3️⃣如果在当前类中没有找到该方法，则通过super_class向上一级超类查找，如果找不到则会沿着继承树向上一直搜索直到NSObject,如果一直找到NSObject都没有找到，并且消息转发都失败了就会报错：unrecognized selector。

### 1.2 消息转发（Message Forwarding）

消息转发是在unrecognized selector之前，是查找方法的最后三次机会。

1️⃣ 第一次机会：所属类动态方法分析

首先如果沿继承树没有查找到相关方法，则会向**接受者所属的类**进行一次请求，看**是否能够动态的添加一个方法**。

```objective-c
+ (BOOL)resolveInstanceMethod:(SEL)name;
```

当找不到相关实例方法，就会调用该类方法去询问是否可以动态添加，如果返回YES，就会再次执行相关方法。如果要给一个类动态添加一个方法，可以调用runtime库中的class_addMethod方法，原型是：

```objective-c
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

class_addMethod函数的第一个参数是需要添加方法的类；第二个参数是一个selector，也就是实例方法的名字；第三个参数是一个IMP类型变量，也就是函数实现，需要传入一个C函数，这个函数至少有两个参数，一个是id self，一个是SEL _cmd；第四个参数是函数类型。

```objective-c
//------------------------Person.h-----------------------------
#import <Foundation/Foundation.h>

@interface Person : NSObject
@property (nonatomic, copy) NSString* name;
@property (nonatomic, assign) NSUInteger age;
@end
  
//------------------------Person.m-----------------------------
#import "Person.h"
#import <objc/runtime.h>

@implementation Person
@synthesize name = _name;
@synthesize age = _age;

//如果需要传参直接在参数列表后面添加就好了
void dynamicAdditionMethodIMP(id self, SEL _cmd) {
    NSLog(@"dynamicAdditionMethodIMP");
}

+ (BOOL)resolveInstanceMethod:(SEL)name {
    NSLog(@"resolveInstanceMethod: %@", NSStringFromSelector(name));
    if (name == @selector(appendString:)) {
        class_addMethod([self class], name, (IMP)dynamicAdditionMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:name];
}

+ (BOOL)resolveClassMethod:(SEL)name {
    NSLog(@"resolveClassMethod %@", NSStringFromSelector(name));
    return [super resolveClassMethod:name];
}
@end

  
//------------------------main.m-----------------------------
#import <Foundation/Foundation.h>
#import <objc/objc.h>
#import <objc/runtime.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        id p = [[Person alloc] init];     //注意一定是id类型，否则编译会因为无appenString方法而报错
        [p appendString:@""];
    }
    return 0;
}

```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/3.png)
main函数中创建了一个Person的实例对象，⚠️注意一定要用**id**类型声明，否则编译期就会报错，因为找不到相关函数声明。而id类型由于可以指向任意类对象，所以编译时找到NSString类的appendString: 方法就不会报错。

Person类没有声明和定义appendString: 方法，运行时应该报错：unrecognized selector，但是运行的结果发现实际上并没有报错，这是因为在Person.m文件中重写了类方法 + (BOOL)resolveInstanceMethod:(SEL)name 。

2️⃣备援接收者

当对象所属类不能动态添加方法时，runtime就会询问当前的接受者是否有其他对象可以处理未知的selector，相关方法原型如下：

```objective-c
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

该方法的参数就是未知的selector，这是一个实例方法，因为是询问该实例对象是否有其他实例对象可以接收未知selector，如果没有就返回nil。

3️⃣消息重定向

当没有备援接收者使，只剩下最后一次消息转发的机会，即：消息重定向。这个时候runtime会将未知消息的所有细节都封装为NSInvocation对象，然后调用下面的方法：

```objective-c
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```

调用这个方法如果不能处理就会调用父类的相关方法，一直到NSObject的这个方法，如果此时也无法处理就会抛出异常。

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/4.png)


### 1.3 OC的属性property

现在有一个自定义类Person：

```objective-c
//----------------Person.h--------------------
@interface Person : NSObject

@property (nonatomic, copy) NSString* name;
@property (nonatomic, assign) NSUInteger age;

@end
  
//----------------Person.m--------------------
@implementation Person

@synthesize name = _name;
@synthesize age = _age;

@end
```

同样经过clang转写后，得到属性的底层实现的两个相关结构体：

```objective-c
//clang转写Person.m的相关代码
struct _prop_t {
	const char *name;
	const char *attributes;
};

static struct /*_prop_list_t*/ {
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[2];
} _OBJC_$_PROP_LIST_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	2,
	{{"name","T@\"NSString\",C,N,V_name"},
	{"age","TQ,N,V_age"}}
};
```

其中**_prop_t**结构体描述了每一个属性，包括名称和属性值。

**_prop_list_t**表示属性列表，记录了每一个属性的大小，属性个数以及具体的属性描述，每加入一个属性就会在**prop_list**增加一个属性描述。属性描述中"T@"表示类型对象，后面是类型名称，C表示copy，N表示Nonatomic，默认修饰符则不会出现，V_name表示实例变量。

我们可以通过以下代码来获取上面的结构体：

```objective-c
//------------------------main.m-----------------------------
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [Person alloc];
        p = [p init];
        p.name = @"jasmine";
        
        unsigned int propertyCount = 0;
        objc_property_t *propertyList = class_copyPropertyList([p class], &propertyCount);
        for(int i=0; i< propertyCount; i++)
        {
            const char *name = property_getName(propertyList[i]);
            const char *attributes = property_getAttributes(propertyList[i]);
            NSLog(@"%s %s",name,attributes);
        }
    }
    return 0;
}

```

在<objc/runtime.h>中找到objc_property_t的相关定义：

```objective-c
/// An opaque type that represents an Objective-C declared property.
typedef struct objc_property *objc_property_t;
```

可以看出**objc_property_t**是一个指向**objc_property**结构体的指针，struct objc_property其实就是上文所说的**_prop_t**结构体，通过**class_copyPropertyList**就可以得到相关类的所有属性，其相关声明如下：

```objective-c
/** 
 * Describes the properties declared by a class.
 * 
 * @param cls The class you want to inspect.
 * @param outCount On return, contains the length of the returned array. 
 *  If \e outCount is \c NULL, the length is not returned.        
 * 
 * @return An array of pointers of type \c objc_property_t describing the properties 
 *  declared by the class. Any properties declared by superclasses are not included. 
 *  The array contains \c *outCount pointers followed by a \c NULL terminator. You must free the array with \c free().
 * 
 *  If \e cls declares no properties, or \e cls is \c Nil, returns \c NULL and \c *outCount is \c 0.
 */
OBJC_EXPORT objc_property_t _Nonnull * _Nullable
class_copyPropertyList(Class _Nullable cls, unsigned int * _Nullable outCount)
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0, 2.0);
```

class_copyPropertyList的第一个参数是相关类的类对象，第二个参数指明property的数量。通过该方法能够获取所有属性，然后通过**property_getName**和**property_getAttributes**方法获取某个属性的name和attributes值，上面main函数的输出结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/5.png)
可以看出结果与之前通过clang转写后得到的属性完全一致。

### 1.4 Category的实现原理

#### 1.4.1 category相关结构体

首先自定义一个类MyClass，并为其添加类别：

```objective-c
//--------------------MyClass.h-------------------
#import <Foundation/Foundation.h>

@interface MyClass : NSObject

- (void)printName;

@end

@interface MyClass (MyAddtion)

@property (nonatomic, copy) NSString *name;

- (void)printName;

@end

//--------------------MyClass.m-------------------
#import "MyClass.h"
  
@implementation MyClass

- (void)printName
{
    NSLog(@"--------MyClass.");
}

@end

@implementation MyClass(MyAddtion)

- (void)printName
{
    NSLog(@"--------MyAddtion.");
}

@end
```

通过clang命令转写为cpp文件后，找到与category相关的结构体：

```objective-c
struct _category_t {
	const char *name;             //类名
	struct _class_t *cls;         //指针指向类
	const struct _method_list_t *instance_methods;   //category中给类添加的实例方法列表
	const struct _method_list_t *class_methods;      //category中给类添加的类方法列表
	const struct _protocol_list_t *protocols;        //category实现的所有协议的列表
	const struct _prop_list_t *properties;           //category中添加的属性
};
```

从**_category_t**结构体中可以看出category可以为类添加实例方法、类方法、甚至添加协议、属性，但是**不能添加实例变量。**

接着往下查找，可以找到如下代码：

```objective-c
static struct /*_method_list_t*/ {                         //实例方法列表
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_MyClass_$_MyAddtion __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"printName", "v16@0:8", (void *)_I_MyClass_MyAddtion_printName}}
};

static struct /*_prop_list_t*/ {                          //属性列表
	unsigned int entsize;  // sizeof(struct _prop_t)
	unsigned int count_of_properties;
	struct _prop_t prop_list[1];
} _OBJC_$_PROP_LIST_MyClass_$_MyAddtion __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_prop_t),
	1,
	{{"name","T@\"NSString\",C,N"}}
};

extern "C" __declspec(dllexport) struct _class_t OBJC_CLASS_$_MyClass;

static struct _category_t _OBJC_$_CATEGORY_MyClass_$_MyAddtion __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"MyClass",
	0, // &OBJC_CLASS_$_MyClass,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_MyClass_$_MyAddtion,
	0,
	0,
	(const struct _prop_list_t *)&_OBJC_$_PROP_LIST_MyClass_$_MyAddtion,
};
static void OBJC_CATEGORY_SETUP_$_MyClass_$_MyAddtion(void ) {
	_OBJC_$_CATEGORY_MyClass_$_MyAddtion.cls = &OBJC_CLASS_$_MyClass;
}
#pragma section(".objc_inithooks$B", long, read, write)
__declspec(allocate(".objc_inithooks$B")) static void *OBJC_CATEGORY_SETUP[] = {
	(void *)&OBJC_CATEGORY_SETUP_$_MyClass_$_MyAddtion,
};
static struct _class_t *L_OBJC_LABEL_CLASS_$ [1] __attribute__((used, section ("__DATA, __objc_classlist,regular,no_dead_strip")))= {
	&OBJC_CLASS_$_MyClass,
};
static struct _category_t *L_OBJC_LABEL_CATEGORY_$ [1] __attribute__((used, section ("__DATA, __objc_catlist,regular,no_dead_strip")))= {
	&_OBJC_$_CATEGORY_MyClass_$_MyAddtion,
};
```

可以看出编译期编译器做了以下工作：

1️⃣编译器生成了实例方法列表OBJC$CATEGORY_INSTANCE_METHODS_MyClass_$_MyAddtion和属性列表OBJC_$_PROP_LIST_MyClass_$_MyAddtion，而且实例方法里面的是我们在MyAddtion这个category中写的方法printName，属性列表中填充的也是MyAddtion中添加的name属性。

2️⃣编译器生成了category_t类型的OBJC_$_CATEGORY_MyClass_$_MyAddtion，并用前面的列表初始化本身。

3️⃣最后编译器在DATA段下的objc_catlis section中保存了一个大小为1的category_t数组，L_OBJC_LABEL_CATEGORY_$（如果有多个category，会生成对应长度数组），用于运行期category的加载。

#### 1.4.2 运行时category的加载

首先在runtime源码工程的objc-runtime-new.mm文件中找到以下函数：

```objective-c
// Discover categories. 
    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);

            if (!cls) {
                // Category's target class is missing (probably weak-linked).
                // Disavow any knowledge of this category.
                catlist[i] = nil;
                if (PrintConnecting) {
                    _objc_inform("CLASS: IGNORING category \?\?\?(%s) %p with "
                                 "missing weak-linked target class", 
                                 cat->name, cat);
                }
                continue;
            }

            // Process this category. 
            // First, register the category with its target class. 
            // Then, rebuild the class's method lists (etc) if 
            // the class is realized. 
            bool classExists = NO;
            if (cat->instanceMethods ||  cat->protocols    //实例方法、协议、属性
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);   //添加到主类中cls
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category -%s(%s) %s", 
                                 cls->nameForLogging(), cat->name, 
                                 classExists ? "on existing class" : "");
                }
            }

            if (cat->classMethods  ||  cat->protocols  //类方法、协议
                /* ||  cat->classProperties */) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);    //添加到元类cls->ISA()
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
                if (PrintConnecting) {
                    _objc_inform("CLASS: found category +%s(%s)", 
                                 cls->nameForLogging(), cat->name);
                }
            }
        }
    }
```

从上面代码可以看出完成了两个工作：

1️⃣将category的==实例方法、协议、属性添加到类==上

2️⃣将category的==类方法、协议添加到类的元类==上

上面的实现最终都是通过调用**remethodizeClass**实现的，该函数的原型：

```objective-c
static void remethodizeClass(class_t *cls)
{
    category_list *cats;
    BOOL isMeta;
 
    rwlock_assert_writing(&runtimeLock);
 
    isMeta = isMetaClass(cls);
 
    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls))) {
        chained_property_list *newproperties;
        const protocol_list_t **newprotos;
 
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s",
                         getName(cls), isMeta ? "(meta)" : "");
        }
 
        // Update methods, properties, protocols
 
        BOOL vtableAffected = NO;
        attachCategoryMethods(cls, cats, &vtableAffected);
 
        newproperties = buildPropertyList(NULL, cats, isMeta);
        if (newproperties) {
            newproperties->next = cls->data()->properties;
            cls->data()->properties = newproperties;
        }
 
        newprotos = buildProtocolList(cats, NULL, cls->data()->protocols);
        if (cls->data()->protocols  &&  cls->data()->protocols != newprotos) {
            _free_internal(cls->data()->protocols);
        }
        cls->data()->protocols = newprotos;
 
        _free_internal(cats);
 
        // Update method caches and vtables
        flushCaches(cls);
        if (vtableAffected) flushVtables(cls);
    }
}
```

添加类的实例方法，又调用了**attachCategoryMethods**方法，进一步找到该方法原型：

```objective-c
static void 
attachCategoryMethods(class_t *cls, category_list *cats,
                      BOOL *inoutVtablesAffected)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);
 
    BOOL isMeta = isMetaClass(cls);
    method_list_t **mlists = (method_list_t **)
        _malloc_internal(cats->count * sizeof(*mlists));
 
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int i = cats->count;
    BOOL fromBundle = NO;
    while (i--) {
        method_list_t *mlist = cat_method_list(cats->list[i].cat, isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= cats->list[i].fromBundle;
        }
    }
 
    attachMethodLists(cls, mlists, mcount, NO, fromBundle, inoutVtablesAffected);
 
    _free_internal(mlists);
 
}
```

attachCategoryMethods把所有实例方法列表拼成一个大实例方法列表，然后调用**attachMethodLists**方法:

```objective-c
for (uint32_t m = 0;
             (scanForCustomRR || scanForCustomAWZ)  &&  m < mlist->count;
             m++)
        {
            SEL sel = method_list_nth(mlist, m)->name;
            if (scanForCustomRR  &&  isRRSelector(sel)) {
                cls->setHasCustomRR();
                scanForCustomRR = false;
            } else if (scanForCustomAWZ  &&  isAWZSelector(sel)) {
                cls->setHasCustomAWZ();
                scanForCustomAWZ = false;
            }
        }
 
        // Fill method list array
        newLists[newCount++] = mlist;   //添加的是category中的方法
    .
    .
    .
 
    // Copy old methods to the method list array
    for (i = 0; i < oldCount; i++) {
        newLists[newCount++] = oldLists[i];  //添加类原有的方法
    }
```

上面代码仅展示了**attachMethodLists**的重要部分，需要注意⚠️的是：

1️⃣category的方法没有替换原有类已有的方法，也就是说如果原有类和category都有methodA，那么category添加过后，类的方法列表中会有两个methodA

2️⃣category的方法在newLists（添加category方法后的类方法列表）的前面，而原来的类方法被放到newLists后面，这就是常说的category方法“覆盖”掉原来类的同名方法：原因是运行时候是按照方法列表newLists的顺序查找，只要找到对应的方法，就会返回，也就忽略了后面的同名方法。

#### 1.4.3 关联对象

上一节说到category能够为一个类添加属性，但是系统不会生成getter、setter方法，此时就需要通过runtime来进行关联对象操作。

通过关联对象添加属性本质上是使用类别进行扩展，通过添加getter和setter方法从而在访问时可以使用点语法进行方法。

关联对象时使用的C函数如下：

```objective-c
//为一个实例对象添加一个关联对象，由于是C函数只能使用C字符串，这个key就是关联对象的名称，value为具体的关联对象的值，policy为关联对象策略，与我们自定义属性时设置的修饰符类似
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy);
//通过key和实例对象获取关联对象的值
id objc_getAssociatedObject(id object, const void *key);
//删除实例对象的关联对象
void objc_removeAssociatedObjects(id object);
```

​	其中objc_AssociationPolicy的定义如下：

```objective-c
/**
 * Policies related to associative references.
 * These are options to objc_setAssociatedObject()
 */
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

可以看出这些关键词和property使用的修饰符含义相同。

举个🌰，为已有类NSArray添加一个关联对象。

```objective-c
//-----------MyClass+cate.h----------
#import "MyClass.h"

NS_ASSUME_NONNULL_BEGIN

@interface MyClass (cate)

@property (nonatomic,copy)NSString *name;

@end

NS_ASSUME_NONNULL_END
  
//-----------MyClass+cate.m----------
#import "MyClass+cate.h"
#import <objc/runtime.h>

@implementation MyClass (cate)

- (void)setName:(NSString *)name
{
    objc_setAssociatedObject(self,"name",name,OBJC_ASSOCIATION_COPY);
}

- (NSString*)name
{
    NSString *nameObject = objc_getAssociatedObject(self, "name");
    return nameObject;
}

@end
  
//------------------------main.m-----------------------------
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "MyClass.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        MyClass *obj = [[MyClass alloc]init];
        obj.name = @"jack";   //在MyClass类对象中可以访问到类别添加的属性了   
    }
    return 0;
}
```

从上面代码中可以看出，通过类别扩展setter和getter方法能够为一个已有的类添加属性。

### 1.5 Method swizzling

#### 1.5.1 实例方法底层

首先自定义一个类Person：

```objective-c
//------------------------Person.h-----------------------------
#import <Foundation/Foundation.h>

@interface Person : NSObject

@property (nonatomic, copy) NSString* name;
@property (nonatomic, assign) NSUInteger age;

- (instancetype)initWithName:(NSString*)name age:(NSUInteger)age;
- (void)showMyself;
- (void)helloWorld;

@end
  
//------------------------Person.m-----------------------------
#import "Person.h"
#import <objc/runtime.h>

@implementation Person

@synthesize name = _name;
@synthesize age = _age;

- (instancetype)initWithName:(NSString*)name age:(NSUInteger)age
{
    if(self = [super init])
    {
        _name = name;
        _age = age;
    }
    return self;
}
- (void)showMyself
{
    NSLog(@"My name is %@ I\'m %ld years old.",self.name,self.age);
}
- (void)helloWorld
{
    NSLog(@"Hello World");
}

@end
```

找到与实例方法相关的定义有：

```objective-c
struct _objc_method {
	struct objc_selector * _cmd;
	const char *method_type;
	void  *_imp;
};

static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[7];
} _OBJC_$_INSTANCE_METHODS_Person __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	7,
	{{(struct objc_selector *)"initWithName:age:", "@32@0:8@16Q24", (void *)_I_Person_initWithName_age_},
	{(struct objc_selector *)"showMyself", "v16@0:8", (void *)_I_Person_showMyself},
	{(struct objc_selector *)"helloWorld", "v16@0:8", (void *)_I_Person_helloWorld},
	{(struct objc_selector *)"name", "@16@0:8", (void *)_I_Person_name},
	{(struct objc_selector *)"setName:", "v24@0:8@16", (void *)_I_Person_setName_},
	{(struct objc_selector *)"age", "Q16@0:8", (void *)_I_Person_age},
	{(struct objc_selector *)"setAge:", "v24@0:8Q16", (void *)_I_Person_setAge_}}
};
```

一个实例方法在底层就是一个方法描述和一个C函数的具体实现，通过以下代码可以获取方法描述结构体：

```objective-c
//------------------------main.m-----------------------------
#import <Foundation/Foundation.h>
#import <objc/runtime.h>
#import "Person.h"
#import "MyClass.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool { 
        Person *p = [[Person alloc] initWithName:@"jasmine" age:18];
        [p showMyself];
        unsigned int count = 0;
        Method *methodList = class_copyMethodList([p class], &count);
        for(int i=0;i<count;i++)
        {
            SEL s = method_getName(methodList[i]);
            NSLog(@"%@",NSStringFromSelector(s));
            if([NSStringFromSelector(s) isEqualToString:@"helloWorld"])
            {
                IMP imp = method_getImplementation(methodList[i]);
                imp();
            }
        }
    }
    return 0;
}
```

通过**class_copyMethodList**方法可以获取相关类的所有实例方法列表methodList，然后通过method_getName方法获取成员变量_cmd，这是一个selector变量。NSStringFromSelector方法能够获取实例方法的名称，method_getImplementation可以获取实例方法的具体实现imp。

上述代码执行结果如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/6.png)
#### 1.5.2 Method swizzling

Method swizzling用于改变一个已经存在的selector的实现，使得能够在运行时通过改变selector在类的消息分发列表中的映射从而改变方法的调用。

Method swizzling的本质就是修改上一节中的方法描述结构体**_objc_method**，其中**struct objc_selector**成员变量**_cmd**是**selector**选择子，还有一个函数指针**_imp**指向实例方法的具体实现。

例如对于上面的Person类，要交换它的实例方法，有如下代码：

```objective-c
//------------------------main.m-----------------------------
#import <Foundation/Foundation.h>
//#import <objc/objc.h>
#import <objc/runtime.h>
#import "Person.h"
#import "MyClass.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person alloc] initWithName:@"jasmine" age:18];
        Method method1 = class_getInstanceMethod([p class], @selector(helloWorld));
        Method method2 = class_getInstanceMethod([p class], @selector(showMyself));
        method_exchangeImplementations(method1, method2);
        
        [p showMyself];
        [p helloWorld];
    }
    return 0;
}
```

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/7.png)
从运行结果可以看出两个实例方法的具体实现被交换了。上面的代码使用了函数**method_exchangeImplementations**：

```objective-c
/** 
 * Exchanges the implementations of two methods.
 * 
 * @param m1 Method to exchange with second method.
 * @param m2 Method to exchange with first method.
 * 
 * @note This is an atomic version of the following:
 *  \code 
 *  IMP imp1 = method_getImplementation(m1);
 *  IMP imp2 = method_getImplementation(m2);
 *  method_setImplementation(m1, imp2);
 *  method_setImplementation(m2, imp1);
 *  \endcode
 */
OBJC_EXPORT void method_exchangeImplementations(Method m1, Method m2) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
```

该函数用于交换两个方法的实现，也就是**struct objc_selector**中的函数指针**_imp**被交换了。交换前selector和imp的对应关系如下：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/8.png)
交换后，selector和imp的对应关系则变为：

![](https://github.com/skyjasmine/iOS-/blob/master/images/images(日志5)/9.png)
在实际应用中，可能会出现以下需求：我们想在一款iOS app中追踪每一个视图控制器被用户呈现了几次：可以通过在每个视图控制器的**viewDidAppear:**中添加追踪代码实现，但是这样会大量重复样板代码；也可以通过继承实现，但是这样又需要继承UIViewController、UITableViewController、UINavigationController等，这样也会比较繁琐。

经过这一节的学习，我们可以使用类别category来实现method swizzling.

```objective-c
#import "UIViewController+Tracking.h"
#import <objc/runtime.h>

@implementation UIViewController (Tracking)

+ (void)load
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken,^{
        Class class = [self class];
        
        SEL originalSelector  = @selector(viewWillAppear:);
        Method originMethod = class_getInstanceMethod(class, originalSelector);
        
        SEL swizzledSelector = @selector(xxx_ViewWillAppear:);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        
        if(didAddMethod)
        {
            class_replaceMethod(class, swizzledMethod, method_getImplementation(originMethod), method_getTypeEncoding(originMethod));
        }
        else
        {
            method_exchangeImplementations(originMethod, swizzledMethod);
        }
    });
}

- (void)xxx_ViewWillAppear:(BOOL)animated
{
    [self xxx_ViewWillAppear:animated];
    NSLog(@"ViewWillAppear %@", self);
}

@end
```

(void)myViewWillAppear:(BOOL)animated函数在函数内调用myViewWillAppear，正常情况下会出现递归调用，在交换方法实现后，myViewWillAppear：方法的实现已经被替换成了UIViewController-viewWillAppear：的实现，因此这里并不会产生递归调用。

使用类别category来实现method swizzling需要注意⚠️以下几点：

1️⃣**swizzling**应该只在**+load**中完成。因为load方法能够保证在类第一次被加载时候调用，这样可以保证一定会执行方法交换操作。

2️⃣**swizzling**应该只在**dispatch_once**中完成。**dispatch_once**可以确保代码只会被执行一次，就算在不同线程中也只会执行一次。

3️⃣记住给需要转换的所有方法加前缀以区别原生方法。**在交换方法实现后记得要调用原生方法的实现**。

### 1.6 weak实现机理

weak修饰符最主要的作用是为了**防止引用循环**，经常用于**block**和**delegate**。

weak不论是用作property修饰符还是用来修饰一个变量的声明，其作用都是**不增加新对象的引用计数**，被释放时也不会减少新对象的引用计数，同时在**新对象被销毁时**，**weak修饰的属性或变量均会被设置为nil**，这样可以防止野指针错误。

runtime是如何在实现在weak修饰的变量的对象在被销毁时自动置为nil：==runtime维护了一个hash表，对于weak修饰的对象会放入一个**hash表**中，用**weak指向的对象内存地址作为key**，当此对象的引用计数为0的时候会dealloc，如果weak指向的对象内存地址是a，那么就会以a为键在weak表中搜索，找到所有以a为键的weak对象，从而设置为nil==。

对下面几行代码通过Clang编译成C++：

```objective-c
int main(int argc, const char * argv[]) {
        NSObject *obj = [[NSObject alloc] init];
        id __weak obj1 = obj;
    return 0;
}
```

编译后的结果：

```objective-c
int main(int argc, const char * argv[]) {
        NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
        id __attribute__((objc_ownership(weak))) obj1 = obj;
    return 0;
}
```

#### 1.6.1 weak表

weak_table_t是一个全局weak引用的表，使用不定类型对象的地址作为key，用weak_entry_t类型结构体对象作为value。

```objective-c
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;          //保存了所有指向指定对象的weak指针
    size_t    num_entries;             
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

weak_entry_t是存储在weak表中的一个结构体，它负责维护和存储指向一个对象的所有弱引用hash表。定义如下：

```objective-c
/// The address of a __weak object reference
typedef objc_object ** weak_referrer_t;

/**
 * The internal structure stored in the weak references table. 
 * It maintains and stores
 * a hash set of weak references pointing to an object.
 * If out_of_line==0, the set is instead a small inline array.
 */
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_lne : 1;
            uintptr_t        num_refs : PTR_MINUS_1;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };
};
```

**objc_object**是weak_entry_t表中weak弱引用对象的范型对象的结构体结构。

#### 1.6.2 weak底层实现原理

在runtime源码中的NSObject.mm文件中找到了关于初始化和管理weak表的方法。

1️⃣初始化weak表方法

```objective-c
/** 
 * Initialize a fresh weak pointer to some object location. 
 * It would be used for code like: 
 *
 * (The nil case) 
 * __weak id weakPtr;
 * (The non-nil case) 
 * NSObject *o = ...;
 * __weak id weakPtr = o;
 * 
 * This function IS NOT thread-safe with respect to concurrent 
 * modifications to the weak variable. (Concurrent weak clear is safe.)
 *
 * @param location Address of __weak ptr. 
 * @param newObj Object ptr. 
 */
id
objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<false/*old*/, true/*new*/, true/*crash*/>
        (location, (objc_object*)newObj);
}
```

2️⃣存储weak对象的方法

```objective-c
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(HaveOld  ||  HaveNew);
    if (!HaveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (HaveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    if (HaveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}
```

* weak_unregister_no_lock旧对象解除注册操作

  该方法的作用是将旧对象在weak_table中解除weak指针的对应绑定。

* weak_register_no_lock新对象注册操作

  该方法把新对象进行注册操作，完成与对应的weak_table的绑定操作。

#### 1.6.3 weak释放为nil过程

一个对象的释放过程如下：

1.调用objc_release;

2.对象引用计数为0时，执行dealloc；

3.dealloc中调用_objc_rootDealloc函数，objc_rootDealloc中调用objc_dispose函数；

4.调用objc_destructInstance，最后调用objc_clear_deallocating.

objc_clear_deallocating方法原型如下：

```objective-c
void objc_clear_deallocating(id obj) 
{
    assert(obj);
    assert(!UseGC);

    if (obj->isTaggedPointer()) return;
    obj->clearDeallocating();
}

// Slow path of clearDeallocating() 
// for objects with indexed isa
// that were ever weakly referenced 
// or whose retain count ever overflowed to the side table.
NEVER_INLINE void
objc_object::clearDeallocating_slow()
{
    assert(isa.indexed  &&  (isa.weakly_referenced || isa.has_sidetable_rc));

    SideTable& table = SideTables()[this];
    table.lock();
    if (isa.weakly_referenced) {
        weak_clear_no_lock(&table.weak_table, (id)this);
    }
    if (isa.has_sidetable_rc) {
        table.refcnts.erase(this);
    }
    table.unlock();
}
```

对weak置nil的操作最终调用执行weak_clear_no_lock方法：

```objective-c
/** 
 * Called by dealloc; nils out all weak pointers that point to the 
 * provided object so that they can no longer be used.
 * 
 * @param weak_table 
 * @param referent The object being deallocated. 
 */
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```

从源码中可以看出，objc_clear_deallocating函数进行了如下操作：

1.从weak表中获取废弃对象的地址作为键值的记录；

2.将包含在记录中的所有附有weak修饰符变量的地址，赋值为nil；

3.将weak表中该记录删除；

4.从引用计数表中删除废弃对象的地址为键值的记录。

