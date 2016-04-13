# 这里暂时作为记录我写程序的日志

#我的想法#：利用高通的Vuforia，写一个可以把scenekit代替OpenGL，并把UserDF和Image合并，如果还可以，就把扫码融入识别的场景中（我同事用Unity实现此功能，非常地耗Cpu，iPhone 5 ，5s，6都发热得非常严重），把整个识别的VC独立出来，app其他的界面就可以分别作为TableView，或者一些瀑布流，不用局限于此app只是用来识别。

感谢博客http://qiita.com/akira108/items/a743138fca532ee193fe 帮我完成了最难的一部分，就是用Scenekit代替OpenGL在Vuforia上面显示，目前有日文和英文版。理解得还不是很完全，大体是知道把识别的摄像头位置赋值给Scenekit场景中，摄像头的位置。

The AR in this context means reconstructing the marker's relative 3D position towards a camera from the 2D image which taken by the camera. 

<!--在头文件设置协议，让他人实现此协议中的方法-->
@protocol UserDefinedTargetsEAGLViewSceneSoure <NSObject>
- (SCNScene *)sceneForEAGLView:(UIView *)view;
@end


<!--在EAGLView 里，添加属性，-->
@property (nonatomic, strong) SCNRenderer *renderer; // renderer  ， 
@property (nonatomic, strong) SCNNode *cameraNode; // node which holds camera ， 
@property (nonatomic, assign) CFAbsoluteTime startTime; // elapsed time in 3D scene

<!--设置类属性SCNRenderer-->
<!--以OpenGL ES 的 context为渲染对象 创建新的SCNRenderer对象-->
<!--设置自动灯光-->
<!--设置自动播放属性为True-->
<!--设置类属性SCNNode 为camera，新建SCNCamera赋值给类属性SCNNode，并把类属性SCNNode添加在renderer的的rootNode上-->
- (void)setupRenderer {
            self.renderer = [SCNRenderer rendererWithContext:context options:nil];
            self.renderer.autoenablesDefaultLighting = YES;
            self.renderer.playing = YES;

    if (self.sceneSource != nil) {
        self.renderer.scene = [self.sceneSource sceneForEAGLView:self];

        SCNCamera *camera = [SCNCamera camera];
        self.cameraNode = [SCNNode node];
        self.cameraNode.camera = camera;
        [self.renderer.scene.rootNode addChildNode:self.cameraNode];
        self.renderer.pointOfView = self.cameraNode;
    }

}
<!--此方法在initWithFrame 后调用，用EAGLView创建的对象调用-->

<!--要显示SCNScene的模型在Vuforia中，就要把vuforia摄像头的矩阵转化为SCenekit摄像头的矩阵-->
<!--// Converts Vuforia matrix to SceneKit matrix-->
- (SCNMatrix4)SCNMatrix4FromQCARMatrix44:(QCAR::Matrix44F)matrix {
    GLKMatrix4 glkMatrix;

    for(int i=0; i<16; i++) {
        glkMatrix.m[i] = matrix.data[i];
    }
    return SCNMatrix4FromGLKMatrix4(glkMatrix);
}
<!--Calculate inverse matrix and assign it to cameraNode-->
- (void)setCameraMatrix:(QCAR::Matrix44F)matrix {
    SCNMatrix4 extrinsic = [self SCNMatrix4FromQCARMatrix44:matrix];
    SCNMatrix4 inverted = SCNMatrix4Invert(extrinsic); // inverse matrix!
    self.cameraNode.transform = inverted; // assign it to the camera node's transform property.
}

<!--从内场景矩阵里建立透明的投影矩阵-->
- (void)setProjectionMatrix:(QCAR::Matrix44F)matrix {
    self.cameraNode.camera.projectionTransform = [self SCNMatrix4FromQCARMatrix44:matrix];
}

<!--替换QCAR中识别到时，显示物体的方法-->
// *** QCAR will call this method periodically on a background thread ***
- (void)renderFrameQCAR
{
    [self setFramebuffer];

    // Clear colour and depth buffers
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    // Render video background and retrieve tracking state
    QCAR::State state = QCAR::Renderer::getInstance().begin();
    QCAR::Renderer::getInstance().drawVideoBackground();

    glEnable(GL_DEPTH_TEST);
    // We must detect if background reflection is active and adjust the culling direction.
    // If the reflection is active, this means the pose matrix has been reflected as well,
    // therefore standard counter clockwise face culling will result in "inside out" models.
    glEnable(GL_CULL_FACE);
    glCullFace(GL_BACK);
    if(QCAR::Renderer::getInstance().getVideoBackgroundConfig().mReflection == QCAR::VIDEO_BACKGROUND_REFLECTION_ON)
        glFrontFace(GL_CW);  //Front camera
    else
        glFrontFace(GL_CCW);   //Back camera

    // Render the RefFree UI elements depending on the current state
    refFreeFrame->render();

    [self setProjectionMatrix:vapp.projectionMatrix]; // Actually you don't have to call that every frame. Because projection matrix does not change.


    for (int i = 0; i < state.getNumTrackableResults(); ++i) {
        // Get the trackable
        const QCAR::TrackableResult* result = state.getTrackableResult(i);
        //const QCAR::Trackable& trackable = result->getTrackable();
        QCAR::Matrix44F modelViewMatrix = QCAR::Tool::convertPose2GLMatrix(result->getPose()); // Obtain model view matrix

        ApplicationUtils::translatePoseMatrix(0.0f, 0.0f, kObjectScale, &modelViewMatrix.data[0]);
        ApplicationUtils::scalePoseMatrix(kObjectScale, kObjectScale, kObjectScale, &modelViewMatrix.data[0]);

        [self setCameraMatrix:modelViewMatrix]; // SCNCameraにセット
        [self.renderer renderAtTime:CFAbsoluteTimeGetCurrent() - self.startTime]; // Render objects into OpenGL context

        ApplicationUtils::checkGlError("EAGLView renderFrameQCAR");
    }

    glDisable(GL_DEPTH_TEST);
    glDisable(GL_CULL_FACE);

    QCAR::Renderer::getInstance().end();
    [self presentFramebuffer];

}

以上以下是日志
4.11:修改结果，ImageTarget 与UserDefined 的 renderFrameQCAR方法一致，使用教程修改过后的，两个都可以使用SceneKit展现模型。

未实现功能：把UserDefined 与 ImageTarget 合并到同一个场景；
                     失去识别图时，把模型用动画过渡到中间，增加触摸手势旋转（参考以前写的项目）；
                     把扫码功能与识别场景融合（考虑是否实用）；
                     AR能用到什么地方


陈作华的GitHub
