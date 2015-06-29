#关于使用block的retain cycle风险

在Objective-C中，对于异步操作的接口，操作完毕后通知调用者的回调方式主要有delegate和block两种。

用block的好处是代码量少，可以将调用接口的代码和处理回调的代码写在一起，更易读。比如：

```
[object asyncOperationWithCompletion:^{
	//do completion
}];
```
但是block回调方式不小心处理很容易导致retain cycle问题。

