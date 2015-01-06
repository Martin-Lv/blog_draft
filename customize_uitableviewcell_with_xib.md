#用xib文件自定义UITableViewCell

最方便的使用xib文件自定义UITableViewCell的方法是这样：

1. 在view controller中，将nib注册到对应的UITableView:

```
[self.tableView registerNib:[UINib nibWithNibName:@"TableCell" bundle:[NSBundle mainBundle]]];

```

2. 在`cellForRowAtIndexPath`中，只需要

```
TableCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TableCell"];
// setup cell
return cell;
```

这样就不需要

```
if (cell == nil){
	//init new cell
}
```

这样麻烦的操作了。