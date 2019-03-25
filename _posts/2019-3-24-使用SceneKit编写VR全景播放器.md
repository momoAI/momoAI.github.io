最近用SceneKit做了全景看房的功能，现总结下如何实现的。
先看下最终的效果：

![gif1.gif](https://upload-images.jianshu.io/upload_images/2427856-38b405125d79a753.gif?imageMogr2/auto-orient/strip)

#### VR图片全景播放器有以下功能:
- 360度
- 手势滑动，缩放
- 陀螺仪
- 分屏（VR眼镜）
- 热点hotpot
- 头控/eyepick

手势滑动，缩放，陀螺仪功能都是调节球面图片显示的位置；
热点和头控功能本质是一样的，都是在原有模型上增加3维的视图。它们用途不一样，头控功能（全景图片一般就是eyepick功能）一般是戴VR眼镜后，通过模型的位置触发控制事件。
展示全景图的原理很简单：将图片渲染至球体模型内表面上，手机处于球体中心（图中红色区域），当旋转手机的时候，
球体向相反的方向旋转，这样我们就可以看到球体上的画面了。

![球](https://upload-images.jianshu.io/upload_images/2427856-90bd216786daf817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####怎么将图片绘制于球体上呢？
这需要使用openGL这个框架，openGL渲染球体图片步骤大致如下：
1. 生成顶点数据，也就是球面上点坐标数据。顶点越多生成的球体越平滑，但也有极限，当顶点大于一定值的时候再多的顶点也看不出差别来反而会影响性能。

![顶点数据](https://upload-images.jianshu.io/upload_images/2427856-944cde5f164d2259.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 生成纹理数据，也就是图片的颜色缓存数据。
3. 着色器将颜色数据渲染至顶点上。

#### 全景播放器第三方库
- [MD360Player4iOS](https://github.com/ashqal/MD360Player4iOS)：支持全景图片/视频，有分屏/陀螺仪/手势移动功能，但没有热点及头控功能；

- [Panorama](https://github.com/robbykraft/Panorama)：只支持全景图片，比较轻量。也只有分屏/陀螺仪/手势功能；

- [PanoramaGL](https://github.com/shaojiankui/PanoramaGL)：只支持全景图片，具有陀螺仪/手势/热点功能，但这个库比较久远仍是MRC，没人维护；

- [得图SDK](http://www.detu.com/develop/page/41)：支持全景图片/视频，也只有分屏/陀螺仪/手势移动功能

现在主流的和全景图片有关的三方库，基本上都没有热点及头控功能；之前有试过在MD360Player4iOS基础上增加这两个功能，但因为自己openGL零基础后来还是暂时放弃了。
后来发现系统SceneKit框架也可以实现以上所有功能，使用起来也非常简单。接下来我们来了解下SceneKit，看如何实现全景播放功能。

#### SceneKit
（全景视频播放器需使用SpriteKit，这里主要先介绍图片播放器，之后再讲视频播放器）
SceneKit是什么？
> SceneKit is a high-level 3D graphics framework that helps you create 3D animated scenes and effects in your apps. It incorporates a physics engine, a particle generator, and easy ways to script the actions of 3D objects so you can describe your scene in terms of its content — geometry, materials, lights, and cameras — then animate it by describing changes to those objects.
SceneKit是一个高级的3D图形框架，它帮助您在应用程序中创建3D动画场景和效果。它包含了一个物理引擎，一个粒子发生器，以及简单的方法来编写3D对象的动作脚本，这样你就可以用它的内容来描述你的场景——几何，材料，灯光和摄像机——然后通过描述这些对象的变化来动画它。

SceneKit是处理3D图形的，在介绍怎么使用SceneKit 时。我们先来看下与3D有关的知识：坐标系与旋转表达式。
- SceneKit的3D坐标系为右手坐标系：
这个坐标系没有单位，而是根据屏幕的宽度和高度进行相对运算，屏幕上边为1 下边为-1 左边为 -1 右边为 1 。
请牢记这个坐标系，接下来有关图形处理都绕不开它。

![坐标系](https://upload-images.jianshu.io/upload_images/2427856-7b36903813ea7e64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 旋转表达式
旋转表达式主要有四种：
    1. 轴角 2. 欧拉角 3. 四元素 4. 旋转矩阵
这篇[博客]([https://blog.csdn.net/silangquan/article/details/39008903](https://blog.csdn.net/silangquan/article/details/39008903)
)大概介绍了这四种表达式。旋转表达式主要处理模型在空间位置的旋转，全景图片播放时需要用到。

SceneKit比较强大，类比较多，接下来只主要介绍与实现全景有关的几个类：
- SCNView
SCNView主要负责显示3D模型对象的视图，能够添加到UIView类型的视图上。
- SCNScene
场景：由几何模型，灯光，照相机及其他属性组成的环境。场景能添加各种节点，
他包含了一个rootNode（根节点）属性，可以添加各种node。
- SCNNOde
节点：一个抽象的概念，是个看不见摸不到的东西，没有几何形状，但是有位置，以及自身坐标系。在场景中添加节点后，就可以在这个节点上放我们的元素了，比如几何模型，灯光，摄像机等。节点上可以添加子节点的，每个节点都有自身坐标系。
它的属性包含：camera geometry position rotation eulerAngles pivot orientation等，其中rotation eulerAngles pivot orientation就是各种旋转表达式，可以处理模型在空间的角度。
- SCNGeometry
几何模型：全景图片就是渲染在模型上的然后显示在屏幕上。系统自带的模型有很多种：SCNPlane  SCNBox  SCNSphere  SCNCylinder SCNText。我们也可以通过SCNShape自定义各种奇形怪状的模型。
- SCNCamera
相机（观察者）：这个类似我们现实中的相机，它也有焦距、视角等。图形渲染到模型后，要添加相机我们才能看见。
    1. 视角：xFov yFov(默认60度)，视角越大，屏幕上显示的体积越小；
    2. 焦距：focusDistance(默认2.5)，焦距越大，视角越小；

![camera](https://upload-images.jianshu.io/upload_images/2427856-3abd58d022d74de5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- SCNAction
动画：可以为节点添加各种动画，包括：移动，旋转，缩放，自定义…

怎么设置才能将图片渲染至模型上呢？这里需要先理解SCNGeometry的相关几个属性：
- materials（SCNMaterial类）：材质，要渲染的图片就是添加到材质上。一个模型可以添加多个材质，默认有一个材质，可以通过firstMaterial属性获取。
- cullMode（SCNMaterial属性）：渲染时剔除的表面，SCNCullModeBack内表面，SCNCullModeFront外表面。
- diffuse（SCNMaterial属性）：
  > The diffuse property specifies the amount of light diffusely reflected from the surface. The diffuse light is reflected equally in all directions and is therefore independent of the point of view.
漫反射属性指定从表面漫反射的光量。漫射光在各个方向上反射均匀，因此与视点无关。
- contents（diffuse.contents）：渲染的内容，可以是颜色，图片，图层，路径，纹理等。
全景图片渲染设置：geometry.firstMaterial.diffuse.contents = image；就可以了。

理解了一些基本知识后，开始编写代码：
##### 显示图片
```
    // 初始化scene
    _scnView = [[SCNView alloc] init];
    _scnView.scene = [SCNScene scene];
    [self.view addSubview:_scnView];

    // 绘制球体
    SCNSphere *sphere = [SCNSphere sphereWithRadius:_config.shpereRadius];
    // 前面提过坐标系是根据屏幕相对运算的，具体值可以根据显示效果调节，这里球体radius设置为10，

    sphere.firstMaterial.cullMode = SCNCullModeFront; // 剔除球体外表面
    sphere.firstMaterial.doubleSided = NO; // 只渲染一个表面
    // 相机是处于球体内部的，
    _sphereNode = [SCNNode node]; // 节点
    _sphereNode.geometry = sphere;
    _sphereNode.position = SCNVector3Make(0, 0, 0); // 位置（屏幕中心）
    // 渲染图片
    sphere.firstMaterial.diffuse.contents = _config.contents;
    [_scnView.scene.rootNode addChildNode:_sphereNode]; // 添加至场景根节点
```
到这里，一个内表面显示图片的球体创建并添加成功，但是现在view上面并不显示，还需要添加相机节点：
```
    // 相机
    _camera = [SCNCamera camera];
    _camera.automaticallyAdjustsZRange = YES; // 自动添加可视距离
    _camera.xFov = _config.cameraFocalX; // 相机视角
    _camera.yFov = _config.cameraFocalY;
    _camera.focalBlurRadius = 0; // 模糊
    _cameraNode = [SCNNode node];
    _cameraNode.camera = _camera;
    [_scnView.scene.rootNode addChildNode:_cameraNode];
```
然后运行代码，手机屏幕上就能看到图片了。

![demo](https://upload-images.jianshu.io/upload_images/2427856-600ef5b287db6c8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*如果仔细对比原始的平铺图片会发现，现在显示的图片是反过来的，是镜像的；这是因为图片是贴在球体上，而我们的相机是从球体中心往外观察的，类似于现实世界中我们在房间里看贴在窗户玻璃外的窗花一样*
我们如何让它正常显示呢？前面分析过图片渲染的原理，关键的一点就是纹理，那么翻转纹理坐标就能解决这个问题了：
```
    sphere.firstMaterial.diffuse.contentsTransform = SCNMatrix4MakeScale(-1, 1, 1);
    sphere.firstMaterial.diffuse.contentsTransform = SCNMatrix4Translate(sphere.firstMaterial.diffuse.contentsTransform, 1, 0, 0);
```
这里使用了矩阵操作，先把坐标沿y轴翻转实现镜像，翻转后坐标偏移了所以接着需要平移回来。
还有一种方式，翻转后不平移，而是指定超出纹理坐标范围的纹理映射行为SCNWrapMode：mode有以下四种

![wrap](https://upload-images.jianshu.io/upload_images/2427856-4a8e9673e99f6e0f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

指定repeat即可
```
    sphere.firstMaterial.diffuse.contentsTransform = SCNMatrix4MakeScale(-1, 1, 1);
    sphere.firstMaterial.diffuse.wrapS = SCNWrapModeRepeat;
    sphere.firstMaterial.diffuse.wrapT = SCNWrapModeRepeat;
```

但这时仅仅显示了全景图的一部分，并不支持360度查看及陀螺仪查看等功能。我们可以添加手势及陀螺仪来控制全景图的360度滑动：

##### 手势滑动，缩放功能
在scnView父视图上添加两个手势：pinchGesture，panGesture。根据手势操作，调节相机的参数实现相应功能：
```
- (void)addGesture {
    self.pinchGesture = [[UIPinchGestureRecognizer alloc]initWithTarget:self action:@selector(pinchGesture:)];
    self.panGesture = [[UIPanGestureRecognizer alloc]initWithTarget:self action:@selector(panGesture:)];
    [self addGestureRecognizer:_pinchGesture];
    [self addGestureRecognizer:_panGesture];
    _pinchGesture.enabled = _config.pinchEnabled;
    _panGesture.enabled = _config.panEnabled;
}

- (void)pinchGesture:(UIPinchGestureRecognizer *)gesture {
    if (gesture.state != UIGestureRecognizerStateEnded && gesture.state != UIGestureRecognizerStateFailed) {
        if (gesture.scale != NAN && gesture.scale != 0.0) {
            float scale = gesture.scale - 1;
            if (scale < 0) {
                scale *= (_config.scaleMax - _config.scaleMin);
            }
            _currentScale = scale + _prevScale;
            _currentScale = [self validateScale:_currentScale]; // 控制缩放的最小最大比例
            CGFloat valScale = [self validateScale:_currentScale];
            double xFov = _config.cameraFocalX * (1 - (valScale - 1));
            double yFov = _config.cameraFocalY * (1 - (valScale - 1));
            // 调节相机视角，前面分析了视角越大看到的体积越小，所以这里要反过来。即手势放大时，视角要调小这样看到的图像才是放大的效果；
            _camera.xFov = xFov;
            _camera.yFov = yFov;
        }
    } else if(gesture.state == UIGestureRecognizerStateEnded){
        _prevScale = _currentScale;
    }
}

- (void)panGesture:(UIPanGestureRecognizer *)gesture {
    // 控制图片滑动原理：手势滑动，效果是手机屏幕上的图片要跟着滑动，
    //  因为我们的图片是渲染至球体上的，所以可以控制球体转动来实现滑动效果。
    // 一般的，我们都是控制相机(观察者)。因为相机处于球体内部，相机需要往相反的方向转动。
    if (gesture.state == UIGestureRecognizerStateBegan){
        CGPoint currentPoint = [gesture locationInView:gesture.view];
        self.lastPointX = currentPoint.x;
        self.lastPointY = currentPoint.y;
    }else{
        CGPoint currentPoint = [gesture locationInView:gesture.view];
        float distX = currentPoint.x - self.lastPointX;
        float distY = currentPoint.y - self.lastPointY;
        self.lastPointX = currentPoint.x;
        self.lastPointY = currentPoint.y;
        // 手势滑动角度的微调
        distX *= - 0.005 * 0.5;
        distY *= - 0.005 * 0.5;
        SCNMatrix4 modelMatrix = SCNMatrix4Identity;
        if (fabs(distX)  > fabs(distY)) {
            self.fingerRotationY += distX;
        }else {
            self.fingerRotationX += distY;
        }
        // 因为是右手坐标系，所以相机水平转动时是绕Y轴转动，垂直方向转动时需绕X轴转动。Z轴保持不变。这里旋转表达式用的是旋转矩阵
        modelMatrix = SCNMatrix4Rotate(modelMatrix, self.fingerRotationY, 0, 1, 0);
        modelMatrix = SCNMatrix4Rotate(modelMatrix, self.fingerRotationX,1, 0, 0);
        _cameraNode.pivot = modelMatrix;
    }
}

- (float)validateScale:(float)scale{
    if (scale < _config.scaleMin) {
        scale = _config.scaleMin;
    }else if (scale > _config.scaleMax) {
        scale = _config.scaleMax;
    }
    return scale;
}

```

##### 陀螺仪功能
陀螺仪功能是让图片跟着手机的方位转动，原理和手势滑动一样：
```
- (void)addMotionFunction {
    _motionManager = [[CMMotionManager alloc]init];
    _motionManager.deviceMotionUpdateInterval = 1.0 / 30.0;
    _motionManager.gyroUpdateInterval = 1.0f / 30;
    _motionManager.showsDeviceMovementDisplay = YES;
    if (_motionManager.isDeviceMotionAvailable) {
        [_motionManager startDeviceMotionUpdatesToQueue:[NSOperationQueue mainQueue] withHandler:^(CMDeviceMotion * _Nullable motion, NSError * _Nullable error) {
            if (!self.config.motionEnabled) {
                return;
            }
            CMAttitude *attitude = motion.attitude;
            if (attitude == nil) {
                return;
            }
            //                self.cameraNode.eulerAngles = SCNVector3Make(attitude.pitch - M_PI / 2 , attitude.roll, attitude.yaw);
            // 这里旋转表达式用的是四元素(陀螺仪返回的attitude.quaternion就是四元素)
            self.cameraNode.orientation = [self orientationFromCMQuaternion:attitude.quaternion];
        }];
    }
}

- (SCNQuaternion)orientationFromCMQuaternion:(CMQuaternion)quaternion {
    GLKQuaternion gq1 = GLKQuaternionMakeWithAngleAndAxis(GLKMathDegreesToRadians(- 90), 1, 0, 0);
    // 这里x轴要同时旋转90度，这是因为手机陀螺仪的坐标系不一致：手机正放于桌面上的坐标为(0,0,0);而scnView坐标系是手机正立的时候为(0,0,0)；

    GLKQuaternion gq2 = GLKQuaternionMake(quaternion.x, quaternion.y, quaternion.z, quaternion.w);
    GLKQuaternion qp  = GLKQuaternionMultiply(gq1, gq2);
    return SCNVector4Make(qp.x, qp.y, qp.z, qp.w);
}
```
##### 添加遮罩
大部分全景图片都是由全景相机拍摄出来的，全景相机是360度的，在拍摄时相机底部的支架也会拍摄进去：

![支架](https://upload-images.jianshu.io/upload_images/2427856-194132d56259e668.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了美观，不影响整体效果 ，我们需要用一张图片盖住。怎么在球面图形上面加张图片呢？其实我们只要在创建一个渲染图片的平面模型，找准位置添加到场景rootNode上就可以了：
```
    _overlayNode = [SCNNode node];
    _overlayNode.geometry= [SCNPlane planeWithWidth:1 height:1];
    _overlayNode.geometry.firstMaterial.diffuse.contents = overlayIcon; // 图片
    _overlayNode.position = SCNVector3Make(0, - 4, 0);  // 支架位于相机正下方，也就是坐标系Y轴负方向
    _overlayNode.rotation = SCNVector4Make(1, 0, 0, - M_PI / 2); // 旋转 否则看不到
    // 这里旋转90度 还是坐标的原因：默认情况下添加的SCNPlane模型是平铺在XY平面，而我们添加的遮罩X,Z都是0，所以需要旋转至XZ平面才能看到遮罩

    _overlayNode.geometry.firstMaterial.cullMode = SCNCullModeBack;
    [_scnView.scene.rootNode addChildNode:_overlayNode];
```
![遮罩](https://upload-images.jianshu.io/upload_images/2427856-638952cc59f86691.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### 头控功能(eyepick)
其原理和上面的添加遮罩是一样的，都是在场景中添加节点。不过这些节点需要触发事件，实现相关的控制功能。这里的控制功能基本都是控制切换上一张图片，下一张图片，实现头戴设备后也能实现查看图集的需求。
```
    // 添加头控节点 
    _potNode = [SCNNode node]; // 选择pick节点
    _potNode.geometry= [SCNPlane planeWithWidth:0.3 height:0.3];
    _potNode.geometry.firstMaterial.diffuse.contents = potIcon;
    _potNode.position = SCNVector3Make(0, 0, - 9); 
    _potNode.geometry.firstMaterial.cullMode = SCNCullModeBack;
    [_cameraNode addChildNode:_potNode]; // 加在_camera上，camera转动时保持不变
    
    _preNode = [SCNNode node]; // 上一张图片function节点
    _preNode.geometry= [SCNPlane planeWithWidth:0.3 height:0.3];
    _preNode.geometry.firstMaterial.diffuse.contents = preIcon;
    _preNode.position = SCNVector3Make(- 1.5, 0.5, - 9);
    _preNode.geometry.firstMaterial.cullMode = SCNCullModeBack;
    [_sphereNode addChildNode:_preNode];

    _nextNode = [SCNNode node]; // 下一张图片function节点
    _nextNode.geometry= [SCNPlane planeWithWidth:0.3 height:0.3];
    _nextNode.geometry.firstMaterial.diffuse.contents = nextIcon;
    _nextNode.position = SCNVector3Make(1.5, 0.5, - 9);
    _nextNode.geometry.firstMaterial.cullMode = SCNCullModeBack;
    [_sphereNode addChildNode:_nextNode];
```
节点添加完后，并正常显示了，接下来就要加上触发事件，触发的时机就是当function节点和pick节点重合的时候。只判断重合还不够，因为在浏览图片时，相机转动时偶发情况下function节点和pick节点碰巧重合。因此在重合的基础上，还需加上延时动画，当重合的时间达到动画的时间后才触发事件。
```
// 添加头控动画
- (void)addEyepickerAnimation {
    _animationNode = [SCNNode node];
    _animationNode.geometry = [SCNPlane planeWithWidth:0.3 height:0.3];
    _animationNode.hidden = YES;
    [_potNode addChildNode:_animationNode];
    __weak typeof(self) weakSelf = self;
    _animationAction = [SCNAction customActionWithDuration:3.f actionBlock:^(SCNNode * _Nonnull node, CGFloat elapsedTime) {
        int time = (int) (elapsedTime * (images.count - 1) / 3.0);
        node.geometry.firstMaterial.diffuse.contents = images[time];
        if (time == images.count - 1 && (weakSelf.isPreAnimating || weakSelf.isNextAnimating)) { // 动画结束
            FWPanoramaHotpotType type = [weakSelf.animationKey isEqualToString:@"pre"] ? FWPanoramaHotpotTypePrev : FWPanoramaHotpotTypeNext;
            if (type == FWPanoramaHotpotTypePrev) {
                weakSelf.preAnimationEnd = YES;
                [weakSelf removePreAnimation];
            }else {
                weakSelf.nextAnimationEnd = YES;
                [weakSelf removeNextAnimation];
            }
            if ([weakSelf.delegate respondsToSelector:@selector(renderView:didPickHotpot:)]) {
                [weakSelf.delegate renderView:weakSelf didPickHotpot:type];
            }
        }
    }];
}

// scnView的代理方法，图片渲染都会走这里
- (void)renderer:(id <SCNSceneRenderer>)renderer updateAtTime:(NSTimeInterval)time {
    SCNVector3 prePosition = [_preNode convertPosition:_preNode.position toNode:_cameraNode]; // 计算相对坐标
    SCNVector3 nextPosition = [_nextNode convertPosition:_nextNode.position toNode:_cameraNode];
//    NSLog(@"camera  x;%f,y:%f,z:%f",prePosition.x,prePosition.y,prePosition.z);
    BOOL preOverlap = prePosition.x > - 0.3 / 2 && prePosition.x < 0.3 / 2 && prePosition.y > - 0.3 / 2 && prePosition.y < 0.3 / 2;
    if (!_preAnimationEnd && preOverlap) {
        // 两个node基本重合
        if (!_isPreAnimating) {
            [self runPreAnimation];
        }
    }else if (!_isNextAnimating && !preOverlap) {
        _preAnimationEnd = NO;
        [self removePreAnimation];
    }
    
    BOOL nextOverlap = nextPosition.x > - 0.3 / 2 && nextPosition.x < 0.3 / 2 && nextPosition.y > - 0.3 / 2 && nextPosition.y < 0.3 / 2;
    if (!_nextAnimationEnd && nextOverlap) {
        // 两个node基本重合
        if (!_isNextAnimating) {
            [self runNextAnimation];
        }
    }else if (!_isPreAnimating && !nextOverlap) {
        _nextAnimationEnd = NO;
        [self removeNextAnimation];
    }
}
```

##### 节点点击事件
上面两个eyepick节点的事件，是由头控触发的；那如果我们要做到通过手动点击节点来触发事件，该怎么做呢？
1. 首先，我们需要拿到手点击屏幕的坐标；
2. 然后通过这个坐标，计算该点对应的节点；
3. 如果有对应的节点，再判断是否是我们需要的目标节点；
```
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    // 获取到手势的对象
    UITouch *touch = [touches allObjects].firstObject;
    
    // 手势在SCNView中的位置
    CGPoint touchPoint = [touch locationInView:self.scnView];
    
    // 该方法会返回一个SCNHitTestResult数组，这个数组中每个元素的node都包含了指定的点
    NSArray *hitResults = [self.scnView hitTest:touchPoint options:nil];
    if (hitResults.count > 0) {
        SCNHitTestResult *hit = [hitResults firstObject];
        SCNNode *node = hit.node;
        if (node == _preNode) {
            NSLog(@"hit prenode");
        }else if (node == _nextNode) {
            NSLog(@"hit nextnode");
        }
    }
}
```
（以上代码由楼下junior_a提供）

##### 分屏功能
实现分屏，就是将1个scnView分成两个，这两个scnView的显示和操作都是一样的。要实现这种效果，可以添加两个subview并将scnView的contents赋值给两个subview。
```
@property (nonatomic, strong) SCNView *leftView;
@property (nonatomic, strong) SCNView *rightView;

 [_leftView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.right.top.mas_equalTo(0);
            make.height.mas_equalTo(self.bounds.size.height / 2);
        }];
 [_rightView mas_makeConstraints:^(MASConstraintMaker *make) {
            make.left.right.bottom.mas_equalTo(0);
            make.height.mas_equalTo(self.bounds.size.height / 2);
        }];
_leftView.layer.contents = self.scnView.layer.contents;
_rightView.layer.contents = self.scnView.layer.contents;
[self.view addSubview:_leftView];
[self.view addSubview:_rightView];
```

##### 视频播放器
视频播放器，原理和图片播放器是一样的：改动上面的一小段代码，就能实现和图片同样功能的视频播放器;
改动的地方就是将渲染在球体模型上的图片，换成skView包装的视频播放器AVPlayer：
```
- (void)createSphere {
    SCNSphere *sphere = [SCNSphere sphereWithRadius:_config.shpereRadius];
    sphere.firstMaterial.cullMode = SCNCullModeFront; // 剔除球体外表面
    sphere.firstMaterial.doubleSided = NO; // 只渲染一个表面
    _sphereNode = [SCNNode node]; // 节点
    _sphereNode.geometry = sphere;
    _sphereNode.position = SCNVector3Make(0, 0, 0);
    
    // 渲染图片
//    sphere.firstMaterial.diffuse.contents = _config.contents;
//    [_scnView.scene.rootNode addChildNode:_sphereNode];
    
    // 渲染视频
    NSString *path = [[NSBundle mainBundle] pathForResource:@"360" ofType:@"mp4"];
    NSURL *url = [NSURL fileURLWithPath:path];
     AVPlayerItem *item = [AVPlayerItem playerItemWithURL:url];
     _player = [AVPlayer playerWithPlayerItem:item];
    [_player play];

    // 需要使用SpriteKit
    _videoNode = [[SKVideoNode alloc] initWithAVPlayer:_player]; // 播放器节点  
    _videoNode.size = CGSizeMake(self.frame.size.width, self.frame.size.height); // 这里的size的单位和上面讲的SceneKit不一样，这里就是实际的像素点单位 这里设置和当前view一样
    _videoNode.position = CGPointMake(_videoNode.size.width / 2, _videoNode.size.height / 2);
    _skScene = [SKScene sceneWithSize:_videoNode.size];
    _skScene.scaleMode = SKSceneScaleModeAspectFit;
    [_skScene addChild:_videoNode];
    sphere.firstMaterial.diffuse.contents = _skScene;
    [_scnView.scene.rootNode addChildNode:_sphereNode];
}
```
另外，和普通的视频播放器一样，我们可以通过_player对象控制视频的播放（播放/暂停/快进等）

**至此，全景播放器的所有功能都实现了。所有代码也就400行，是不是很简单呢**
觉得有用的[点个赞](https://github.com/momoAI/FWPanoramaView)哈
