---
layout: default
---

### 角色平滑法线如果要保存，建议存入uv中
>角色蒙皮动画会修改顶点法线结果，如果存入uv中则不受影响，且需要存储为切空间，因为fbx文件只能保存vector2的问题，可以压缩z到y，或使用多一套uv记录


```c#
//pack
Vector4 normalTS = smoothNormals[i].normalized;
float packedNormalY = (normalTS.y * 0.5f + 0.5f) * Mathf.Sign(normalTS.z);
Vector2 PackedUV2 = new Vector2(normalTS.x, packedNormalY);
listVtxUV.Add(PackedUV2);
```


```c#
//unpack
half zSign = sign(packedUV.y);
half unpackedY = (abs(packedUV.y) - 0.5) * 2;
half unpackedZ = sqrt(1 - packedUV.x * packedUV.x - unpackedY * unpackedY) * zSign;
half3 normalTS = half3(packedUV.x, unpackedY, unpackedZ);
half3 binormal = cross(normalize(packedAttributes.normalOS), normalize(packedAttributes.tangentOS.xyz)) * packedAttributes.tangentOS.w;
half3x3 matTangentToObject = half3x3(packedAttributes.tangentOS.xyz, binormal.xyz, packedAttributes.normalOS);
half3 smoothedNormalOS = mul(normalize(normalTS), matTangentToObject);

```

### 蒙皮模型脚本计算的向量传入不正确
>蒙皮模型用的是绑骨的变换矩阵，由子到父的变换，需要脚本传递挂载mesh节点的变换矩阵变换模型顶点才会和传入向量同空间


### 捏脸骨骼动画骨骼控制关系
>动画骨骼驱动捏脸骨骼，动画做父捏脸做子

### fbx模型uv数据
>即便在3d软件中生成并计算了vector4 类型uv数据，最终保存到fbx中的仍然为vector2



***
[back](../../question-page.html)