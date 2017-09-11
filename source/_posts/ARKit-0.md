---
title: ARKit-hello world
date: 2017-09-08 15:18:21
tags: ARKit
---
## 要求
iOS11 Xcode9 iPhone6S+

## 创建工程

打开Xcode9，新建工程，选择ARKit工程模板，如下图所示：

![ARKitTemplate](https://cdn-images-1.medium.com/max/1600/1*4d-Qovl_HSLbeEEB1MDEdg.png)

在选择Content Technology时要选择“Scene Kit”，这是用于3D渲染的，默认的“Sprite Kit”是用于2D渲染的，如下图所示：

![SceneKit](https://cdn-images-1.medium.com/max/1600/1*YxNXFEY8IHUVz_Hde7GXeA.png)

运行程序，你可以看到一个飞机，如下图所示：

![HelloAR](https://cdn-images-1.medium.com/max/1600/1*Azhl1SRVJD8680S4Qxznjw.png)

## ARKit核心类

ARSCNView:一个View类，它通过SceneKit帮助我们将实际的场景与3D的内容渲染在一起，它做了以下事情：
1. 将摄像头的数据实时渲染到场景中作为3D场景的背景
2. 统一ARKit与SceneKit的坐标系
3. 匹配SceneKit的Camera与ARKit的追踪信息

ARSession:所有的AR会话都需要一个ARSession的实例，它用来控制camera以及获取传感器的信息。每个ARSCNView的实例都有一个ARSession的实例，所以你只需要在开始的时候对它进行一些设置

ARWorldTrackingConfiguration:这个类表明ARSession要使用6自由度在真实世界跟踪用户，包括位移和旋转。这让我们不仅仅可以在原地看到增强的内容，还可以在3D的空间内移动物体。如果你不需要这个移动的部分，用户会和你的增强内容呆在一起你可以使用ARSessionConfiguration去初始化你的ARSession实例

回到我们的工程，我们可以看到在viewWillAppear方法中ARSession实例被初始化了，其中self.sceneView是一个ARSCNView的实例，代码如下所示：

    - (void)viewWillAppear:(BOOL)animated {
        [super viewWillAppear:animated];
        // Create a session configuration
        ARWorldTrackingSessionConfiguration *configuration =[ARWorldTrackingSessionConfiguration new];
        // Run the view's session
        [self.sceneView.session runWithConfiguration:configuration];
    }

## 绘制一个立方体

我们现在要用Scenekit绘制一个3D的立方体。SceneKit有很多基础类，SCNScene是一个所有3D内容的容器，你可以加很多不同位置，不同旋转角度，不同尺度的3D的集合体到这个容器里面

要往场景里面加内容，首先你需要创建这些集合体，可以使很复杂的也可以很简单像球、立方体和平面。然后把这个集合体和场景的一个节点绑定在一起把它加到场景中，SceneKit会把这个包含内容的场景渲染出来

我们可以同过在viewDidLoad方法中添加以下代码添加一个场景并在场景中画一个立方体:

    - (void)viewDidLoad {
    [super viewDidLoad];
    // Container to hold all of the 3D geometry  
    SCNScene *scene = [SCNScene new];
    // The 3D cube geometry we want to draw
    SCNBox *boxGeometry = [SCNBox 
                            boxWithWidth:0.1 
                            height:0.1 
                            length:0.1 
                            chamferRadius:0.0];
    // The node that wraps the geometry so we can add it to the scene
    SCNNode *boxNode = [SCNNode nodeWithGeometry:boxGeometry];
    // Position the box just in front of the camera
    boxNode.position = SCNVector3Make(0, 0, -0.5);
    // rootNode is a special node, it is the starting point of all
    // the items in the 3D scene
    [scene.rootNode addChildNode: boxNode];
    // Set the scene to the view
    self.sceneView.scene = scene;
    }

ARKit的坐标系是以米为单位的，所以在这里我们创建的是10X10X10厘米的盒子

ARKit和SceneKit的坐标系看起来和下图一样:

![coordinate](https://cdn-images-1.medium.com/max/1600/1*fOeokerm3049ltZhG6e-vQ.png)

你可以看到上面的代码我们把位置设置为camera前面的-0.5单位，因为camera是面向Z轴负方向的

当ARSession启动的时候camera的位置会被初始化到0，0，0

现在你运行这个程序，你会看到一个小的3D的立方体在这个空间中，他会始终保持他的位置不管你往哪里移动，你可以走到周围观察它的上下

一个烦恼是我们想加入一些默认的灯光在这个3D的场景中，然后我们就可以看到立方体的各个面，我们之后可以加一些高级的光源，现在我们可以在SCNScene的实例上简单的把autoenablesDefaultLighting设置一下

    self.sceneView.autoenablesDefaultLighting = YES;
