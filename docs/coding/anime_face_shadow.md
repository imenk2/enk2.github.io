---
layout: default

title: 卡通渲染脸部阴影调研
date: 2025-09-25
last_modified_at: 2025-09-25
---

# 卡通渲染脸部阴影调研

```
这里没有传统的sdf脸部阴影形式介绍
```

## 360
## 平滑法线+mask
![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-23 20-40-03.gif)

[链接](https://x.com/rukikuri/status/1685593563911061504)

基于平滑法线的360°脸部阴影，脸上有固定阴影mask

### 基于多向sdf的脸部阴影形式

![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-23 10-58-39.gif)

[链接](https://zhuanlan.zhihu.com/p/670837192)


使用65张朝向的sdf 序列，根据灯光角度采样4张融合结果

### 伪360 模型法线

| ![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-23 20-12-22.gif) | ![Branching](../../assets/img/anime_face_shadow/GIF 2025-8-13 15-41-47.gif) |

蓝色协议中，角色脸部对必要区域进行法线方向反转，但是因为照顾左右两侧光照，纵向光照不理想
>示例中角色头部3335三角面

## Horizontal 180

### 区域性SDF

| ![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-22 19-40-20.gif) | ![Branching](../../assets/img/anime_face_shadow/face2.png) |

赛马娘中使用伦勃朗三角光区域性SDF进行计算，生成基础光照后根据贴图修改光照结果

### 区域翻转法线

![Branching](../../assets/img/anime_face_shadow/face1.png)

[链接](https://zhuanlan.zhihu.com/p/1908718263602489063)

学院偶像大师则类似,区别是直接使用三角区域对平滑法线翻转得到光照结果

# 测试

| ![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-25 12-01-27.gif) | ![Branching](../../assets/img/anime_face_shadow/GIF 2025-9-25 14-24-32.gif) |

使用学园偶像大师的方式结合赛马娘，区分三角遮罩SDF，进行光照着色,期间使用sdf区域提取的遮罩范围按照SDF过度翻转法线
>这里的贴图参考赛马娘，使用脸颊[128,255]鼻翼[128,0]的方式


>从图中截取这两部分的各自遮罩范围与映射后的[0,1]范围值

```shader
half SDFDir = (sdfTex * 2.0 - 1.0); // [-1, 1]
half SDF = -SDFDir;// 用于翻转法线X方向
half cheekMask = saturate(SDFDir);// 截取脸颊Mask
half noseMask = saturate((0.5 - sdfValue) * 2.0);// 截取鼻翼Mask
```

***

[back](../../coding-page.html)