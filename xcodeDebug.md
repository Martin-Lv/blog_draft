# xcode 解决EXC_BAD_ACCESS 或 SIGBAT的方法

点击procuct -> scheme -> edit scheme -> 在run的arguments 选项卡中添加两个environment Variables，并都设为YES：

####NSZombieEnabled

启动这个环境变量后控制台会打出类似这样的消息：

```
 -[NSArray addObject:]: message sent to deallocated instance 0x7179910

```

提示向已经释放的对象发送消息，这样就容易找到出问题的地方。


####MallocStackLogging

启动这个环境变量后，可以通过在控制台输入如下命令：

```
shell malloc_history pid address
```

其中address替换成在刚才的输出信息中打出的instance地址。

再次运行，程序崩溃时就会打出stack trace信息，可以通过这些信息定位问题的源头。

最后需要把这两个环境变量设为NO，因为开启会导致内存不会被释放。

