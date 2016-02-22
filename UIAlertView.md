#iOS文字编辑对话框的两个不大不小的坑

有一种常见的需求：弹出一个对话框让用户编辑一段文字，当textField的字符串为空时，禁止用户点击确定，防止用户输入空字符串。同时当用户输入文字达到长度上限时禁止用户继续输入。

iOS 7上不得不用UIAlertView。`UIAlertViewDelegate`中有个方法：

```
optional public func alertViewShouldEnableFirstOtherButton(alertView: UIAlertView) -> Bool
```

文档中没有详细说，但是头文件注释中是这样说的：

> Called after edits in any of the default fields added by the style

也就是说只要实现了这个方法，用户在textField中修改字符串时`UIAlertView`就会调用它。

不过`UIAlertView`相关的实现有个bug，或者说特性：

`alertViewShouldEnableFirstOtherButton`里面的FirstOtherButton，指的是通过

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle otherButtonTitles:(nullable NSString *)otherButtonTitles, ... ;
```

这个方法初始化一个`UIAlertView`的时候，在otherButtonTitles:里面传的第一个title对应的按钮。如果初始化的时候没有传这个参数，而是通过`addButtonWithTitle:`添加了一个按钮，是不能通过`func alertViewShouldEnableFirstOtherButton(alertView: UIAlertView) -> Bool`来控制按钮是否enable的。

本来这也不是什么大问题，但是在用Swift写`UIAlertView`的时候，不管是不是在otherButtonTitles:里面传按钮标题，回调都不会被调用。

我们有理由猜测，在Swift中，

```
public convenience init(title: String, message: String, delegate: UIAlertViewDelegate?, cancelButtonTitle: String?, otherButtonTitles firstButtonTitle: String, _ moreButtonTitles: String...)
```

这个init方法，并不是调用对应的Objective-C方法，而是调用

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle;
```

这个方法创建了`UIAlertView`，再调用`addButtonWithTitle:`添加按钮来实现的。因此正好踩上这个坑。

要验证也很简单，我们用Objective-C自己写一个wrapper方法：

```
- (instancetype)initWithTitle:(nullable NSString *)title message:(nullable NSString *)message delegate:(nullable id)delegate cancelButtonTitle:(nullable NSString *)cancelButtonTitle okButtonTitle:(nullable NSString *)okButtonTitle
{
    return [self initWithTitle:title message:message delegate:delegate cancelButtonTitle:cancelButtonTitle otherButtonTitles:okButtonTitle, nil];
}
```

然后在swift中调用这个包装方法，绕开swift默认的init方法。

写wrapper的过程中发现，给可变参数个数的方法写wrapper不好处理。因为参数个数不定，难以表达“将接收的所有参数传递给内层的方法调用”这个语义。大概这也是Swift接口没有直接调用同名的Objective-C接口的原因。

经过验证，用自己的wrapper方法初始化的`UIAlertView`可以正常调用`alertViewShouldEnableFirstOtherButton(alertView: UIAlertView)`方法，验证了我们的猜测，也解决了问题。

UIAlertController就简单了，`UIAlertAction`自带enabled属性，需要的时候设一下即可。

关于避免用户输入字符串超过长度上限，也有一个细节需要注意：

`UITextFieldDelegate`不像`UITextViewDelegate`一样有一个`textViewDidChange(_ textView: UITextView)`方法，没法在text改变的时候即时收到回调。虽然有一个`func textField(_ textField: UITextField, shouldChangeCharactersInRange range: NSRange, replacementString string: String) -> Bool`方法可以实现近似的效果，但用户在输入一个词后，点击系统键盘上方的联想词语，这个方法是不会被调用的。所以用户可以一直点击联想词语，突破字符串长度限制。

解决方法也不复杂：

```
textField.addTarget(self, action: "textFieldDidChange:", forControlEvents: .EditingChanged)
```

然后再实现textFieldDidChange方法即可。当然取别的名字也可以。
