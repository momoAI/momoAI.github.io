在很多APP上都看到过带有3D效果的广告轮播，当时觉得很酷炫。后来我就尝试去写代码实现类似的效果，实现起来其实也比较简单，首先看下运行后的效果:
![效果](https://upload-images.jianshu.io/upload_images/2427856-6fc1591b26591818.gif?imageMogr2/auto-orient/strip)
## 下面主要分享下主要的实现过程
### 首先创建自定义的UICollectionView

```
#import <UIKit/UIKit.h>

@interface AbbrCollectionView : UICollectionView

@property (nonatomic, strong) NSArray *images;

@end
```

images存储要轮播显示的图片（demo里是本地的图片，正常情况应该都是网络请求的图片）

```
#import "AbbrCollectionView.h"
#import "AbbrLayoutView.h"

@interface AbbrCollectionView ()<UICollectionViewDelegate,UICollectionViewDataSource>

@property (nonatomic, strong) AbbrLayoutView *layout;

@end

@implementation AbbrCollectionView

- (instancetype)initWithFrame:(CGRect)frame {
    _layout = [[AbbrLayoutView alloc] init];
    _layout.scrollDirection = UICollectionViewScrollDirectionHorizontal;
    _layout.sectionInset = UIEdgeInsetsMake(0, 5, 0, 5);
    _layout.itemSize = CGSizeMake(frame.size.width / 2, frame.size.height - 10);
    if (self = [super initWithFrame:frame collectionViewLayout:_layout]) {
        self.backgroundColor = [UIColor orangeColor];
        self.showsHorizontalScrollIndicator = NO;
        self.contentInset = UIEdgeInsetsMake(0, (int)(frame.size.width / 4) + 5, 0, (int)(frame.size.width / 4) + 5);
        self.delegate = self;
        self.dataSource = self;
        [self registerClass:[UICollectionViewCell class] forCellWithReuseIdentifier:@"collection"];
    }
    return self;
}

- (void)setImages:(NSArray *)images {
    _images = images;
    [self scrollToItemAtIndexPath:[NSIndexPath indexPathForRow:_images.count / 2 inSection:0] atScrollPosition:UICollectionViewScrollPositionCenteredHorizontally animated:NO];
}
```

```
#import "ViewController.h"
#import "AbbrCollectionView.h"

@interface ViewController ()

@property (nonatomic, strong) NSMutableArray *datas;
@property (nonatomic, strong) AbbrCollectionView *collectionView;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    CGFloat screenW = [UIScreen mainScreen].bounds.size.width;
    CGFloat screenH = [UIScreen mainScreen].bounds.size.height;
    _datas = [NSMutableArray array];
    for (int i = 1; i < 10; i ++) {
        [_datas addObject:[NSString stringWithFormat:@"timg%d.jpg",i]];
    }
    _collectionView = [[AbbrCollectionView alloc] initWithFrame:CGRectMake(0, screenH / 4, screenW, screenH / 2)];
    _collectionView.images = _datas;
    [self.view addSubview:_collectionView];
}

@end
```

因为是3D的旋转效果，原生的UICollectionViewFlowLayout布局并不能满足要求，这里需要自定义一个AbbrLayoutView的布局。初始化的时候根据需要设置Layout的itemSize及sectionInset等属性。在设置图片数据源images时，让CollectionView滚动至中间位置。
然后实现UICollectionView代理及数据源

```
- (void)setImages:(NSArray *)images {
    _images = images;
    [self scrollToItemAtIndexPath:[NSIndexPath indexPathForRow:_images.count / 2 inSection:0] atScrollPosition:UICollectionViewScrollPositionCenteredHorizontally animated:NO];
}

- (NSInteger)collectionView:(UICollectionView *)collectionView numberOfItemsInSection:(NSInteger)section {
    return _images.count;
}

- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath {
    UICollectionViewCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:@"collection" forIndexPath:indexPath];
    UIImage *image = [UIImage imageNamed:_images[indexPath.row]];
    cell.backgroundView = [[UIImageView alloc] initWithImage:image];
    cell.layer.cornerRadius = 10;
    cell.layer.masksToBounds = YES;
    return cell;
}
```

这里cell只是显示图片，并没有其他控件，所以直接使用UICollectionViewCell没有自定义。
屏幕显示的不止当前一张图片，不能直接使用系统的pagingEnabled属性设置分页效果，所以接下来还需实现UIScrollView的代理自己控制分页的效果：

```
- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate {
    CGFloat scrDynameter = scrollView.contentOffset.x / (_layout.itemSize.width + 10);
    if(!decelerate){
        NSInteger scrIndex = (NSInteger)(scrDynameter + 1);
        [self scrollToItemAtIndexPath:[NSIndexPath indexPathForRow:scrIndex inSection:0] atScrollPosition:UICollectionViewScrollPositionCenteredHorizontally animated:YES];
    }
}

- (void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView {
    CGFloat scrDynameter = scrollView.contentOffset.x / (_layout.itemSize.width + 10);
    NSInteger scrIndex = (NSInteger)(scrDynameter + 1);
    [self scrollToItemAtIndexPath:[NSIndexPath indexPathForRow:scrIndex inSection:0] atScrollPosition:UICollectionViewScrollPositionCenteredHorizontally animated:YES];
}
```

### 自定义UICollectionViewFlowLayout

```
#import "AbbrLayoutView.h"

@implementation AbbrLayoutView

///  返回collectionView上面当前显示的所有元素（比如cell）的布局属性:这个方法决定了cell怎么排布
///  每个cell都有自己对应的布局属性：UICollectionViewLayoutAttributes
///  要求返回的数组中装着UICollectionViewLayoutAttributes对象
- (NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSArray *attributes = [super layoutAttributesForElementsInRect:rect];
    CGRect visitRect = {self.collectionView.contentOffset,self.collectionView.bounds.size};
// 需要copy原来的attributes 否则会有警告：This is likely occurring because the flow layout subclass xxxx is modifying attributes returned by UICollectionViewFlowLayout without copying them
    NSMutableArray *attributesCopy = [NSMutableArray array];
    for (UICollectionViewLayoutAttributes *attribute in attributes) {
        UICollectionViewLayoutAttributes *attributeCopy = [attribute copy];
        [attributesCopy addObject:attributeCopy];
    }
    
    for (UICollectionViewLayoutAttributes *attribute in attributesCopy) {
        CGFloat distance = CGRectGetMidX(visitRect) - attribute.center.x;
        CGFloat coefficient = distance / (self.itemSize.width * 6); // 根据每个cell的距离设置旋转系数
        attribute.transform3D = CATransform3DMakeRotation(coefficient * M_PI , 1, 0, 0); // 旋转
    }
    return attributesCopy;
}

// 当边界发生改变时，是否应该刷新布局。如果YES则在边界变化（一般是scroll到其他地方）时，将重新计算需要的布局信息。
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds {
    return YES;
}

@end
```

自定义UICollectionViewFlowLayout需要重写- (NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect方法，旋转效果都写在这个方法里，通过NSArray<UICollectionViewLayoutAttributes *>数组返回。
####到这里似乎已大功告成了，很开心的运行代码，但发现结果并不是我们想要的：
![22.jpg](https://upload-images.jianshu.io/upload_images/2427856-75fb83a7c628691c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
明明设置的是CATransform3DMakeRotation旋转效果，为什么运行的结果是缩放的效果呢？
这是因为，在CALayer的显示系统中，默认的相机使用正交投影，正交投影没有远小近大效果，所以在本例中，只能造成相应轴上的缩放。所以需要通过矩阵连乘自己构造透视投影矩阵，代码如下:

```
/* center:相机的位置，相机的位置是相对于要进行变换的CALayer的来说的，
原点是CALayer的anchorPoint在整个CALayer的位置，
例如CALayer的大小是(100, 200), anchorPoint值为(0.5, 0.5)，
此时anchorPoint在整个CALayer中的位置就是(50, 100)，正中心的位置。
传入透视变换的相机位置为(0, 0)，那么相机所在的位置相对于CALayer就是(50, 100)。
如果希望相机在左上角，则需要传入(-50, -100)。disZ:相机离z=0平面（也可以理解为屏幕）的距离*/
CATransform3D CATransform3DMakePerspective(CGPoint center, float disZ) {
    CATransform3D transToCenter = CATransform3DMakeTranslation(-center.x, -center.y, 0);
    CATransform3D transBack = CATransform3DMakeTranslation(center.x, center.y, 0);
    CATransform3D scale = CATransform3DIdentity;
    scale.m34 = -1.0f / disZ;
    return CATransform3DConcat(CATransform3DConcat(transToCenter, scale), transBack);
}

CATransform3D CATransform3DPerspect(CATransform3D t, CGPoint center, float disZ) {
    return CATransform3DConcat(t, CATransform3DMakePerspective(center, disZ));
}
```

有关这部分内容可以参考这篇[博客](http://www.cocoachina.com/bbs/read.php?tid-117061-page-1.html)
最后把刚才attribute.transform3D = CATransform3DMakeRotation(coefficient * M_PI , 1, 0, 0);的代码换成透视投影就能实现3D的效果：

```
- (NSArray<UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect {
    NSArray *attributes = [super layoutAttributesForElementsInRect:rect];
    CGRect visitRect = {self.collectionView.contentOffset,self.collectionView.bounds.size};
    NSMutableArray *attributesCopy = [NSMutableArray array];
    for (UICollectionViewLayoutAttributes *attribute in attributes) {
        UICollectionViewLayoutAttributes *attributeCopy = [attribute copy];
        [attributesCopy addObject:attributeCopy];
    }
    
    for (UICollectionViewLayoutAttributes *attribute in attributesCopy) {
        CGFloat distance = CGRectGetMidX(visitRect) - attribute.center.x;
        CGFloat coefficient = distance / (self.itemSize.width * 6);
        //        attribute.transform3D = CATransform3DMakeRotation(coefficient * M_PI , 1, 0, 0);
        CATransform3D rotate = CATransform3DMakeRotation(coefficient * M_PI, 0, 1, 0);
        attribute.transform3D = CATransform3DPerspect(rotate, CGPointMake(0, 0), 200);
    }
    return attributesCopy;
}
```

[demo](https://github.com/momoAI/HHCollectionView)
