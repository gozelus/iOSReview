<a href='#1'>一.objc对象内存模型</a>

<a href='#2'>二.Runtime</a>

<a href='#3'>三.Category</a>

  - <a href='#3-1'>3.1 Category简介</a>
  - <a href='#3-2'>3.2 Category类比extension</a>
  - <a href='#3-3'>3.3 Category内存结构</a>
  - <a href='#3-4'>3.4 Category加载</a>
  - <a href='#3-5'>3.5 Category中的Load方法</a>
  - <a href='#3-6'>3.6 Category与关联对象</a>

<a href='#4'>四. KVO 原理</a>

<a href='#5'>五. iOS App 启动过程 </a>


<a href='#6'>六. Block原理</a>

   - <a href='#6-1'> Block 基本原理</a>
   - <a href='#6-1'> __block 关键字捕获原理 </a>

<a href='#7'>七. GCD</a>

<a href='#8'>八. Runloop </a>

   - <a href='#8-1'> Runloop基本原理 </a>
   - 


<a href='#9'>九. ARC </a>

   - <a href='#9-1'> ARC 基本原理 </a>
   - <a href='#9-2'> autorelease pool 原理</a>

<h2 id='1'> 一、对象内存模型 </h2>

- isa指针的作用

- 对象的isa指向类对象，类对象的isa指向元类，元类的isa指向根元类，根元类的isa指向自己。

- 类对象的`superClass`指针指向父类对象，直到指向根类对象，根类对象的`superClass`指向`nil`，元类也如此，直到根元类，根元类的`superClass`指向根类对象

- 对象的内存分布
	- 对象的变量表
	- 对象大小
	- 对象方法表
	- cache?
	- 协议
	- char *name?
	
		```c++
		struct objc_class { 
		    Class isa  OBJC_ISA_AVAILABILITY;
		
		#if !__OBJC2__
		    Class super_class                                        OBJC2_UNAVAILABLE;
		    const char *name                                         OBJC2_UNAVAILABLE;
		    long version                                             OBJC2_UNAVAILABLE;
		    long info                                                OBJC2_UNAVAILABLE;
		    long instance_size                                       OBJC2_UNAVAILABLE;
		    struct objc_ivar_list *ivars                             OBJC2_UNAVAILABLE;
		    struct objc_method_list **methodLists                    OBJC2_UNAVAILABLE;
		    struct objc_cache *cache                                 OBJC2_UNAVAILABLE;
		    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
		#endif
		
		} OBJC2_UNAVAILABLE;
		/* Use `Class` instead of `struct objc_class *` */
	
		```
		
<h2 id='2'> Runtime </h2>

1. 消息接收
	- 编译时间确定接受到的消息，运行时间通过`@selector`找到对应的方法。
	- 消息接受者如果能直接找到`@selector`则直接执行方法，否则转发消息。若最终找不到，则运行时崩溃。
2. 术语
	- `SEL`
		- 方法名的C字符串，用于辨别不同的方法。
		- 用于传递给`objc_msgSend(self, SEL)`函数
	- `Class`
		- `Class`是指向类对象的指针，继承于`objc_object`
	- ``categoty``
		- ``categoty``是结构体``categoty`_t`的指针：`typedef struct `categoty`_t *`categoty`;
`
		- ``categoty``是在`app`启动时加载镜像文件，通过`read_imgs`函数向类对象中的`class_rw_t`中的某些分组中添加指针，完成``categoty``的属性添加
	- `Method`
		- 方法包含以下
			- `IMP`:函数指针，方法的具体实现
			- `types`:`char*`，函数的参数类型，返回值等信息
			- `SEL`:函数名
3. 消息转发
	- `objc_msgSend()`函数并不返回数据，而是它转发消息后，调用了相关的方法返回了数据。
	- 整个流程: 
	       1. 检测这个 `selector `是不是要忽略的。比如 `Mac OS X `开发，有了垃圾回收就不理会 `retain`, `release` 这些函数了。
	       2. 检测这个 `target` 是不是 `nil` 对象。`ObjC` 的特性是允许对一个` nil `对象执行任何一个方法不会 Crash，因为会被忽略掉。
          3. 如果上面两个都过了，那就开始查找这个类的 `IMP`，先从 `cache `里面找，完了找得到就跳到对应的函数去执行。
          4. 如果 `cache` 找不到就找一下方法分发表。
          5. 如果分发表找不到就到超类的分发表去找，一直找，直到找到`NSObject`类为止。
          6. 进入动态解析:`resolveInstanceMethod:`和`resolveClassMethod:`方法
          7. 若上一步返回`NO`,进入重定向：`- (id)forwardingTargetForSelector:(SEL)aSelector`和`+ (id)forwardingTargetForSelector:(SEL)aSelector `
          8. 若上一步返回的对象或者类对象仍然没能处理消息或者返回`NO`，进入消息转发流程：`forwardInvocation`
    -  重定向和消息转发都可以用于实现**多继承**
