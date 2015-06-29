#关于XCode链接选项 -ObjC -all\_load -force\_load

在Xcode工程的Other linker flags中添加-ObjC 标志可以解决使用静态库中的category时出现unrecognized selector的问题。

UNIX的静态库（xxx.a）其实就是一个目标文件（xxx.o）的集合。在C语言中，编译一个源文件时如果遇到在其他文件中定义的函数，则会留下一个 undefined symbol。在链接时会通过找到其他文件中的定义来确定这个symbol。

在Objective-C中，由于方法调用都是在运行期确定的，因此没有针对方法的symbol，只有针对类的。

这样在静态库中如果使用了category扩展已有的类，编译器不知道如何将category和已有的类整合在一起，就会导致unrecognized selector问题。添加-ObjC标志后，编译器会把一个类相关的所有目标文件都加载，这样就解决了这个问题。但是由于这样做会使可执行文件体积变大，所以没有设为默认选项。

在64位ios应用环境下，由于链接器的一个bug，在静态库中只有category没有对应的class定义时，-ObjC标志会失效。这时可以使用-all\_load强制加载所有目标文件，或者使用-force\_load指定加载某一个包。

在Xcode4.2之后，这个链接器bug已经被修复，因此-all\_load 和 -force\_load标志都**不再需要**了。在必要时添加-ObjC即可。

参见：

https://developer.apple.com/library/mac/qa/qa1490/_index.html

http://stackoverflow.com/questions/7942616/why-is-force-load-no-longer-required-for-my-three20-dependencies-in-xcode-4-2/7942924#7942924
