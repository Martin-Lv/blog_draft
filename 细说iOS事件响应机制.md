#细说iOS触摸事件响应机制（一）


iOS设备支持的用户操作事件有三种：

- Multitouch Event（多点触摸事件）
- Motion Event（设备运动事件）
- Remote Control Event（远程控制事件）

其中最常用的就是多点触摸事件。一个触摸事件的响应过程如下：

1. 用户触摸屏幕时，UIKit会生成UIEvent对象来描述触摸事件。对象内部包含了触摸点坐标等信息。
2. 通过Hit Test确定用户触摸的是哪一个UIView。这个步骤通过`- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event`方法来完成。
3. 找到被触摸的UIView之后，如果它能够响应用户事件，相应的响应函数就会被调用。如果不能响应，就会沿着响应链（Responder Chain）寻找能够响应的UIResponder对象（UIView是UIResponder的子类）来响应触摸事件。

##Hit Test

我们知道，iOS应用界面是通过许多UIView以树状结构组织而成的。视图树的根视图就是UIViewController的view属性，它的frame是最大的。每个视图的子视图（subview），一般都是在父视图（superview）的内部。视图的层级关系一级一级延伸，越往下视图面积越小。

Hit Test的过程是这样的：

从视图树的根部开始，按照下面的规则遍历所有的子视图。

1. 检查一个视图A的所有子视图，如果触摸点不在任何一个子视图内部，那么A就是用户触摸的视图。Hit Test结束。
2. 如果某一个触摸点在某一个子视图B内部，按照1的方法检查B的所有子视图。依此类推。

很明显，Hit Test就是对视图树的一个广度优先搜索过程。

我们尝试自己实现一下`- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event`方法：

```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    //为了方便展示Hit Test的过程，添加一些log
    NSMutableString *log = [NSMutableString string];
    UIView *superview = self.superview;
    while (superview) {
        [log appendString:@"--"];
        superview = superview.superview;
    }
    [log appendString:NSStringFromClass(self.class)];
    printf("%s\n",log.UTF8String);
    //排除几种视图不需要响应触摸事件的情况
    if (self.hidden || !self.userInteractionEnabled || self.alpha <= 0.01 ) {
        return nil;
    }
    if (![self pointInside:point withEvent:event]) {
        return nil;
    }
    //同一层级的视图，后添加的在上层，有更高的响应优先级。所以要把subviews数组反过来遍历
    for (UIView *view in self.subviews.reverseObjectEnumerator) {
        CGPoint localPoint = [self convertPoint:point toView:view];
        UIView *hitView = [view hitTest:localPoint withEvent:event];
        if (hitView) {
            return hitView;
        }
    }
    return self;
}
```

为了验证我们的实现，需要利用runtime把UIView的默认实现替换掉。我们写一个UIView子类，就叫MLHitTestHackView，然后把刚才的实现放进去：

```
@implementation MLHitTestHackView

+ (void)load
{
    [self hackHitTest];
}

+ (void)hackHitTest
{
    SEL hitTestSEL = @selector(hitTest:withEvent:);
    Method customHitTest = class_getInstanceMethod([MLHitTestHackView class], hitTestSEL);
    Method originalHitTest = class_getInstanceMethod([UIView class], hitTestSEL);
    method_exchangeImplementations(customHitTest, originalHitTest);
}

- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
  //这里放上文的实现
}

@end

```

随手画几个view测试一下，如图所示：

![view.jpg](/Users/lvmenglin/Documents/views.jpg)

Button的Action方法中打个log：

```
@IBAction func onClickButton(sender: AnyObject) {
    print("button clicked")
}
```

点击Button输出结果如下：

```
UIWindow
--UIView
----_UILayoutGuide
----_UILayoutGuide
----UIView
------UIView
------UIButton
--------UIButtonLabel
UIStatusBarWindow
--UIStatusBar
UIWindow
--UIView
----_UILayoutGuide
----_UILayoutGuide
----UIView
------UIView
------UIButton
--------UIButtonLabel
button clicked
```

可以看到整个Hit Test流程和上文描述的一致，遵循广度优先的规则。

通过这个log输出我们还可以看到一些UIKit没有暴露的细节，比如UIButton内部有个UIButtonLabel视图。