4. `Objective-C Associated Objects`关联对象
    - 在 `OS X 10.6 `之后，`Runtime`系统让`Objc`支持向对象动态添加变量。涉及到的函数有以下三个：

     ```objc
     void objc_setAssociatedObject ( id object, const void *key, id value, objc_AssociationPolicy policy );
     id objc_getAssociatedObject ( id object, const void *key );
     void objc_removeAssociatedObjects ( id object );
     ```
    - 这些方法以键值对的形式动态地向对象添加、获取或删除关联值。其中关联政策是一组枚举常量：

     ```c++
     enum {
           OBJC_ASSOCIATION_ASSIGN  = 0,
           OBJC_ASSOCIATION_RETAIN_NONATOMIC  = 1,
           OBJC_ASSOCIATION_COPY_NONATOMIC  = 3,
           OBJC_ASSOCIATION_RETAIN  = 01401,
           OBJC_ASSOCIATION_COPY  = 01403
     };
     ```
5. `Method Swizzling` 方法混淆
	- 作用是修改`SEL`对应的`IMP`指针
	- 用于`debug`，避免数组越界等问题.
 

## ARC
1. 引用计数如何存储？
	- 有些对象如果支持使用`TaggedPointer`，苹果会直接将其指针值作为引用计数返回；如果当前设备是 64 位环境并且使用 `Objective-C 2.0`，那么“一些”对象会使用其 `isa` 指针的一部分空间来存储它的引用计数；否则 `Runtime` 会使用一张**散列表**来管理引用计数。
	- 优化后的`isa`指针不仅仅包括常见的对象信息，还包括了如：
		- 对象是否在析构
		- 对象是否拥有`weak`对象
		- 对象是否包含`associated object`等等

		这些信息会把帮助对象的析构，使得对象析构更快。
		
<h2 id='3'> Categoty </h2>

