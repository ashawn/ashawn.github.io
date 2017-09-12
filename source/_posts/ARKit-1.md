---
title: ARKit-平面检测与可视化
date: 2017-09-10 09:54:10
tags: AR
---
在我们的第一个hello world ARKit应用中我们设置了我们的工程并在真实的世界中渲染了一个虚拟的3D立方体，并且会在你四周移动的时候持续追踪

在这篇文章中，我们会着眼于在真实场景中提取3D几何体的信息并且将它进行可视化。检测几何体对于增强现实的应用是非常重要的，因为如果你想做到和在真实世界一样的交互你需要知道用户点击了桌子上、正在看地板或者其他和平常生活类似的3D交互

ARKit可以检测平面。一旦我们检测到了一个平面，我们可以把他进行可视化，然后把这个平面的规格和角度展现出来

## 计算机视觉

在我们深入代码之前，理解清楚在ARKit下面发生了什么是非常有用的，因为这个技术还不完美，有很多情况会降低或者影响到你的应用的表现

AR的目的就是可以在真实世界的特别的点上插入虚拟的内容，并且当你在真实的世界移动的时候可以持续的追踪。ARKit的基本处理流程会把iOS的摄像头采集到的视频流的每一帧进行处理然后把特征点提取出来。特征点可以是很多东西，但是你如果想去在图片中检测出有趣的特征点你可以在多个帧中进行追踪。一个特征点可能是一个物体的一个角或者一个布的纹理的一个边等等。有很多方法去生成这些特征点，你可以在网上读到更多的信息，但对于我们的目标指导我们可以从一个图像中提取到很多独特的进行认证的特征点就够了

一旦你有了这些特征点，你就可以在多帧之间追踪这些特征点不论你走到哪里，你可以获取这些相关点然后估计3D姿态信息，比如说当前的摄像头位置以及这些特征点的位置。当用户移动的越多，我们就会获得更多的特征点，这些3D姿态估计也会得到改善

在平面检测中，一旦你有了一定数量的特征点，你就可以尝试把平面对着那些点放到合适的位置，然后在尺寸、角度和位置上找到最佳的匹配。ARKit在持续分析这些特征点然后在代码中向我们报告所有他找到的平面

下面是我手机在检测我的沙发扶手的一个截屏，你可以看到这个布有很好的纹理，大量的独特又有趣的特征点可以被追踪到，每一个十字星就是一个ARKit找到的独特的特征点

