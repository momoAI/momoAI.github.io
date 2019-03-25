之前好不容易弄明白了UIButton的titleEdgeInsets和imageEdgeInsets属性，然而今天应用到的时候又磕磕绊绊的花了好长时间才理清楚。长痛不如短痛，今天就在这里花点时间做个笔记，同时希望可以方便不了解这两个属性用法的伙伴有所理解。

我们知道UIButton由一个titleLabel和imageView组成的，显而易见titleEdgeInsets和imageEdgeInsets分别对应着UIButton中的titleLabel和imageView。这两个属性是UIEdgeInsets结构体：
```
typedef struct UIEdgeInsets {
    CGFloat top, left, bottom, right;  // specify amount to inset (positive) for each of the edges. values can be negative to 'outset'
} UIEdgeInsets;
```
titleEdgeInsets和imageEdgeInsets作用类似于UIScrollView的contentInset（UIButton也有contentEdgeInsets）,top,left,bottom,right分别设置距离上左下右边的内边距，默认都为0。这里要说明下top,left,bottom,right正负值的情况，正值是向内缩，负值向外扩。
![正值/负值效果](https://upload-images.jianshu.io/upload_images/2427856-e2dbddab0905976c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过具体应用场景来分析下：
# 应用场景1: UIButton图片的大小
设置button图片的时候（非背景图片backgroundImage）,当设置的图片本身的size比button的size要大时，默认情况下图片会填充整个button。这时有这样的需求：button的大小不变但图片需要调小（一般是为了让button在界面显示大小适中，但点击的范围又要尽量大），而button的imageView的frame设置是无效的。一个解决方法是把原图切小到符合要求的尺寸，再设置图片。另一个简单的方法就是设置imageEdgeInsets。
button的size为100x100，设置的图片size也是100x100。我们可以设置button. imageEdgeInsets = UIEdgeInsetsMake(25, 25, 25, 25)来达到缩小图片的目的。这样button看起来只有50x50大，但实际图片上下左右25范围内仍能触发点击事件。（一些新手是这样实现的：创建了一个100x100的透明的button，然后创建50x50的imageView添加到button上）
# 应用场景2：调整titleLabel和imageView的相对位置
我们知道UIButton的默认布局是图片在左，文字在右的。但很多情况下，需求都要求图片在右文字在左，或者图片文字为上下布局。很多小伙伴都冲动的直接去自定义button了，同样我们只要设置titleEdgeInsets和imageEdgeInsets就能简单的实现这样的需求。我们以一个demo为例：
![创建button](https://upload-images.jianshu.io/upload_images/2427856-d829c1a94d1220ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我拖了一个红色背景100x100的button，设置的图片为50x50。默认情况titleEdgeInsets和imageEdgeInsets都为0。现在我们要把图片和文字调换过来，前面的场景是调整了图片的大小，现在调换位置其实原理是一样的。因为都是左右移动，那么上下边距就不用调节。图片需要右移，也就是图片左边left往内缩右边right往外扩。前面说了正值是内缩，负值是向外扩，现在就可以确定imageEdgeInsets的left为正right需为负。具体数值可以通过titleLabel的frame确定：
```
    CGFloat labelW = _button.titleLabel.frame.size.width;
    _button.imageEdgeInsets = UIEdgeInsetsMake(0, labelW, 0, -labelW);
```
![图片移动](https://upload-images.jianshu.io/upload_images/2427856-234e680a1452a4e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同样的文字需要左移，也就是文字左边left为负往外扩右边right为正往内缩：
```
    CGFloat imageW = _button.imageView.frame.size.width;
    _button.titleEdgeInsets = UIEdgeInsetsMake(0, -imageW, 0, imageW);
```
![label移动](https://upload-images.jianshu.io/upload_images/2427856-f7631837a20093a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
至此图片和文字调换过来了，它们的frame也改变了：
> (lldb) po _button.imageView.frame.origin.x
39.333333333333343
(lldb) po _button.titleLabel.frame.origin.x
10.666666666666666

图片和文字为上下布局时，也是类似的调整，只是设置的就是top和bottom了。
从上面两个场景可以总结出，设置titleEdgeInsets和imageEdgeInsets能实现button的titleLabel和imageView的缩放及平移。
UIButton还有contentEdgeInsets，设置后对图片文字都生效的。
>  _button.contentEdgeInsets = UIEdgeInsetsMake(20, 0, 0, 0);

![content](https://upload-images.jianshu.io/upload_images/2427856-bd73624ac8fa400c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


