#构造异步NSOperation实现异步操作的线程同步
在多线程编程时，NSOperationQueue是线程同步的利器。

一个简单的场景：某个类内部不是线程安全的，同一时刻只能有一个线程操作这个类。只要将相关操作放在串行NSOpeartionQueue中即可。

我们写一个简单的类来模拟一下：

```
class TestClass {
    func method1(){
        print("method 1 begin")
        for _ in 0 ... 100000 {
            continue
        }
        print("method 1 end")
    }
    
    func method2(){
        print("method 2 begin")
        for _ in 0 ... 100000 {
            continue
        }
        print("method 2 end")
    }
}

```
测试代码：

```
let operationQueue = NSOperationQueue()
operationQueue.maxConcurrentOperationCount = 1

operationQueue.addOperationWithBlock { () -> Void in
    unsafeObject.method1()
}
operationQueue.addOperationWithBlock { () -> Void in
    unsafeObject.method2()
}

let runloop = NSRunLoop.currentRunLoop()
while runloop.runMode(NSDefaultRunLoopMode, beforeDate: NSDate.distantFuture()){
    continue
}

```

这里我们启动一个自己的runloop，避免主线程执行完之后整个程序退出，`operationQueue`中的代码来不及执行。
输出结果如下：

```
method 1 begin
method 1 end
method 2 begin
method 2 end
```

可以看到，两个方法严格地顺序执行。

但有些情况下，我们调用的接口是异步的。在NSBlockOperation中调用后，马上就返回了，无法起到线程同步的作用。

给`TestClass`增加一个异步方法：

```
let queue = dispatch_queue_create("UnsafeClass_background_queue", DISPATCH_QUEUE_CONCURRENT)

func asyncMethod(done:()->Void){
    print("async method begin")
    dispatch_async(queue) { () -> Void in
        for _ in 0 ... 100000 {
            continue
        }
        print("async method end")
        done()
    }
    return
}
```

测试代码中增加异步方法的调用：

```
operationQueue.addOperationWithBlock { () -> Void in
    object.method1()
}
operationQueue.addOperationWithBlock({ () -> Void in
    object.asyncMethod({ () -> Void in
    })
})
operationQueue.addOperationWithBlock { () -> Void in
    object.method2()
}
```

程序输出如下：

```
method 1 begin
method 1 end
async method begin
method 2 begin
async method end
method 2 end
```

可以看到`method2()`和`asyncMethod()`是并发执行的。这是很自然的，我们添加的第二个`NSOperation`在执行完`asyncMethod()`的调用后就返回了，不会等它`dispatch_async`到其他dispatch queue的代码执行完毕。

那么，怎样做到让异步方法也顺序执行呢？其实苹果在设计`NSOperation`时也考虑到了这一点，我们需要创建一个异步的`NSOperation`子类。

```
typealias MLAsyncOperationBlock = (operation:MLAsyncOperation)->Void

class MLAsyncOperation: NSOperation {
    private var ml_executing = false
    private var ml_finished = false
    
    private var block:MLAsyncOperationBlock?
    
    override var asynchronous:Bool {
        return true
    }
    
    override var finished:Bool{
        return ml_finished
    }
    
    override var executing:Bool{
        return ml_executing
    }
    
    func ml_setFinished(finished:Bool){
        willChangeValueForKey("isFinished")
        ml_finished = finished
        didChangeValueForKey("isFinished")
    }
    
    func ml_setExecuting(executing:Bool){
        willChangeValueForKey("isExecuting")
        ml_executing = executing
        didChangeValueForKey("isExecuting")
    }
    
    convenience init(operationBlock:MLAsyncOperationBlock) {
        self.init()
        block = operationBlock
    }
    
    override func start() {
        if cancelled {
            ml_setFinished(true)
            return
        }
        ml_setExecuting(true)
        block?(operation: self)
    }
    
    func finishOperation(){
        ml_setExecuting(false)
        ml_setFinished(true)
    }
    
    deinit{
        print("operation deinited")
    }
}
```
