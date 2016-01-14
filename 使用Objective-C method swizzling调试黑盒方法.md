#使用Objective-C method swizzling调试黑盒方法

Objective-C本质上是一门非常动态的语言。向一个对象发送消息后，具体会调用哪个方法是在运行期决定的。method swizziling，是通过一些Objective-C运行时接口，来修改一个方法对应的实现。这样，我们可以对一些实现不可见的方法增加日志记录功能，从而方便程序调试。

下面我们通过一些示例来介绍几种method swizzling的操作方法，并比较他们的优缺点。下面的一些类型定义和函数声明包含在`<objc/runtime.h>`中。

##1.交换方法实现

通过`void method_exchangeImplementation(Method m1, Method m2)`函数，我们可以交换两个方法的实现。

获取方法的实现则需要通过`Method class_getInstanceMethod(Class aCalss, SEL aSelector)`函数。

