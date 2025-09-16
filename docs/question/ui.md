---
layout: default
---


### overlay相机镜像模糊把ui糊了
>需要在renderpass里过滤Camera.main相机


### UI上RT拍摄粒子问题
>粒子混合不正确
>Blend SrcAlpha [_BlendMode], One OneMinusSrcAlpha
>粒子的 blendMode  是 One 时
>colormask 改为 RGB
>显示RT的shader blend alpha改为，One OneMinusSrcAlpha
>RT为 R16G16B16A16 sfloat，才能表现HDR颜色


### RT上角色头发变透明
二选一


1：
>第二次绘制的Alpha Blend覆盖了RT上第一次的 alpha，造成遮挡部分也有alpha
>对blendpass的alpha混合做 One One，在RT拍摄的角色上开启该设置

2：

```glsl
Blend One OneMinusSrcAlpha

half4 frag(v2f i) : SV_Target
{
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, IN.uv);
    baseMap.rgb *= baseMap.a;
    return baseMap;
}
```

### RT上直接显示模型，但需要修改显示顺序
>renderer中有order属性，修改即可


***

[back](../../question-page.html)