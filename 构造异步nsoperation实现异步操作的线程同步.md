#构造异步NSOperation，实现异步任务的线程同步

题目有点绕，所谓异步任务的线程同步，就是控制多个异步任务的执行顺序。

一个常见的场景：一个任务依赖多个异步任务，必须在几个任务都收到了完成回调以后再执行。

如果换成同步任务，解决方案何其简单。可以用`dispatch_group`，更方便的是用`NSOperation`的`addDependency`功能，让最后执行的任务依赖前面几个任务即可。

我们知道，`NSOperation`内部维护了一个状态机来表示内部任务的执行状态。一共有下面几个状态：

- ready
- executing
- finished
- cancelled

如果我们用`addDependency`给两个`NSOperation`设置了依赖关系，那么一个`NSOperation`对应的方法或block执行完毕后，会变为finished状态，这时另一个`NSOperation`才会执行。

如果我们能让异步任务表现得像同步任务一样，在异步任务收到回调后才变为`finished`状态，不就可以用`addDependency`来控制异步任务的执行顺序了吗？

事实上苹果在`NSOperation`的接口里给我们留了一个口子。看`NSOperation`的接口：

```
NS_CLASS_AVAILABLE(10_5, 2_0)
@interface NSOperation : NSObject {
@private
    id _private;
    int32_t _private1;
#if __LP64__
    int32_t _private1b;
#endif
}

- (void)start;
- (void)main;

@property (readonly, getter=isCancelled) BOOL cancelled;
- (void)cancel;

@property (readonly, getter=isExecuting) BOOL executing;
@property (readonly, getter=isFinished) BOOL finished;
@property (readonly, getter=isConcurrent) BOOL concurrent; // To be deprecated; use and override 'asynchronous' below
@property (readonly, getter=isAsynchronous) BOOL asynchronous NS_AVAILABLE(10_8, 7_0);
@property (readonly, getter=isReady) BOOL ready;

- (void)addDependency:(NSOperation *)op;
- (void)removeDependency:(NSOperation *)op;

@property (readonly, copy) NSArray<NSOperation *> *dependencies;

typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
	NSOperationQueuePriorityVeryLow = -8L,
	NSOperationQueuePriorityLow = -4L,
	NSOperationQueuePriorityNormal = 0,
	NSOperationQueuePriorityHigh = 4,
	NSOperationQueuePriorityVeryHigh = 8
};

@property NSOperationQueuePriority queuePriority;

@property (nullable, copy) void (^completionBlock)(void) NS_AVAILABLE(10_6, 4_0);

- (void)waitUntilFinished NS_AVAILABLE(10_6, 4_0);

@property double threadPriority NS_DEPRECATED(10_6, 10_10, 4_0, 8_0);

@property NSQualityOfService qualityOfService NS_AVAILABLE(10_10, 8_0);

@property (nullable, copy) NSString *name NS_AVAILABLE(10_10, 8_0);

@end
```

我们需要重点关注的是`asynchronous`属性。如果它值为`true`，那么这个`NSOperation`执行完毕后不会自动变为finished状态，需要手动设置。这正是我们想要的。

我们可以写一个`NSOperation`的子类，给异步任务提供一个设为finished状态的接口。

上代码：

```
typealias MLAsyncOperationBlock = (operation:MLAsyncOperation)->Void

class MLAsyncOperation: NSOperation {
    private var ml_executing = false{
        willSet {
            willChangeValueForKey("isExecuting")
        }
        didSet {
            didChangeValueForKey("isExecuting")
        }
    }
    private var ml_finished = false{
        willSet {
            willChangeValueForKey("isFinished")
        }
        didSet {
            didChangeValueForKey("isFinished")
        }
    }
    
    private var block:MLAsyncOperationBlock?
    
    override var asynchronous:Bool {
        return true
    }
    
    override var concurrent:Bool {
        return true
    }
    
    override var finished:Bool{
        return ml_finished
    }
    
    override var executing:Bool{
        return ml_executing
    }
    
    convenience init(operationBlock:MLAsyncOperationBlock) {
        self.init()
        block = operationBlock
    }
    
    override func start() {
        if cancelled {
            ml_finished = true
            return
        }
        ml_executing = true
        block?(operation: self)
    }
    
    func finishOperation(){
        ml_executing = false
        ml_finished = true
    }
    
    deinit{
        print("operation deinited")
    }
}
```
几个需要说明的点：

