---
title: iOS 插入数据到 TableView 顶部
date: 2018-08-26 12:46:18
tags:
---
困扰了几天的问题终于得到解决了，感谢 github，感谢开源~

在这里记录下遇到的问题，算是记录也避免他人少走一些弯路。

首先记录下插入数据的流程吧：

1. 更新数据源
2. 调用 ``insertRowsAtIndexPaths: withRowAnimation``方法进行插入
3. 设置 tableView 的 ``setContentOffset `` 

然后再细说一下遇到的问题吧。

- 更新数据源之后进行插入崩溃

崩溃的很莫名其妙，后来查找是插入的``indexPaths`` 的个数和数据源的个数不一致导致的(忘记了一个额外的时间 cell )。这里有一点需要注意的是，插入之后的 DataSource 个数必须等于插入前的加上待插入的个数之和。

另外，为了避免在插入的过程中有其他的线程也来进行插入。一般都会用 ``beginUpdates `` 和 ``endUpdates `` 包起来，保证插入完成之后再插其他的。

```
[UIView setAnimationsEnabled:NO];
[self.tableView beginUpdates];
[self.viewModel.listData addObject:model_time];
 NSIndexPath *indexpath = [NSIndexPath indexPathForRow:MAX(0, self.viewModel.listData.count -1) inSection:0];
[self.tableView insertRowAtIndexPath:indexpath withRowAnimation:UITableViewRowAnimationNone];
[self.tableView endUpdates];
[UIView setAnimationsEnabled:YES];
```

上面的代码片段，除了 ``beginUpdates `` 和 ``endUpdates ``之外还有一个``setAnimationsEnabled `` 这个，这个又是刚什么用的呢？这是接下来我要讲的第二个问题。

- 插入数据之后刷新 ``tableView`` 有上个``cell`` 的残影。

尽管你已经把插入动画设置为 ``UITableViewRowAnimationNone `` 了还是会有。Stack Overflow 查找了一下，发现在插入的时候把``UIView``的动画去掉就可以了。

- 插入多条数据``tableView``会滚动到顶部。

对于滚动到顶部这个问题，查找了好多回答，清一色的都是设置 ``setContentOffset `` ，问题是尽管我设置了正确的 ``setContentOffset `` 还是有问题。一直被这个问题困扰了几天，后来干脆去``github``上翻看``IM``相关项目的源码，发现了插入数据的时候有些不同。

原本我是这样插入数据的：

```
[UIView setAnimationsEnabled:NO];
[self.tableView beginUpdates];
[self.viewModel.listData insertObjects:models atIndex:0];
NSMutableArray<NSIndexPath*>* indexPaths = [[NSMutableArray alloc] initWithCapacity:models.count];
for (int i = 0; i < models.count; i++) {
       NSIndexPath *indexPath = [NSIndexPath indexPathForRow:i inSection:0];
       [indexPaths addObject:indexPath];
}
[self.tableView insertRowsAtIndexPaths:indexPaths withRowAnimation:UITableViewRowAnimationTop];
[self.tableView endUpdates];
[UIView setAnimationsEnabled:YES];
```

改成这样就可以了：

```

[UIView setAnimationsEnabled:NO];
[self.tableView beginUpdates];
NSMutableArray<NSIndexPath*>* indexPaths = [[NSMutableArray alloc] initWithCapacity:models.count];
NSMutableIndexSet *indexSets = [[NSMutableIndexSet alloc] init];
[models enumerateObjectsUsingBlock:^(RSChatTaskModel * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
NSIndexPath *indexPath = [NSIndexPath indexPathForRow:idx inSection:0];
[indexPaths addObject:indexPath];
[indexSets addIndex:idx];
    }];
[self.viewModel.listData insertObjects:models atIndexes:indexSets];
[self.tableView insertRowsAtIndexPaths:indexPaths withRowAnimation:UITableViewRowAnimationTop];
[self.tableView endUpdates];
[UIView setAnimationsEnabled:YES];
```

- 计算``setContentOffset ``的方式

有``2``种方式：
1、在``scrollViewDidScroll``的时候，在插入之前拿到插入前的``tableView``的``contentOffset`` ，插入之后(``endUpdates ``的时候会去重新计算``tableView:estimatedHeightForRowAtIndexPath``)再去拿``contentOffset``，两者相加即是插入``cell``的高度

```
插入cells的高度 =  afterContentSize.height - beforeContentSize.height
```

- 另一个就是加载下一页的时机和防止重复加载的问题，这里我设置为还剩下``5``条数据的时候开始加载下一页，并且是往下拉加载更多的时候才会去加载。



插入``cells``到``tableview``顶部完整代码片段为：

```
NSLog(@"loadMore: 加载下一页数据...");
CGSize beforeContentSize = self.tableView.contentSize;
CGPoint beforeContentOffset = self.tableView.contentOffset;
[UIView setAnimationsEnabled:NO];
[self.tableView beginUpdates];
NSMutableArray<NSIndexPath*>* indexPaths = [[NSMutableArray alloc] initWithCapacity:models.count];
NSMutableIndexSet *indexSets = [[NSMutableIndexSet alloc] init];
[models enumerateObjectsUsingBlock:^(RSChatTaskModel * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:idx inSection:0];
    [indexPaths addObject:indexPath];
    [indexSets addIndex:idx];
}];
[self.viewModel.listData insertObjects:models atIndexes:indexSets];
[self.tableView insertRowsAtIndexPaths:indexPaths withRowAnimation:UITableViewRowAnimationTop];
[self.tableView endUpdates];
[UIView setAnimationsEnabled:YES];
    
CGSize afterContentSize = self.tableView.contentSize;
CGPoint afterContentOffset = self.tableView.contentOffset;

NSArray *subArr = [self.viewModel.listData subarrayWithRange:NSMakeRange(0, models.count)];
float insertCellsH = 0.0;
for (RSChatTaskModel *model in subArr) {
    insertCellsH += [model.height floatValue];
}

CGPoint newContentOffset = CGPointMake(afterContentOffset.x, beforeContentOffset.y + (afterContentSize.height - beforeContentSize.height));
[self.tableView setContentOffset:newContentOffset animated:NO];
```
