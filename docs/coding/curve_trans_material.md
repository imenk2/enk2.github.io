---
layout: default
---

# Unity 后处理：曲线参数传递材质球

```csharp
//将生成的图传递给材质球
 public void GenerateCurveRamp(AnimationCurve curve)
 {
      if (remapTexture == null)
      {
      	  //texture y等于几条数据
          remapTexture = new Texture2D(rampWidth, 1, TextureFormat.R8, false, true);
          remapTexture.wrapMode = TextureWrapMode.Clamp;
      }
		//rampWidth=128
      Color[] cols = new Color[rampWidth];
      for (int i = 0; i < rampWidth; i++)
      {
          cols[i].r = curve.Evaluate((float) i / rampWidth);
      }
      remapTexture.SetPixels(cols);
      remapTexture.Apply();
  }
```


```c
private void OnValidate()
{
    //该方法生命周期在后效创建时默认执行一次，当UI属性修改后再次更新 
    //避免一直setPixels
}
```

[back](../../coding-page.html)