通过对Hit Test过程的分析，我们也明白了为什么一个子视图超出了父视图边界，超出的部分不能响应触摸：一个视图在Hit Test中被找到的前提是它的父视图被找到。如果用户触摸的位置超出了父视图的边界，UIKit无法根据这个触摸点找到父视图，自然也就无法找到子视图。

另外貌似整个Hit Test过程走了两遍，暂时还没有搞清楚原因，欢迎高手指教。

##Hit Test原理的应用

知道了Hit Test的原理，我们可以用它来做一些有趣的事情。

app做了新功能，我们当然希望用户马上去用，这就需要做一些操作引导。一种常见的形式是用户第一次打开新版本界面时，用一个半透明的蒙板蒙在界面上，画一些指引和说明文字。但这种蒙板往往需要用户点一下让它消失，然后才能继续操作，体验不是很好。

我们可以利用Hit Test的机制，让用户点击指定位置时，被蒙板盖住的控件能够响应用户的操作，同时蒙板消失。这样用户按照指示操作马上就能生效，整个流程就顺畅多了。

回顾一下文章开头提到的触摸事件响应过程，一个视图要响应触摸事件，先要通过Hit Test被“找到”。所以只要在`- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event`方法上做做文章，就可以让点击事件穿透蒙板到达下面的控件了。

关键逻辑也不复杂：如果用户触摸了指定的区域，则调用`removeFromSuperview`方法将蒙板移除，然后返回蒙板的superview。因为蒙板一般都是直接加在UIViewController的根视图上，所以这里返回的就是根视图。然后触摸位置对应的视图就可以通过正常的Hit Test流程找到。

这个部分不需要runtime，使用Swift实现。代码如下：

```

import Foundation
import UIKit

extension UIColor {
    class func color(hex hex:UInt, alpha:CGFloat = 1.0) -> UIColor{
        let redValue = CGFloat(hex & 0xFF0000 >> 16) / 255.0
        let greenValue = CGFloat(hex & 0xFF00 >> 8) / 255.0
        let blueValue = CGFloat(hex & 0xFF) / 255.0
        
        return UIColor(red: redValue, green: greenValue, blue: blueValue, alpha: alpha)
    }
}

class MLGuideMaskView: UIView {
    var BGColor = UIColor.color(hex: 0x000000, alpha: 0.6)
    
    var transparentAreaPath:CGPath?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        insideInit()
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        insideInit()
    }
    
    private func insideInit(){
        backgroundColor = UIColor.clearColor()
    }
    //这里用BGColor填充transparentAreaPath之外的区域，让触摸事件可以穿透的区域全透明高亮显示
    override func drawRect(rect: CGRect) {
        let context = UIGraphicsGetCurrentContext()
        CGContextSetFillColorWithColor(context, BGColor.CGColor)
        CGContextAddRect(context, rect)
        if let path = transparentAreaPath{
            CGContextAddPath(context, path)
            CGContextEOFillPath(context)
        }else{
            CGContextFillPath(context)
        }
    }
    
    override func hitTest(point: CGPoint, withEvent event: UIEvent?) -> UIView? {
        //被点击时，XYGuideMaskView消失，同时将点击事件传给下面的view。
        //必须先removeFromSuperview()再调用superview的hitTest。UIView的hitTest方法会逐个调用subview的hitTest，直接调用superview的hitTest会造成循环调用。
        print("hit test in guide mask view")
        var hitInReactiveArea = false
        if let path = transparentAreaPath {
            hitInReactiveArea = CGPathContainsPoint(path, nil, point, false)
        }
        if let superview = superview where hitInReactiveArea{
            removeFromSuperview()
            return superview.hitTest(point, withEvent: event)
        }else{
            //点击其他区域的处理留在touchesBegan中完成
            return self
        }
    }
    
    override func touchesBegan(touches: Set<UITouch>, withEvent event: UIEvent?) {
        removeFromSuperview()
    }
}

```

可以把这个GuideMaskView加到某个界面上试一试，下面的控件响应触摸的同时蒙板消失，效果很有趣。记得设置transparentAreaPath属性。

下一篇文章会分析Responder Chain的原理，同样给出一个有意思的应用。敬请期待。

