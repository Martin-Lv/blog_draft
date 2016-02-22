#UIAlertView的坑

有一种常见的需求：弹出一个`UIAlertView`让用户编辑一段文字，当字符串为空时，禁止用户点击确定，防止用户输入空字符串。

iOS 7上不得不用UIAlertView。`UIAlertViewDelegate`中有个方法：

```
optional public func alertViewShouldEnableFirstOtherButton(alertView: UIAlertView) -> Bool
```

文档中没有详细说，但是头文件注释中是这样说的：

> Called after edits in any of the default fields added by the style

也就是说我们只要实现了这个回调，用户在textField中修改字符串时这个回调就会被调用。

只有调用

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id /*<UIAlertViewDelegate>*/)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle otherButtonTitles:(nullable NSString *)otherButtonTitles, ... NS_REQUIRES_NIL_TERMINATION NS_EXTENSION_UNAVAILABLE_IOS("Use UIAlertController instead.");
```

创建，这个回调才会被调用。

swift中

```
    public convenience init(title: String, message: String, delegate: UIAlertViewDelegate?, cancelButtonTitle: String?, otherButtonTitles firstButtonTitle: String, _ moreButtonTitles: String...)
```

实际上是调用了

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle;
```

这个接口创建了`UIAlertView` 再添加button来实现的，因此正好踩上这个坑。

唯一办法是用OC包装一下

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id /*<UIAlertViewDelegate>*/)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle otherButtonTitles:(nullable NSString *)otherButtonTitles, ... NS_REQUIRES_NIL_TERMINATION NS_EXTENSION_UNAVAILABLE_IOS("Use UIAlertController instead.");
```

然后在swift中调用这个包装，绕开这个bug。
