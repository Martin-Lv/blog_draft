在定制的UITableViewCell中，如果需要对cell中的控件添加事件响应，就要想办法把cell的indexPath传递给响应函数。下面是一个相对方便且耦合度低的方法。UICollectionView同样适用。

假设定制的cell类名为MyUITableViewCell，上面添加了一个按钮myButton，需要在点击MyButton的响应函数中获取cell的indexPath。

解决思路是这样：

1. 在MyUITableViewCell中添加一个指向UITableView的指针，需要获取indexPath时，通过指针向tableView查询。

2. 在MyUITableViewCell中添加一个block属性，在实例化MyUITableViewCell时，给block赋值。在MyUITableViewCell的实现中响应MyButton的点击事件，在响应函数中调用block。

下面来实现一下：

1 在MyUITableViewCell的@inderface中添加两个属性，一个是UITableView指针，一个是响应事件block：

```

typedef void(^OnMyButtonClick)(NSIndexPath *indexPath);

@interface MyUITableViewCell : UITableViewCell

@property (weak, nonatomic) IBOutlet UIButton *myButton;

@property (weak, nonatomic)UITableView *tableView;
@property (nonatomic, strong)OnMyButtonClick onMyButtonClick;
@end
```

2 在MyUITableViewCell的实现中实现myButton的点击响应


```

#import "MyUITableViewCell.h"

@implementation MyUITableViewCell


- (id)initWithFrame:(CGRect)frame
{
    self = [super initWithFrame:frame];
    if (self) {
        // Initialization code
    }
    return self;
}

- (IBAction)onClickMyButton:(id)sender {
	NSIndexPath *indexPath = [[self tableView] indexPathForCell:self];
    if (_onMyButtonClick) {
        _onMyButtonClick(indexPath);
    }
}

@end


```

3 在实例化MyUITableViewCell之后，给两个属性赋值：

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{

	MyUITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:CellIdentifier forIndexPath:indexPath];
	if(cell){
		__weak typeof(self) weakSelf = self;
		cell.tableView = tableView;
		cell.onMyButtonClick = ^(NSIndexPath *indexPath){
			[weakSelf onMyButtonClick:indexPath];
		}
	}
	return cell;
}
```
其中`[weakSelf onMyButtonClick];`调用的是真正的处理点击的方法，实现在对应的viewController中。



有一种做法是将indexPath.row赋值给myButton.tag，点击时直接从tag中取出row的值。这样比较简单，适用于tableViewCell顺序不可变，且只有一个section的情况。

还有一种做法是在MyUITableViewCell中添加一个controller的属性，在MyUITableViewCell实现的响应函数中通过controller调用真正的事件处理方法。这样做的缺点是MyUITableViewCell需要依赖对应controller的实现，耦合度稍微高了一点。