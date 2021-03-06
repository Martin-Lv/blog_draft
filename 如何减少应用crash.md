#如何减少应用crash


##代码中可能导致crash的情况

###swift空指针

一切对Optional类型的操作都要判断是否为空，即使按代码逻辑不可能为空。

原因有二：

1. 逻辑可能会出问题，即使是很小的概率也会使crash率升高。
2. 代码逻辑在迭代中很可能会变动，而每次都检查是否有相关的Optional类型变量从不可能为空变成可能为空，是不现实的。

另外，存在Swift和Objective-C互相调用时，有条件就尽量给Objective-C接口加上Nullability标志。这样Swift调用时可以清楚地知道返回的对象是否可能为空，避免对隐式解包类型忘记检查导致的crash。

除了swift空指针，其他可能为空的情况也需要考虑。比如系统相册中没有图片，网络请求返回空数据等。向OC空对象发送消息是安全的，但传递空参数是不一定安全的。除非空对象是逻辑的一部分，否则永远要做空判断。

###Swift有符号数转无符号数

在Swift中将一个有符号数（`Int`和各种浮点数）转换成无符号数时要当心，如果有符号数的数值小于0，比如下面的语句：

```
let n = UInt(a) //a = -1
```

程序会直接崩溃，而不是像C语言一样数值溢出。Swift这样设计大概是基于“让程序尽早崩溃”的原则，毕竟崩溃比溢出导致的奇怪问题更容易定位和修复。因此我们写代码时，遇到有符号数转换为无符号数的情况要当心。

有一个简单的办法避免这个问题：不使用无符号数，即使数值不可能为负。

当然，如果你要存储超过了`Int.max`的数，或者调用的API用了无符号数，那就没办法了，老老实实做负值检测吧。

###数组越界

安全的做法是对数组下标访问时都先进行越界判断。同样不能完全依赖代码的逻辑。除非是UITableViewDataSource这种逻辑非常清楚稳定的代码，在没有其他线程修改数据源的情况下可以不用检查。

###Objective-C集合类型枚举时修改异常

对可变的OC集合类型，比如NSMutableArray，如果在枚举时另一个线程修改了它，就会引发异常，从而crash。

安全的做法是，如果一个集合类型对象是可变的，先复制它，然后在复制的对象上遍历。

另外，对外暴露的属性接口永远不应该是一个可变的集合类型。如果一个内部的NSMutableArray需要以NSArray的声明暴露给外部，那么应该暴露为copy，且在getter中复制一份返回。因为很有可能调用方在遍历它的时候，它被修改了。

对于看不到实现的集合类型接口，有可能作者没有做这种保护。安全的做法是自己做一下复制。

###NSNotification

如果一个对象注册为一个NSNotification的监听者，而在dealloc之前没有取消注册，那么这个NSNotification被发送时程序就会crash。

避免的方法：

####永远只在init方法中注册，在dealloc方法中取消注册

这样可以确保只注册一次，且对象销毁时取消注册。

取消注册时直接使用

```
[[NSNotificationCenter defaultCenter] removeObserver:self];
```

取消全部监听，避免版本迭代时忘记更新取消注册的代码。

注册两次会导致回调函数被调用两次，应该避免。

如果一个对象需要只在生命周期的一部分时候监听NSNotification（比如UIViewController），那么尝试拆分对象，建立专门处理NSNotification的对象，需要监听时创建，不再需要监听时销毁。

####永远在主线程发送NSNotification

这样可以一劳永逸避免NSNotification发送和对象销毁同时发生的竞态条件问题。

疑问：对象的销毁不一定在主线程。需要确保对象指针置空的代码都在主线程？

###可能会抛出异常的API

使用系统API时注意文档中是否提到传参不当时会抛出异常。调用时加保护判断避免这种情况发生。

###多线程

硬件越来越强大，所以尽量所有代码都在主线程运行，这样可以完美避免多线程带来的各种问题。直到遇到性能瓶颈时再使用GCD或者NSOperationQueue做优化。

不得不在后台执行代码时，尽量将这个部分和其他部分隔离。例如需要在后台接收json数据并更新时，在后台生成中间对象，中间对象传送给主线程，将里面的内容更新到model类，然后丢弃。这样中间对象是不可变的，model类只在主线程进行修改。

可变对象只在主线程做修改，使用中间对象隔离多线程的操作。



###多线程同时操作C++对象

对C++代码进行调用时注意接口是否是线程安全的。如果不是，必须100%遵循接口的多线程调用约定。比如必须在主线程调用、一组接口必须在同一线程调用等。

##修复crash的技巧

在xcode中调试iOS app时，程序crash有时不会断在导致crash的代码处，而是断在main函数中，无从知道是哪一句代码导致了crash。

针对某些特定的crash原因，有一些办法可以定位到导致crash的代码。

###使用NSZombie解决 EXC\_BAD\_ACCESS 或 SIGABORT

点击procuct -> scheme -> edit scheme -> 在 run 的 arguments 选项卡中添加一个 environment Variables，并设为 YES：

`NSZombieEnabled`

启动这个环境变量后，对象应该被释放时不会真的被释放，而是被标记为一个Zombie对象。这样就能知道是向哪个已释放的对象发送消息导致了crash。控制台会打出类似这样的消息：

```
 -[NSArray addObject:]: message sent to deallocated instance 0x7179910

```

提示向已经释放的对象发送消息，这样就容易找到出问题的地方。

注意调试完毕后需要把这个环境变量设为NO，因为开启会导致对象不被释放，内存占用会一直增长。

###解决uncaught exception

如果程序崩溃时提示uncaught exception，可以加一个全局的异常断点。这样当程序抛出没有被捕获的异常时程序就会断掉。

在左侧栏的断点一栏中，点击左下角的“+”号，选择exception breakpoint即可。另外可以设置断点在异常抛出时还是异常被捕获时，以及针对Objective-C 异常还是C++异常。

###用Xcode UI Test代替手工来自动测试，发现crash

用UI Test录制各种操作路径，在不同的手机上重复跑测试，尽量让潜在的crash出现


##工作习惯

记录每个高概率crash发现，重现和修复的过程。

每个版本修复crash工作尽量放在开发周期的前期，测试通过后千万不能再做“看上去不会出问题”的修改。

写新功能，重构，优化，改bug，一次只做一件事。同时进行很容易思路乱掉顾此失彼，引入新的bug。