对于iOS应用, plist文件可以用来固化应用数据，保存应用设置。比如做一个需要登录的应用，可以把用户输入的用户名和密码保存下来，下次启动应用时可以自动填入，不必重复输入了。

###新建plist文件

在xcode中新建文件，选择iOS-Resources-Property List，输入文件名，选择要放置的group即可。这里我们设置文件名为userData.plist。

###在xcode中修改plist文件

点击建好的plist文件，会打开一个表格，最上面一行是Root，这里是plist的根数据，类型必须是dictionary或array。所有的数据都保存在Root里面。这里我们将Root设置为dictionary。

比如我们需要保存登录地址、用户名、密码三个字符串，就在Root下面新建三个行，类型选择string，key分别是address userName passWord。在value一栏我们可以设置默认值，比如把address设置为192.168.1.101。

###读取和写入plist文件数据

需要注意的是，往application bundle中写文件是做不到的，所以无法往我们刚才新建的plist文件中写入数据。如果想在程序中改写几个键值的内容，需要把plist文件复制到app文档目录中。

获取文档目录的代码如下：

```
NSString *destPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
destPath = [destPath stringByAppendingPathComponent:@"userData.plist"];
```
        
第一次启动应用时这个路径指向的文件还不存在。这时我们把bundle中的文件复制过来。这个动作只会执行这一次。

```
NSFileManager *fileManager = [NSFileManager defaultManager];        
if (![fileManager fileExistsAtPath:destPath]) {
     NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"userData" ofType:@"plist"];
     [fileManager copyItemAtPath:sourcePath toPath:destPath error:nil];
}
```

这时我们就可以通过destPath读写plist中的数据了。先要获取plist的根节点：

```    
NSMutableDictionary userDataDict = [[NSMutableDictionary alloc] initWithContentsOfFile:destPath];
```

读取数据：

```    
NSString *address = [NSString stringWithFormat:@"%@", [userDataDict objectForKey:@"address"]];
```

写入数据：

```
[userDataDict setValue:address forKey:@"address"];
[userDataDict writeToFile:destPath atomically:YES];
```

注意写入的时候要用我们复制过去的地址destPath，不要弄错了。