####一：

`NSOperation`内部有一组成员变量来维护它的executing、finished这些状态，我们访问不到。但我们可以另外加一组成员变量，自己来维护这些状态。一个子类不一定要访问父类的成员变量，只要接口表现得和父类一样就行了。
 
####二：
 
`NSOperationQueue`是通过KVO观察内部的`NSOperation`状态的变化，来自动管理`NSOperation`的执行的。我们在设置自己的`ml_executing`属性的时候，需要表现得像`executing`属性被设置了一样。也就是需要调用一下`willChangeValueForKey("isExecuting")`和`didChangeValueForKey("isExecuting")`两个方法。利用Swift属性的willSet和didSet特性，可以非常方便地实现。
 
finished属性同理。
 
####三：

`finishOperation`这个方法，是用户在收到异步回调，任务完成后需要调用的。调用后这个operation就会变为finished状态。为了方便，我给`MLAsyncOperationBlock`加了一个`MLAsyncOperation`类型的参数，在调用内部block的时候会把`self`传进去。这样用户在构造任务block的时候，通过这个参数就直接可以访问到operation本身了。
 
来简单测试一下好不好用。

先写一个类实现一个同步方法和两个异步方法：

```
class TestClass {
    
    let queue = dispatch_queue_create("TestClass_background_queue", DISPATCH_QUEUE_CONCURRENT)

    func method1(){
        print("method 1 begin")
        for _ in 0 ... 100000 {
            continue
        }
        print("method 1 end")
    }
    
    func asyncMethod1(done:()->Void){
        print("async method 1 begin")
        dispatch_async(queue) { () -> Void in
            for _ in 0 ... 100000 {
                continue
            }
            print("async method 1 end")
            done()
        }
        return
    }
    
    func asyncMethod2(done:()->Void){
        print("async method 2 begin")
        dispatch_async(queue) { () -> Void in
            for _ in 0 ... 100000 {
                continue
            }
            print("async method 2 end")
            done()
        }
        return
    }
}

```
测试代码：

```
let operationQueue = NSOperationQueue()
operationQueue.maxConcurrentOperationCount = 5

let object = TestClass()

let op1 = NSBlockOperation { 
    object.method1()
}

let asyncOp1 = MLAsyncOperation { (operation) in
    object.asyncMethod1{ () -> Void in
        operation.finishOperation()
    }
}

let asyncOp2 = MLAsyncOperation { (operation) in
    object.asyncMethod2{
        operation.finishOperation()
    }
}

op1.addDependency(asyncOp1)
op1.addDependency(asyncOp2)

operationQueue.addOperation(asyncOp1)
operationQueue.addOperation(asyncOp2)
operationQueue.addOperation(op1)

let runloop = NSRunLoop.currentRunLoop()
while runloop.runMode(NSDefaultRunLoopMode, beforeDate: NSDate.distantFuture()){
    continue
}

```

由于测试工程是一个command line tool，我用runloop阻塞住了主线程，避免主线程执行完之后整个程序退出，`operationQueue`中的代码来不及执行。

跑一下，输出结果如下：

```
async method 1 begin
async method 2 begin
async method 1 end
async method 2 end
method 1 begin
method 1 end
```

可以看到，两个异步方法并行执行，在两个方法都收到回调完成之后，最后一个方法开始执行。完美实现了我们的需求。

完整的代码可以从[我的Github](https://github.com/Martin-Lv/asyncOperation)下载。



