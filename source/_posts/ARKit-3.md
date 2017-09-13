---
title: ARKit-真实的灯光与基于物理的渲染
date: 2017-09-12 17:01:53
tags: AR
---
翻译自:https://blog.markdaws.net

相关工程地址:https://github.com/markdaws/arkit-by-example

![Virtual cubes on a counter top](https://cdn-images-1.medium.com/max/1600/1*o848QWJYJxzVJRjbsMnxaQ.png)

在这篇文章我们将会插入更加真实的虚拟物体到场景中。要达到这个目的，我们可以通过用一项技术叫做基于物理的渲染去展示模型的更多细节，同时我们会在场景中用更准确的灯光的表现方式

## 场景灯光

AR的主要目标是把真实世界和虚拟的物体混合起来。有时我们添加的物体是有自己的风格的看起来不真实，还有一些情况下我们想要添加一些物体完美的融入我们交互的实际环境

### 强度

为了达到一个高等级的真实，场景中的灯光效果是非常重要的。如果虚拟场景的灯光可以喝真实世界的灯光效果接近，我们就会感到我们添加的内容是更加真实的

比方说，如果你在一个很暗的屋子里，但是加入了一个用很亮的光照射的物体，这样看这个物体就好像在这个场景之外，相反的一个很暗的物体在一个明亮的房间中同样会感觉这个物体在场景之外

![bright cube in dark room](https://cdn-images-1.medium.com/max/1600/1*EHRiizG-VtzhmIL9RUMbSQ.png)

![dark cube in bright room](https://cdn-images-1.medium.com/max/1600/1*RMolADbOF-bxmmbDKYb6IQ.png)

所以我们回到最开始然后建造一个更搞等级的真实场景。首先，假如你的虚拟场景是没有光源的，然后就像在真实世界一样所有的物体都会是黑色的，因为物体的表面没有任何的光反射，我们把场景的灯去掉以后加入几个立方体，效果如下:

![no light](https://cdn-images-1.medium.com/max/1600/1*VZf42yoYgl8sChkAKyX8vg.png)

现在我们可以给我们的场景加入一些光源，在3D图形学中这有很多种光源你可以加到场景里:

![lights](https://cdn-images-1.medium.com/max/1600/1*8BSG48yy39Y7KqrMZGw5Pw.png)

- Ambient -- 模拟一个从所有方向照射过来的等量光，你可以看到所有方向的光均等的照射在物体上，所以这里没有任何的影子
- Directional -- 方向光只有方向没有光源的位置，可以想象为从无限远的地方发射出来的
- Omni -- 也被叫做Spot Light，这是一种有方向也有位置的光。如果你想表现光源与几何体的具体对光强度的影响这个光非常有用
- Spot -- 和Omni类似，不过除了方向和位置以外，spot light的光会在一个一个圆锥形内强度减弱，就像你桌子上的

### autoenablesDefaultLighting

SceneKit SCNView有一个属性叫做autoenablesDefaultLighting，如果你设置这个为true，SceneKit会添加一个Onmi方向光到场景中，灯光的位置在摄像头的位置，方向沿着摄像头的方向，这是一个很好的开始点并且是默认打开的直到你添加了自己的光源，不过这样简单的设置有一些问题:

- 光强一直是1000，所以在不同光照条件下放置看起来效果不是很好
- 光照方向是一直改变的，所以你绕着物体走一圈，他看起来总是光从你这里招过去的，就像你拿着火炬在你面前，大多数的场景是有固定光源的，所以当你在四周走的时候显得很不自然

在我们的应用中我们需要把这个关掉:

    self.sceneView.autoenablesDefaultLighting = NO;

### automaticallyUpdatesLighting

ARKit中的ARSCNView有一个叫做automaticallyUpdatesLighting的属性，按文档所说会自动根据预测出的光照强度给场景添加光源。听起来很棒，但是我可以告诉你他什么都没有做或者是我使用的方法不对，不过没所谓，我们可以用下面的方法得到预测的灯光

### lightEstimationEnabled

ARSessionConfiguration类有一个属性lightEstimationEnabled，设置这个为true，每一个AR帧我们都可以得到一个lightEstimate值我们可以用它去渲染

我们可以根据每帧得到的光照预测值调整我们添加的光照强度，这样就可以解决我们上面所说的问题

## 光照

首先我们添加一个灯到场景中，我们添加一个spot light方向是直接向下的到场景中距离原点几米。这模拟了我现在所处的环境，在天花板上有一个spot light。代码如下:

    - (void)insertSpotLight:(SCNVector3)position {
    SCNLight *spotLight = [SCNLight light];
    spotLight.type = SCNLightTypeSpot;
    spotLight.spotInnerAngle = 45;
    spotLight.spotOuterAngle = 45;
    SCNNode *spotNode = [SCNNode new];
    spotNode.light = spotLight;
    spotNode.position = position;
    // By default the stop light points directly down the negative
    // z-axis, we want to shine it down so rotate 90deg around the 
    // x-axis to point it down
    spotNode.eulerAngles = SCNVector3Make(-M_PI / 2, 0, 0);
    [self.sceneView.scene.rootNode addChildNode: spotNode];
    }

我们不仅加了一个spot light 还添加了一个ambient light，这是因为在实际场景中我们总是有很多光源，所以添加一个环境光会使效果看着更真实

## 光照估计

最后我们提到了lightEstimation，ARKit可以分析场景然后估计出环境光的强度。他的返回值如果是1000代表着中性值，低于这个值是黑高于这个值亮。想要打开这个功能，你需要在场景设置中把lightEstimationEnabled设置为true

    configuration.lightEstimationEnabled = YES;

你做了这些之后，你就可以在ARSCNViewDelegate实现下面的方法，然后修改你加入到场景中的spot light和ambient light的光照强度

    - (void)renderer:(id <SCNSceneRenderer>)renderer updateAtTime:(NSTimeInterval)time {
    ARLightEstimate *estimate = self.sceneView.session.currentFrame.lightEstimate;
    if (!estimate) {
        return;
    }
    // TODO: Put this on the screen
    NSLog(@"light estimate: %f", estimate.ambientIntensity);
    // Here you can now change the .intensity property of your lights
    // so they respond to the real world environment
    }

## 基于物理的渲染

我们现在有了基础的灯光，截下来我们要用PBR技术给我们的几何体加上贴图，下面是几个基本的信息

- Albedo -- 模型的基础颜色，这是材质的diffuse部分，这是材质的没有烘焙灯光和阴影信息的贴图
- Roughness -- 描述了材质的光滑程度，粗糙的表面反射比较少，光滑的材质会有很亮的高光
- Metalness -- 和光滑程度类似描述材质的金属程度

我们拿到了这些贴图以后用下面的方式渲染材质

    mat = [SCNMaterial new];
    mat.lightingModelName = SCNLightingModelPhysicallyBased;
    mat.diffuse.contents = [UIImage imageNamed:@"wood-albedo.png"];
    mat.roughness.contents = [UIImage imageNamed:@"wood-roughness.png"];
    mat.metalness.contents = [UIImage imageNamed:@"wood-metal.png"];
    mat.normal.contents = [UIImage imageNamed:@"wood-normal.png"];

你需要把材质的lightingModelName设置为SCNLightingModelPhysicallyBased，然后设置这些各式各样材质类型

最后最重要的部分是你必须告诉SCNScene你正在用PBR光照，当你做这些这个场景的光源来自你指定的图片，下面是我是用的图片

![spherical](https://cdn-images-1.medium.com/max/1600/1*rLncYnZmC7YeZVgJ6UnSJQ.png)

所以现在光照是从这幅图片中取得的，就好像这个图片被投影成为整个几何体的背景，SceneKit用这个背景去计算出这个几何体是如何被照射的

    UIImage *env = [UIImage imageNamed: @"spherical.jpg"];
    self.sceneView.scene.lightingEnvironment.contents = env;

最后的部分是拿到光照的估计值然后把这个应用到这个环境图片的强度，我们知道了ARKit返回1000代表中性的光照，而环境光的值是把1.0当做中性，所以我们需要进行一个尺度变换

    CGFloat intensity = estimate.ambientIntensity / 1000.0;
    self.sceneView.scene.lightingEnvironment.intensity = intensity;