推荐阅读：[美团技术博客：深入理解Objective-C：Category
](https://tech.meituan.com/DiveIntoCategory.html)

<h3 id = '3-1'> categoty 简介</h3>

- ``categoty``开头动态地给类添加属性及方法，通过这个特性，使得`objc`可以模拟多继承。也可以把一个类拆分到各个文件中，使用方可以按需加载。也可以多人协作共同完成一个类。

<h3 id = '3-2'> categoty 类比 extension </h3>

- `extension`看起来很像一个匿名的``categoty``，但是`extension`和有名字的``categoty``几乎完全是两个东西。 `extension`在编译期决议，它就是类的一部分，在编译期和头文件里的`@interface`以及实现文件里的`@implement`一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡。`extension`一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加`extension`，**所以你无法为系统的类比如`NSString`添加`extension`**。

- 但是``categoty``则完全不一样，**它是在运行期决议的**。
就``categoty``和`extension`的区别来看，我们可以推导出一个明显的事实，`extension`可以添加实例变量，而``categoty``是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）。

<h3 id = '3-3'> categoty 内存结构 </h3>

```c++
typedef struct `categoty`_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} `categoty`_t;
```

我们知道，所有的`OC`类和对象，在`runtime`层都是用`struct`表示的，``categoty``也不例外，在`runtime`层，``categoty``用结构体`categoty_t`（在`objc-runtime-new.h`中可以找到此定义），它包含了

1. 类的名字（name）
2. 类（cls）
3. `categoty`中所有给类添加的实例方法的列表（`instanceMethods`）
4. `categoty`中所有添加的类方法的列表（`classMethods`）
5. `categoty`实现的所有协议的列表（`protocols`）
6. `categoty`中添加的所有属性（`instanceProperties`）

<h3 id = '3-4'> category 的加载 </h3>

`category`是在`runtime`进入入口时被添加进类的，`runtime`通过遍历编译后产生的`DATA`端上的`category_t`数组动态地将内容添加到类上。

需要注意的时，`category`中的方法其实并没有覆盖原类中的同名方法，只是单纯地添加到方法列表中而已。但是由于`objc`在方法调用中查找到类对象中的方法列表时，只要找到了第一个对应消息中的选择子的方法，就认为是找到了对应的方法并调用。所以形成了`category`中的方法会覆盖原类中的同名方法的假象。

根据这个原理，我们可以得知，如果有多个类别中同时都复写了类中的某个方法，那么最终调用的是最后被加载的`category`，也就是最后被编译的`category`文件。

<h3 id = '3-5'>  category  中 Load 方法</h3>

`category`中也可以复写原类中的`+Load`方法，因为类的`Load`方法永远发生在`category`的`Load`方法之前。如果多个`category`都复写了`Load`方法，则`category`的`Load`执行顺序取决于对应`category`文件的编译顺序。

<h3 id = '3-6'> category 与关联对象 </h3>

由于`category`中无法动态添加实例变量，所以可以通过关联对象的手段为对象增加实例变量:

```objc
#import "MyClass+Category1.h"
#import <objc/runtime.h>

@implementation MyClass (Category1)

+ (void)load
{
    NSLog(@"%@",@"load in Category1");
}

- (void)setName:(NSString *)name
{
    objc_setAssociatedObject(self,
                             "name",
                             name,
                             OBJC_ASSOCIATION_COPY);
}

- (NSString*)name
{
    NSString *nameObject = objc_getAssociatedObject(self, "name");
    return nameObject;
}

@end
```
关联对象存储在一张全局的`map`里，其中`map`的`key`是被关联的对象的指针地址，该`map`的`value`存储的是另外一张`map`，此处称为`AssociationsHashMap`，该`AssociationsHashMap`的kv分别是设置关联对象时的kv。

<h3 id = '4'> KVO </h3>

- 原理：
	1. `isa swizzling`方法，通过`runtime`动态创建一个中间类，继承自被监听的类。
	2. 使原有类的`isa`指针指向这个中间类，同时重写改类的`Class`方法，使得该类的`Class`方法返回自己而不是`isa`的指向。
	3. 复写中间类的对应被监听属性的`setter`方法，调用添加进来的方法，然后给当前中间类的父类也就是原类的发送`setter`消息。
- 自定义KVO:
	1. 通过给`NSObject`添加分类的方法，添加新的对象方法:

	```objc
	   #import <Foundation/Foundation.h>
	   @interface NSObject (ACoolCustom_KVO)
	   /**
	    *    自定义添加观察者
	    *
	    *    @param oberserver 观察者
	    *    @param keyPath    要观察的属性
	    *    @param block      自定义回调
	    */
	   - (void)zl_addOberver:(id)oberserver
	             forKeyPath:(NSString *)keyPath
	                  block:(void(^)(id oberveredObject,NSString *keyPath,id newValue,id oldValue))block;
	```
	
	2. 动态创建中间类，	更改原有类的`isa`指向，同时重写中间类的`Class`方法:
	
	```objc
	Class originalClass = object_getClass(self);
    //创建中间类 并使其继承被监听的类
    Class kvoClass = objc_allocateClassPair(originalClass, kvoClassName.UTF8String, 0);
    //向runtime动态注册类
    objc_registerClassPair(kvoClass);
    object_setClass(self, kvoClass);
    //替换被监听对象的class方法...
    //原始类的class方法的实现
    //原始类的class方法的参数等信息
    Method clazzMethod = class_getInstanceMethod(originalClass, @selector(class));
    const char *types = method_getTypeEncoding(clazzMethod);
    class_addMethod(kvoClass, @selector(class), (IMP)new_class, types);
	```
	
	3. 利用`runtime`给对象动态增加关联属性保存外部传进来的回调`block`:

	```objc
	objc_setAssociatedObject(self, kObjectPropertyKey ,block, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
	```
	
	4. 利用`runtime`替换中间类的`setter`方法，在新的方法中，调用`block`后再向父类发送原消息:
	
	```objc
	// 利用函数指针强制转换
    void (*objc_msgSendSuperCasted)(void *, SEL, id) = (void *)objc_msgSendSuper;
    // 给父类 发送原消息
    objc_msgSendSuperCasted(&superclass, _cmd, newValue);
    // 调用block
    ZLObservingBlock block = objc_getAssociatedObject(self, kObjectPropertyKey);
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        block(self, nil, newValue);
    });
	```
		
<h2 id='6'> Block </h2>

<h3 id = '6-1'> Block 基本原理 </h3>

```objc
int main() {
    void (^blk)(void) = ^{
        (printf("hello world!"));
    };
    blk();
    return 0;
}
```

如上，通过`clang`转换为`cpp`源码，截取关键部分:

```c++
// block 的真面目
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

// ..... 一大坨无关代码

// 通过这个结构体包装block，用于快速构造block
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

        (printf("hello world!"));
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main() {
    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
```
比较核心的是以下内容:`__block_impl`结构体，`__mian_block_impl_0`结构体，`__main_block_func_0`函数，`&_NSConcreteStackBlock`，`__main_block_desc_0`结构体。

- `__block_impl`结构体:

	该结构体定义在源文件上方，其中`isa`指针指向当前`block`对象类对象，`FuncPtr`指向`block`保存的函数指针。

- `__main_block_desc_0`结构体:

	- 用于保存`__main_block_impl_0 `结构体大小等信息。
	
- `__main_block_func_0`静态函数:
	
	- 用于存储当前`block`的代码块。

- `__main_block_impl_0`结构体:

	该结构体包装了`__block_impl`结构体，同时包含`__main_block_desc_0`结构体。对外提供一个构造函数，构造函数需要传递函数指针(`__main_block_func_0`静态函数)、`__main_block_desc_0`实例。
	
由此我们可知，上述一个简单的`block`定义及调用过程被转换为了：

1. 定义`block`变量相当于调用`__main_block_impl_0`构造函数，通过函数指针传递代码块进`__main_block_impl_0`实例。

2. 构造函数内部，将外部传递进来的`__main_block_func_0`函数指针，设置内部实际的`block`变量(`__block_impl`类型的结构体)的函数指针。

3. 调用`block`时，取出`__mian_block_impl_0`类型结构体中的`__block_impl`类型的结构体的函数指针(`__main_block_func_0`)并调用。

至此，一个简单的`block`原理描述完毕。
	
<h3 id='6-2'> __block 捕获原理 </h3>

`block`内部可以直接使用外部变量，但是在不加`__block`修饰符的情况下，是无法修改的。比如下面这段代码：

```objc
int main() {
    int a = 1;
    void (^blk)(void) = ^{
        printf("%d", a);
    };
    blk();
    return 0;
}
```

经过`cpp`重写后:

```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int a = __cself->a; // bound by copy

        printf("%d", a);
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main() {
    int a = 1;
    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
```
经过<a href='#6-1'>上一节的讨论</a>，我们知道代码块是由`__main_block_impl_0`构造函数传通过函数指针传递进来的。在这节的例子中，可以发现构造函数中多了一个`int _a`参数。同时`__main_block_impl_0`结构体也多了一个`int a`属性用于保存`block`内部使用的变量`a`。

由于构造函数的参数为整形，在`c++`中，函数的形参为值拷贝，也就是说`__main_block_impl_0`结构体中的属性`a`，是外部`a`变量的拷贝。在代码块内部(也就是`__main_block_func_0 `函数)我们通过`__cself`指针拿到`a`变量。故我们在代码块中是无法修改`a`变量的，同时如果外部`a`变量被修改了，那么`block`内部也是无法得知的。

如果想要在内部修改`a`变量，可以通过`__block`关键字:

```objc
int main() {
    __block int number = 1;
    void (^blk)(void) = ^{
        number = 3;
        printf("%d", number);
    };
    blk();
    return 0;
}
```
```c++
struct __Block_byref_number_0 {
  void *__isa;
__Block_byref_number_0 *__forwarding;
 int __flags;
 int __size;
 int number;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_number_0 *number; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_number_0 *_number, int flags=0) : number(_number->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_number_0 *number = __cself->number; // bound by ref

        (number->__forwarding->number) = 3;
        printf("%d", (number->__forwarding->number));
    }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->number, (void*)src->number, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->number, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main() {
    __attribute__((__blocks__(byref))) __Block_byref_number_0 number = {(void*)0,(__Block_byref_number_0 *)&number, 0, sizeof(__Block_byref_number_0), 1};
    void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_number_0 *)&number, 570425344));
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
    return 0;
}
```
可以看到整形`number`变量变成了`__Block_byref_number_0`结构体实例，在`__main_block_impl_0`构造函数中，将该实例的指针传递进来。`__Block_byref_number_0`结构体中，通过`__forwarding`指向自己，整形`number`存储具体的值。

上节提到，构造函数中的参数是值拷贝，故此处代码块内部拿到的指针拷贝一样可以操作外部的`__Block_byref_number_0`结构体中的值，通过这种方式实现了`block`内部修改外部值。

<h3 id='6-3'> block 引起的循环引用原理 </h3>


<!--<h2 id='7'> GCD </h2>-->

<h2 id='8'> RunLoop </h2>

<h3 id='8-1'> Runloop 基本原理 </h3>

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑是这样的：

```c++
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
```
在`iOS`中，`Runloop`与线程是一一对应的关系，这种关系存在一张全局字典里：

```c++
/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;
 
/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
    OSSpinLockLock(&loopsLock);
    
    if (!loopsDic) {
        // 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
        loopsDic = CFDictionaryCreateMutable();
        CFRunLoopRef mainLoop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
    }
    
    /// 直接从 Dictionary 里获取。
    CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));
    
    if (!loop) {
        /// 取不到时，创建一个
        loop = _CFRunLoopCreate();
        CFDictionarySetValue(loopsDic, thread, loop);
        /// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
        _CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
    }
    
    OSSpinLockUnLock(&loopsLock);
    return loop;
}
 
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```
苹果是不允许直接创建`Runloop`的，只能通过:

```c++
CFRunLoopRef CFRunLoopGetMain() {
    return _CFRunLoopGet(pthread_main_thread_np());
}
 
CFRunLoopRef CFRunLoopGetCurrent() {
    return _CFRunLoopGet(pthread_self());
}
```

这两个函数来获得当前或者某个线程的`Runloop`。根据上面的源码可以得出结论：
> `Runloop`创建是发生在第一次获取之前，若从未获取，则永远不会创建。


<h2 id='9'> ARC </h2>

<h3 id='9-1'> ARC 基本原理 </h3>

使用ARC，开发者不再需要手动的`retain/release/autorelease`. 编译器会自动插入对应的代码，再结合`Objective-C`的`runtime`，实现自动引用计数。

如:

```objc
NSObject * obj;
{
    obj = [[NSObject alloc] init]; //引用计数为1
}
NSLog(@"%@",obj);
```

```objc
NSObject * obj;
{
    obj = [[NSObject alloc] init]; //引用计数为1
    [obj relrease]
}
NSLog(@"%@",obj);
```
其中，`retain`,`release`的方法实现原理：

```c++
- (id)retain {
    return ((id)self)->rootRetain();
}
inline id objc_object::rootRetain()
{
    if (isTaggedPointer()) return (id)this;
    return sidetable_retain();
}

#if SUPPORT_MSB_TAGGED_POINTERS
#   define TAG_MASK (1ULL<<63)
#else
#   define TAG_MASK 1

inline bool 
objc_object::isTaggedPointer() 
{
#if SUPPORT_TAGGED_POINTERS
    return ((uintptr_t)this & TAG_MASK);
#else
    return false;
#endif
}
```
由`retain`方法的实现可知，有些对象如果支持使用 TaggedPointer，苹果会直接将其指针值作为引用计数返回；如果当前设备是 64 位环境并且使用 `Objective-C 2.0`，那么“一些”对象会使用其 `isa` 指针的一部分空间来存储它的引用计数；否则 `Runtime`会使用一张散列表来管理引用计数。

```c++
id objc_object::sidetable_retain()
{
    //获取table
    SideTable& table = SideTables()[this];
    //加锁
    table.lock();
    //获取引用计数
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
         //增加引用计数
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    //解锁
    table.unlock();
    return (id)this;
}
```

与`retain`类似，`release`方法也是如此的逻辑，只是增加了若引用计数为0，则销毁对象:

```c++
SideTable& table = SideTables()[this];
bool do_dealloc = false;
table.lock();
//找到对应地址的
RefcountMap::iterator it = table.refcnts.find(this);
if (it == table.refcnts.end()) { //找不到的话，执行dellloc
    do_dealloc = true;
    table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
} else if (it->second < SIDE_TABLE_DEALLOCATING) {//引用计数小于阈值，dealloc
    do_dealloc = true;
    it->second |= SIDE_TABLE_DEALLOCATING;
} else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
//引用计数减去1
    it->second -= SIDE_TABLE_RC_ONE;
}
table.unlock();
if (do_dealloc  &&  performDealloc) {
    //执行dealloc
    ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
}
return do_dealloc;
```

再介绍下 `SideTable` 这个类，它用于管理引用计数表和 `weak` 表，并使用 `spinlock_lock` 自旋锁来防止操作表结构时可能的竞态条件。它用一个 64*128 大小的 `uint8_t` 静态数组作为 `buffer` 来保存所有的 `SideTable` 实例。并提供三个公有属性。

```c++
spinlock_t slock;//保证原子操作的自选锁
RefcountMap refcnts;//保存引用计数的散列表
weak_table_t weak_table;//保存 weak 引用的全局散列表
```
`weak` 表的作用是在对象执行 `dealloc` 的时候将所有指向该对象的 `weak` 指针的值设为 `nil`，避免悬空指针。这是 `weak` 表的结构：

```c++
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```

苹果使用一个全局的 `weak` 表来保存所有的 `weak` 引用。并将对象作为键，`weak_entry_t` 作为值。`weak_entry_t` 中保存了所有指向该对象的 `weak` 指针。


<h3 id='9-2'> autoreleasepool 原理</h3>

`Autorelease`对象是在当前的`runloop`迭代结束时释放的，而它能够释放的原因是系统在每个`runloop`迭代中都加入了自动释放池`Push`和`Pop`

`Autorelease`对象的各种方法都是对类`AutoreleasePoolPage`的封装，其中`AutoreleasePoolPage`类结构如下:

![Autorelease结构](https://github.com/Rabbbbbbit/iOSReview/blob/master/imgs/51530583gw1elj2ugt21wj20f109m3zl.jpg?raw=true)

 - `AutoreleasePool`并没有单独的结构，而是由若干个`AutoreleasePoolPage`以双向链表的形式组合而成（分别对应结构中的parent指针和child指针）
- `AutoreleasePool`是按线程一一对应的（结构中的`thread`指针指向当前线程）
- `AutoreleasePoolPage`每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存`autorelease`对象的地址
- 上面的`id *next`指针作为游标指向栈顶最新add进来的`autorelease`对象的下一个位置
- 一个`AutoreleasePoolPage`的空间被占满时，会新建一个`AutoreleasePoolPage`对象，连接链表，后来的`autorelease`对象在新的page加入

所以，若当前线程中只有一个`AutoreleasePoolPage`对象，并记录了很多`autorelease`对象地址时内存如下图：
![](https://github.com/Rabbbbbbit/iOSReview/blob/master/imgs/51530583gw1elj5gvphtqj20dy0cx756.jpg?raw=true)

图中的情况，这一页再加入一个`autorelease`对象就要满了（也就是`next`指针马上指向栈顶），这时就要执行上面说的操作，建立下一页`page`对象，与这一页链表连接完成后，新page的next指针被初始化在栈底（`begin`的位置），然后继续向栈顶添加新对象。

所以，向一个对象发送`- autorelease`消息，就是将这个对象加入到当前`AutoreleasePoolPage`的栈顶`next`指针指向的位置

每当进行一次`objc_autoreleasePoolPush`调用时，`runtime`向当前的`AutoreleasePoolPage`中`add`进一个哨兵对象，值为0（也就是个`nil`），那么这一个`page`就变成了下面的样子：

![](https://github.com/Rabbbbbbit/iOSReview/blob/master/imgs/51530583gw1elj5z7hawej20ji0dewff.jpg?raw=true)

`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop`(哨兵对象)作为入参，于是：

- 根据传入的哨兵对象地址找到哨兵对象所处的`page`
- 在当前`page`中，将晚于哨兵对象插入的所有`autorelease`对象都发送一次`- release`消息，并向回移动`next`指针到正确位置
- 从最新加入的对象一直向前清理，可以向前跨越若干个`page`，直到哨兵所在的`page`


refers：[深入理解Objective C的ARC机制](http://www.cocoachina.com/ios/20170427/19109.html)、[Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)、[黑幕背后的Autorelease
](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)


- 属性关键字
- `categoty` Extension
- @synthesize和@dynamic
- 属性
- 多态
- @dynamic关键字
- YYModel基本原理
- AFN保活
- timer
- 锁	1

## 项目
- weex原理
- vue
- nodejs EventLoop

## 操作系统
- 线程进程
- 锁

## 计算机网络
- http
	- https
	- get/post

- tcp/ip
	
