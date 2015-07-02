# xcode crash调试策略

在xcode中调试iOS app时，程序crash时往往不会断在导致crash的代码处，而是断在main函数中，无从知道是哪一句代码导致了crash。

针对某些特定的crash原因，有一些办法可以定位到导致crash的代码。

##解决EXC_BAD_ACCESS 或 SIGBAT

点击procuct -> scheme -> edit scheme -> 在run的arguments 选项卡中添加一个environment Variables，并设为YES：

`NSZombieEnabled`

启动这个环境变量后，对象应该被释放时不会真的被释放，因此就能知道是向哪个已释放的对象发送消息导致了crash。控制台会打出类似这样的消息：

```
 -[NSArray addObject:]: message sent to deallocated instance 0x7179910

```

提示向已经释放的对象发送消息，这样就容易找到出问题的地方。

最后需要把这个环境变量设为NO，因为开启会导致对象不被释放，内存占用会一直增长。

##解决uncaught exception

如果程序崩溃时提示uncaught exception，可以加一个全局的异常断点。这样当程序抛出没有被捕获的异常时程序就会断掉。

在左侧栏的断点一栏中，点击左下角的“+”号，选择exception breakpoint即可。另外可以设置断点在异常抛出时还是异常被捕获时，以及针对Objective-C 异常还是C++异常。


