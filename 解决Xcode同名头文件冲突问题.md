#解决 Xcode 同名头文件冲突问题

一般情况下，在 Objective-C 代码中 import 一个工程中的头文件，是无需指定路径的，只需 `#import "header.h"` 即可。Xcode会自动寻找所有包含在工程中的头文件。

但如果有两个头文件在不同的路径下，文件名相同，Xcode就无法正确地处理。

最近开发过程中就遇到了这种情况。我们组要开发一个 framework 给 A B C 几个 app 使用，这个 framework 又依赖于另一个组提供的静态库 lib.h lib.a。问题来了，app A 和 app B 需要的 lib 接口有差别，于是就有了 A 版 lib.h 和 B 版 lib.h，都需要引入到framework中。于是目录结构成了这样：

```
lib
	A
		lib.h
		lib.a
	B
		lib.h
		lib.a
```


##禁用 USE_HEADERMAP

##指定user header search path

##文件夹名字带空格解决办法

在空格前加一个反斜杠“\”即可。
 
