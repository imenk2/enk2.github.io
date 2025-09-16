---
layout: default
---


### 打包不能使用Shader.Find
>shader被打到包里以后，Shader.Find会报空

### 遮挡显示-不全
>角色需要在不透明队列靠后绘制，不然深度检测不通过stencil不能正确更新


### rt相机拍摄头像与场景主光冲突
>UI界面的rt相机与场景相机culling mask不能有交集，renderer asset里层级剔除
>lingting界面内指定sun将会忽略剔除，拿到sun方向和强度，即便隐藏

### 利用depthprepass的物体选中后才显示
>layer检查是否和predepth一个equal

### 苹果ios真机 rt有马赛克
>检查这张rt filter是否 point，采样器可能自动设置失败

### 模型与场景深度比较
>linear01depth(posCS.z, _ZBufferParams)固定将z映射到 0-1，所以posCS.z <= sceneZ
>如果没有效果，检查shader是否写入深度，renderfeature depthBufferBits = 24

### 场景光照出现马赛克
>检查法线图压缩(BC7)，烘焙方向图压缩(BC7)，天空球贴图设置(三线性)

### 深度相关绘制出现切片
>检查绘制RT是否使8位，造成精度截断

### 粒子decal不显示
>因为粒子预烘焙世界空间模型数据，所以它的world2object 和 object2world都是单位矩阵）可以尝试GUI粒子实例化解决 [文档](https://forum.unity.com/threads/can-decal-shader-use-particle-mesh.1132477/)

### cmd.SetRenderTarget()不成功
>cmd.Clear();
cmd.SetRenderTarget(targetRT);
context.ExecuteCommandBuffer(cmd);

>立即执行cmd命令才会将targetRT设置上，否则后续的绘制时targetRT仍然处于等待提交的阶段，导致实际绘制目标错误。

### 画面无故Z-Fighting
>检查是否有cmd.SetViewProjectionMatrices操作，是否没有兼容TAA

```c#
//检查是否有cmd.SetViewProjectionMatrices操作，是否没有兼容TAA
var proj = renderingData.cameraData.camera.projectionMatrix;
if (renderingData.taaEnable && renderingData.taaInternalActive && renderingData.cameraData.cameraType == CameraType.Game && renderingData.cameraData.camera == Camera.main)
{
    proj = renderingData.taaProjOverride;
}

cmd.SetViewProjectionMatrices(renderingData.cameraData.camera.worldToCameraMatrix, proj);
```

### 小米10录屏花帧
>带有alpha的屏幕blend操作导致，为1可解决，但是认为是mi10的自身bug

### 阴影锯齿
>SAMPLE_TEXTURE2D (tex.Sample)采样需要是Bilinear，
SAMPLE_TEXTURE_DEPTH(tex.SampleCmpLevelZero)是4像素平均值，软阴影shadow应该使用后者，并且做PCF时，不做二值化以免重建阶梯。

### unity 内置粒子系统不能使用srp特性
>需要合批的话，只能规范order和材质队列，简单做法是队列相同3000，对order排序，同材质同order


***
[back](../../question-page.html)