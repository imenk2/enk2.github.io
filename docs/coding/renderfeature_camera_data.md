---
layout: default
---
在做屏幕相关效果时，不可避免地我们要对相机画面做修改，但是你去修改的相机是哪个？如果不做处理，可能会scene有效果，game没效果。

```csharp
//每个相机自己的对应对象
private class MyCameraData
{
	public Material material;
	//销毁管理
	public void Dispose()
    {
        if (material != null)
        {
            SafeDestroy(material);
            material = null;
        }
    }
}
private static Dictionary<Camera, MyCameraData> cacheData = new Dictionary<Camera, MyCameraData>();

private MyCameraData GetData(CameraData cameraData)
{
    var camera = cameraData.camera;
    if (!cacheData.TryGetValue(camera, out var data))
    {
        data = new MyCameraData();
        cacheData.Add(camera, data);
    }

    return data;
}

public override bool Execute(ref CameraData cameraData, CommandBuffer cmd, RenderTargetIdentifier source,
 RenderTargetIdentifier destination)
{
	//根据相机属性创建对应data
   var data = GetData(cameraData);
   if (!GetMaterial(data)) return false;

	//TODO
	
   cmd.Blit(source, destination, data.material);

   return true;
}

public override void Cleanup()
{
    base.Cleanup();
    Dispose();
}
protected override void OnDestroy()
{
    base.OnDestroy();
    Dispose();
}

//获取指定shader
bool GetMaterial(PerCameraData data)
  {
      if (data.material == null)
      {
          var MyShader = Shader.Find(shaderTag);
          if (MyShader == null)
          {
              Debug.Log($"Check for missing shader {shaderTag}");
              return false;
          }

          data.material = CoreUtils.CreateEngineMaterial(Shader.Find(shaderTag));
      }

      if (data.material == null)
      {
          Debug.LogWarning("Missing material. effect will not Active. Check for missing shader");
          return false;
      }

      if (!data.material.shader.isSupported) return false;
      return true;
  }
  
private static void SafeDestroy(Object obj)
{
    if (Application.isEditor)
    {
        DestroyImmediate(obj);
    }
    else
    {
        Destroy(obj);
    }
}
private void Dispose()
{
    foreach (var MyCameraData in _cacheData)
    {
        MyCameraData.Value.Dispose();
    }

    _cacheData.Clear();
}
```


[back](../../coding-page.html)