![sofafeatures](https://cdn-images-1.medium.com/max/1600/1*WY5UmghrTfumK6PEDn-hWw.png)

下一张图片是我照我的冰箱门的，你可以注意到这并没有几个特征点

![doorfeatures](https://cdn-images-1.medium.com/max/1600/1*bvNKXemL0gGOPMlQ3H0zeQ.png)

这是非常重要的，因为为了让ARKit检测到特征点你必须摘到有足够多的特征点的东西去检测。那些会造成糟糕的特征点提取的情况有:

1. 糟糕的光线--弱光或者带有高光反射的强光。杜绝有糟糕光线的环境
2. 缺少纹理--如果你把相机对准一个白墙，这里真的没有什么特别的东西用来提取，ARKit就不能找到并且追踪你。杜绝对准一些纯色或者发光的区域
3. 快速移动--这对于ARKit是情况各异的，通常如果你只是用图片去检测和估计3D姿态，如果你移动摄像头太快你就会得到一些糟糕的图片这会造成追踪的失败。不过ARKit用了叫做VIO的算法，所以ARKit除了用图像信息以外还用到了设备运动传感器的信息去估计用户的姿态。这让ARKit在追踪上面很健壮

## 添加可视化的调试信息

在我们开始之前，给应用添加一些调试信息是非常有用的,也就是渲染原始的世界的同时也把ARKit检测到的特征点信息渲染出来，这是很有用的可以让你知道你是不是在一个可以追踪的很好的地方。要做到这个你可以在我们的ARSCNView实例中打开调试选项:

    self.sceneView.debugOptions = ARSCNDebugOptionShowWorldOrigin | ARSCNDebugOptionShowFeaturePoints;

## 检测平面几何体

在ARKit中你可以通过在你的会话配置中设置planeDetection属性指定你想要检测水平面。这个值可以被设置为ARPlaneDetectionHorizontal或者ARPlaneDetectionNone

一旦你设置了这个属性，你就开始得到ARSCNViewDelegate的回调信息。这里有很多的方法，我们要用到的第一个是

    /**
    Called when a new node has been mapped to the given anchor.
    @param renderer The renderer that will render the scene.
    @param node The node that maps to the anchor.
    @param anchor The added anchor.
    */
    - (void)renderer:(id <SCNSceneRenderer>)renderer
        didAddNode:(SCNNode *)node
        forAnchor:(ARAnchor *)anchor {
    }

这个方法在ARKit每次认为他检测到了一个新的平面时进行回调。我们会得到两个信息，node和anchor。这个SCNNode实例是一个ARKit创建的SceneKit node，他有一些属性可以设置比如位置和角度，然后我们还得到了anchor的实例，这个告诉我们这个被找到的anchor更多的信息，比如说尺寸以及这个平面的中心

这个anchor是ARPlaneAnchorl类型的，从这里我们可以得到这个平面的扩展和中心信息

## 渲染平面

有了上面的信息，我们现在可以在虚拟的世界里画一个SceneKit 3D平面。我们创建了一个Plane类继承自SCNNode去做这件事情。在构建方法里面我们创建了这个平面以及设置他的尺寸:

    // Create the 3D plane geometry with the dimensions reported
    // by ARKit in the ARPlaneAnchor instance
    self.planeGeometry = [SCNPlane planeWithWidth:anchor.extent.x height:anchor.extent.z];
    SCNNode *planeNode = [SCNNode nodeWithGeometry:self.planeGeometry];
    // Move the plane to the position reported by ARKit
    planeNode.position = SCNVector3Make(anchor.center.x, 0, anchor.center.z);
    // Planes in SceneKit are vertical by default so we need to rotate
    // 90 degrees to match planes in ARKit
    planeNode.transform = SCNMatrix4MakeRotation(-M_PI / 2.0, 1.0, 0.0, 0.0);
    // We add the new node to ourself since we inherited from SCNNode
    [self addChildNode:planeNode];

既然我们有了自己的Plane类，回到ARSCNViewDelegate的回调方法，我们可以在ARKit报告了新的Anchor时创建我们的Plane:

    - (void)renderer:(id <SCNSceneRenderer>)renderer 
        didAddNode:(SCNNode *)node 
        forAnchor:(ARAnchor *)anchor {
    if (![anchor isKindOfClass:[ARPlaneAnchor class]]) {
        return;
    }
    Plane *plane = [[Plane alloc] initWithAnchor: (ARPlaneAnchor *)anchor];
    [node addChildNode:plane];
    }

## 更新平面

如果你运行了上面的程序，当你向四周走动时你可以看到新的平面在虚拟世界中渲染出来，尽管这些平面不是在你走动的时候合适的增长的。ARKit在持续的分析这个场景，当它发现一个平面比他之前预计的大了或者笑了的时候，他会更新这个平面的扩展值。所以我们需更新已经渲染了的Plane

我们可以通过ARSCNViewDelegate中的一个方法得到这些更新的信息:

    - (void)renderer:(id <SCNSceneRenderer>)renderer 
    didUpdateNode:(SCNNode *)node 
        forAnchor:(ARAnchor *)anchor {
    // See if this is a plane we are currently rendering
    Plane *plane = [self.planes objectForKey:anchor.identifier];
    if (plane == nil) {
        return;
    }
    [plane update:(ARPlaneAnchor *)anchor];
    }

在我们的Plane类的update方法中我们可以更新平面的宽度和高度:

    - (void)update:(ARPlaneAnchor *)anchor {
    self.planeGeometry.width = anchor.extent.x;
    self.planeGeometry.height = anchor.extent.z;
    // When the plane is first created it's center is 0,0,0 and 
    // the nodes transform contains the translation parameters. 
    // As the plane is updated the planes translation remains the 
    // same but it's center is updated so we need to update the 3D
    // geometry position
    self.position = SCNVector3Make(anchor.center.x, 0, anchor.center.z);
    }

现在我们已经让平面渲染和更新了，我们可以进到应用里面看看了

## 提取信息的结果

下面是我检测的视频的一些截图

这是厨房的，ARKit做得很好，准确的找到了平面的扩展信息和角度信息，正好贴合这个平面

![kitchenplane](https://cdn-images-1.medium.com/max/1600/1*9wAJaYgZ7nBsj9Oap3iuIA.png)

这是地板的，你可以看到当你在四周走动的时候，ARKit在持续的铺满新屏幕，这在你开发应用的时候是一个要注意的点，用户必须一上来就在四周走动一圈然后再放置东西，所以在几何体还不是足够好到能用的时候给用户好的视觉上的提示是一个很重要的部分

![floorplane](https://cdn-images-1.medium.com/max/1600/1*dP6AvQJ52rcuFPyt2QEmyA.png)

下面是和上面同一场景几秒钟之后的效果，ARKit已经把所有的平面整合成了一个平面。提示，在ARSCNViewDelegate中你需要处理这样的情况当ARKit合并几个平面时会删除一些ARPlaneAnchor的实例

![floorplane2](https://cdn-images-1.medium.com/max/1600/1*51G5dcUbKXe__op2mLBzTA.png)

这是一个很有趣的场景，我在楼上距离地面大概有12到15英尺，在糟糕的光线条件下，ARKit依然可以提取出一个平面，难以置信！

![aboveplane](https://cdn-images-1.medium.com/max/1600/1*HAKUQCP9a7hAyjK55UNXeg.png)

这是一个梯子旁边的比较窄的墙，注意平面在实际的表面边缘超出部分是怎么延伸的

![smallplane](https://cdn-images-1.medium.com/max/1600/1*h1Mcnnn46HHVWLytd6xpIg.png)

## 重要的结论

1. 不要期望于一个平面可以完美的对齐表面，你可以从视频里面看出来，平面被检测出来，但是角度可能会不准确，所以你如果开发一个应用想要得到一个特别准确的几何体你可能会失望

2. 边缘不是特别好，平面经常会大一点或者小一点，不要试着做一个需要完美的对齐的应用

3. 追踪很健壮并且很快，你可以看到我移动的很快但是追踪的依然很棒

4. 特征点检测的效果很好，在糟糕的光线和较远的距离下依然检测到了一些平面
