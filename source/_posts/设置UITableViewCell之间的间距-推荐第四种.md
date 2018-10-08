---
title: 设置UITableViewCell之间的间距(推荐第四种)
date: 2018-05-24 11:17:26
tags:
---

1.设置假的间距，我们在``tableviewcell``的``contentView``上添加一个view，比如让其距离上下左右的距离都是``10``；这个方法是最容易想到的；

2.用``UIContentView``来代替``tableview``，然后通过下面这个函数来设置``UICollectionViewCell``的上下左右的间距；

```
//协议中的方法，用于返回单元格的大小  
- (CGSize)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout sizeForItemAtIndexPath:(NSIndexPath *)indexPath{  
    return CGSizeMake(ScreenWidth-20,150);  
}  
//协议中的方法，用于返回整个CollectionView上、左、下、右距四边的间距  
- (UIEdgeInsets)collectionView:(UICollectionView *)collectionView layout:(UICollectionViewLayout *)collectionViewLayout insetForSectionAtIndex:(NSInteger)section{  
    //上、左、下、右的边距  
    return UIEdgeInsetsMake(10, 10, 10, 10);  
}
```

3.用控件``tableview``，比如有十条数据，那就给``tableview``分十组，每组只放一条数据，也就是一个``cell``，然后设置``UICollectionViewCell``的``head view``和``foot view``来设置``cell``的间距，但是这个方法只能设置上下间距，如果想设置距离屏幕左右的距离，可以设置``uitableview``距离左右的距离；``uitableview``的``style``为``UITableViewStyleGrouped``；不然``headview``会浮动；

```
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView{  
    return 10;  
}  
  
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section{  
    return 1;  
}  
  
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath{  
    return 50;  
}  
  
- (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section{  
    return 10;  
}  
  
- (CGFloat)tableView:(UITableView *)tableView heightForFooterInSection:(NSInteger)section{  
    return 0.00001;  
}  
  
- (UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section{  
    UIView *headView = [[UIView alloc]init];  
    headView.backgroundColor = [UIColor redColor];  
    return headView;  
}  
  
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath{  
    static NSString *TableSampleIdentifier = @"cellStr";  
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:TableSampleIdentifier];  
    if (cell == nil) {  
        cell = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleValue1 reuseIdentifier:TableSampleIdentifier];  
    }  
    return cell;  
}

```

4.重新设置的``UITableViewCellframe``。

代码如下：

```
#import "MyViewCell.h"  
  
@implementation MyViewCell  
  
- (void)awakeFromNib {  
    [super awakeFromNib];  
    // Initialization code  
}  
  
- (void)setFrame:(CGRect)frame{  
    frame.origin.x += 10;  
    frame.origin.y += 10;  
    frame.size.height -= 10;  
    frame.size.width -= 20;  
    [super setFrame:frame];  
}  
  
- (void)setSelected:(BOOL)selected animated:(BOOL)animated {  
    [super setSelected:selected animated:animated];  
  
    // Configure the view for the selected state  
}  
  
@end

```

参考链接： [这里](https://blog.csdn.net/u014220518/article/details/51995989)