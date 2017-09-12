---
title: ARKit-添加几何体以及物理效果
date: 2017-09-12 11:01:19
tags: AR
---
在上一篇文章中我们用ARKit检测了真实世界的水平面，然后将检测到的平面进行了可视化。在这篇文章中我们会添加一些虚拟的元素到我们的AR体验中然后开始与我们检测到的平面进行交互

读完整篇文章以后我们可以丢立方体到这个虚拟的世界了，把真实的物理效果应用到立方体上他们互相之间就有了交互，就会有轻微的碰撞并且导致立方体被碰到一边

## 射线检测

你在第一个教程的时候就了解到我们可以在任意的XYZ位置添加虚拟的3D内容，然后他会在真实的世界中被渲染和追踪。现在我们有了平面检测，所以我们就想要添加一些可以和这些平面交互的舞台。这可以让你的应用看起来像你放置了物体在你的桌子椅子或者地板上

在这个应用中，当用户点击屏幕的时候，我们做了一次射线检测，这包含了获取2D屏幕坐标然后从摄像头的位置穿过2D屏幕上的点发出一道射线然后进入场景中。如果这个射线接触到任何一个平面我们就获得了一个射线检测的结果，我们就可以拿到这个射线和平面的交点坐标放置我们的物体

代码是很简单的，ARSCNView包含有一个hitTest方法，你只需要传入一个屏幕的坐标，他就会从摄像头的起点到那个点投影一个3D的射线然后返回检测的结果:

    - (void)handleTapFrom: (UITapGestureRecognizer *)recognizer {
    // Take the screen space tap coordinates and pass them to the
    // hitTest method on the ARSCNView instance
    CGPoint tapPoint = [recognizer locationInView:self.sceneView];
    NSArray<ARHitTestResult *> *result = [self.sceneView   hitTest:tapPoint types:ARHitTestResultTypeExistingPlaneUsingExtent];
    // If the intersection ray passes through any plane geometry they
    // will be returned, with the planes ordered by distance 
    // from the camera
    if (result.count == 0) {
        return;
    }
    // If there are multiple hits, just pick the closest plane
    ARHitTestResult * hitResult = [result firstObject];
    [self insertGeometry:hitResult];
    }

得到了ARHitTestResult，我们就可以取得射线和平面的交点的世界坐标，然后往那个坐标上放置一些虚拟的物体。在这篇文章中我们只会放置一个简单的立方体，之后我们会让他看的更加真实

    - (void)insertGeometry:(ARHitTestResult *)hitResult {
    float dimension = 0.1;
    SCNBox *cube = [SCNBox boxWithWidth:dimension 
                                height:dimension 
                                length:dimension 
                        chamferRadius:0];
    SCNNode *node = [SCNNode nodeWithGeometry:cube];
    // The physicsBody tells SceneKit this geometry should be
    // manipulated by the physics engine
    node.physicsBody = [SCNPhysicsBody         
                        bodyWithType:SCNPhysicsBodyTypeDynamic 
                                shape:nil];
    node.physicsBody.mass = 2.0;
    node.physicsBody.categoryBitMask = CollisionCategoryCube;
    // We insert the geometry slightly above the point the user tapped
    // so that it drops onto the plane using the physics engine
    float insertionYOffset = 0.5;
    node.position = SCNVector3Make(
    hitResult.worldTransform.columns[3].x,
    hitResult.worldTransform.columns[3].y + insertionYOffset,
    hitResult.worldTransform.columns[3].z
    );
    // Add the cube to the scene
    [self.sceneView.scene.rootNode addChildNode:node];
    // Add the cube to an internal list for book-keeping
    [self.boxes addObject:node];
    }

## 添加物理效果

AR是用来增强现实世界的，所以为了让我们的物体变得更加真实，我们可以加一些物理效果让他有些重量的感觉

从上面的代码中我们可以看到我们给每一个立方体一个physicsBody，这表明这个几何体是受到SceneKit的物理引擎控制的。我们同样给每一个Plane一个physicsBody，所以他们两个就可以进行交互了

## 停止平面检测

一旦我们检测完了我们的环境然后得到了一定数量的平面，我们就不需要ARKit持续的给我们新的平面并且很可能更新现有的平面，这样会影响到我们已经添加到场景当中的物体

在这个应用中如果用户按下两个手指并维持超过一秒，我们会隐藏所有的平面然后关闭平面检测。要做到这个只需要更新ARSession设置的planeDetection属性，然后重新跑这个会话。默认的这个会话会保持同样的坐标系以及我们一经发现的所有anchor:

    // Get our existing session configuration
    ARWorldTrackingSessionConfiguration *configuration = (ARWorldTrackingSessionConfiguration *)self.sceneView.session.configuration;
    // Turn off future plane detection and updating
    configuration.planeDetection = ARPlaneDetectionNone;
    // Re-run the session for the changes to take effect
    [self.sceneView.session runWithConfiguration:configuration];