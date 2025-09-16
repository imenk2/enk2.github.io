---
layout: default

title: water line
date: 2025-09-16
last_modified_at: 2025-09-16
---

# 吃水线
***

![Branching](../../assets/img/water_line/show1.gif)

游戏中的吃水线指的是，相机在水面与水下交界处所观察到的由水体切割而来的线。

吃水线有很多中实现方式，大致分类为以下几种：

有在屏幕绘制一个网格，并使用与水体一致的波形同步屏幕与相交水体形状，使这个带有波浪的mesh绘制mask的方式（水体与屏幕mesh都做顶点运动）。

还有在屏幕上使用与水体高度同步的屏幕uv，计算一个虚拟的波形mask的方式（水体本身没有顶点运动）

最后就是本文中介绍的，使用水体背面来绘制屏幕mask的实现方式。

这种方式的好处是不需要同步模型的波形，因为本身绘制是一个东西，缺点便是两次draw并且顶点计算（着色与mask），在模型水域不是很广的情况下，这会是一个比较节省的方式



在准备阶段，首先需要一个水面模型，为了点运动时足够平滑它的顶点不能太少，一个由任意波形着色的材质。当拥有这些后，开始书写renderfeature。

首先使用脚本将水面的高度传递给renderfeature，使用cmd.DrawProcedural绘制一个全屏的水面高度mask（下图gif的起始0帧黑白区域，后续是水体mask）

```c#
float zBufferParamsX, zBufferParamsY;
if (SystemInfo.usesReversedZBuffer)
{
    zBufferParamsY = 1f;
    zBufferParamsX = data.camera.farClipPlane / data.camera.nearClipPlane - 1f;
}
else
{

    zBufferParamsY = data.camera.farClipPlane / data.camera.nearClipPlane;
    zBufferParamsX = 1f - zBufferParamsY;
}

var farPlaneLerp = (1f - zBufferParamsY * m_Settings.m_FarPlaneMultiplier) / (zBufferParamsX * m_Settings.m_FarPlaneMultiplier);
data.material.SetFloat(FarPlaneOffset, farPlaneLerp);

//上面的代码计算的是当前水体在屏幕上实际远裁面的深度
//可以使用_ZBufferParams来实现
```
在shader 计算vert时，将mesh偏移到这个深度下

```glsl
float4 GetFullScreenTriangleVertexPosition(uint vertexID, float z = UNITY_NEAR_CLIP_VALUE)
{
    float2 uv = float2((vertexID << 1) & 2, vertexID & 2);
    return float4(uv * 2.0 - 1.0, z, 1.0);
}
o.vertex = GetFullScreenTriangleVertexPosition(v.vertexID);//正常屏幕三角形模型
```
移动到原裁剪范围的屏幕模型深度，这个时候它的世界坐标计算就反映了水体可见范围的真实世界高度

```hlsl
float4 ComputeClipSpacePosition(float2 positionNDC, float deviceDepth)
{
 float4 positionCS = float4(positionNDC * 2.0 - 1.0, deviceDepth, 1.0);
 return positionCS;
}
float3 ComputeWorldSpacePosition(float2 positionNDC, float deviceDepth, float4x4 invViewProjMatrix)
{
 float4 positionCS  = ComputeClipSpacePosition(positionNDC, deviceDepth);
 float4 hpositionWS = mul(invViewProjMatrix, positionCS);
 return hpositionWS.xyz / hpositionWS.w;
}

//位移到远裁深度世界位置，比较与水体的世界高度
float3 positionWS = ComputeWorldSpacePosition(i.uv, _FarPlaneOffset, _MATRIX_I_VP);
half heightY = positionWS.y > _WaterCenterPosWorld.y ? 1 : -1;
```
然后得到一个基础的水体与天空的分界遮罩，后续只需要renderlist的方式，把水的背面填充到这个遮罩即可（水实际绘制两次，一次正向着色，一次underMask）

![Branching](../../assets/img/water_line/show2.gif)


最后，使用绘制而得到的underMask，做屏幕后处理，用来处理水下效果。因为水体的额外渲染，可以一起拿一下水下深度，方便做水下雾效等其他效果。


最后是完整的测试代码（unity6）
```c#
using System;
using UnityEngine;
using UnityEngine.Experimental.Rendering;
using UnityEngine.Rendering.RenderGraphModule;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class UnderWaterRenderFeature : ScriptableRendererFeature
{
    private UnderWaterRenderPass m_UnderWaterRenderPass;
    [Serializable]
    public class Settings
    {
        public enum DonwnSample
        {
            _0x = 0,
            _1x = 1,
            _2x = 2,
            _4x = 4,
        }

        public RenderPassEvent m_RenderPassEvent = RenderPassEvent.AfterRenderingTransparents;
        public DonwnSample m_Size = DonwnSample._2x;
        public Material m_Water;
        [Range(0.001f, 1.0f)] public float m_FarPlaneMultiplier = 0.0015f;
        
        public const string m_UnderWaterMaskName = "_UnderWaterMask";
        public const string m_UnderWaterDepthName = "_UnderWaterDepth";
        public const string m_WaterDepthName = "_WaterDepth";
        public RTHandle m_UnderWaterMask;
        public RTHandle m_UnderWaterDepth;
        public RTHandle m_WaterDepth;

        public bool Init()
        {
            if (m_Water == null)
            {
                Shader mShader = Shader.Find("Unlit/water_test");
                if(mShader)
                {
                    m_Water = CoreUtils.CreateEngineMaterial(mShader);
                }
                else
                {
                    return false;
                }
            }

            return true;
        }
        public void Dispose()
        {
            if(m_UnderWaterMask != null) m_UnderWaterMask.Release();
            if(m_UnderWaterDepth != null) m_UnderWaterDepth.Release();
        }
    }

    public Settings m_Settings;
    
    public override void Create()
    {
        name = "Under Water Mask";
        if (m_Settings == null) m_Settings = new Settings();
        if (m_Settings.m_Water == null) m_Settings.Init();
        if(m_UnderWaterRenderPass == null) m_UnderWaterRenderPass = new UnderWaterRenderPass();
    }

    public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
    {
        if (renderingData.cameraData.cameraType != CameraType.Game) return;
        
        var descriptor = renderingData.cameraData.cameraTargetDescriptor;
        descriptor.width = descriptor.width >> (int)m_Settings.m_Size;
        descriptor.height = descriptor.height >> (int)m_Settings.m_Size;
        descriptor.depthBufferBits = 0;
        descriptor.msaaSamples = 1;
        GetRenderTextureHandle(ref m_Settings.m_UnderWaterMask, descriptor, Settings.m_UnderWaterMaskName);
        GetRenderTextureHandle(ref m_Settings.m_UnderWaterDepth, descriptor, Settings.m_UnderWaterDepthName, true);
        GetRenderTextureHandle(ref m_Settings.m_WaterDepth, descriptor, Settings.m_WaterDepthName, true);
       
        m_UnderWaterRenderPass.Setup(m_Settings);
        renderer.EnqueuePass(m_UnderWaterRenderPass);
    }

    public void GetRenderTextureHandle(ref RTHandle handle, RenderTextureDescriptor descriptor, string sName, bool isDepth = false)
    {
            if (isDepth)
            {
                descriptor.graphicsFormat = GraphicsFormat.D16_UNorm;
                descriptor.depthBufferBits = 16;
                descriptor.depthStencilFormat = GraphicsFormat.None;
                RenderingUtils.ReAllocateHandleIfNeeded(ref handle, descriptor, FilterMode.Point,
                    TextureWrapMode.Clamp, name: sName);
            }
            else
            {
                descriptor.graphicsFormat = GraphicsFormat.R16_SFloat;
                descriptor.depthBufferBits = 16;
                descriptor.depthStencilFormat = GraphicsFormat.None;
                RenderingUtils.ReAllocateHandleIfNeeded(ref handle, descriptor, FilterMode.Bilinear,
                    TextureWrapMode.Clamp, name: sName);
            }
    }
    protected override void Dispose(bool disposing)
    {
        m_UnderWaterRenderPass?.Dispose();
        m_UnderWaterRenderPass = null;
        m_Settings?.Dispose();
        m_Settings = null;
    }
}

public class UnderWaterRenderPass : ScriptableRenderPass
{
    private static UnderWaterRenderFeature.Settings m_Settings;
    private static readonly int FarPlaneOffset = Shader.PropertyToID("_FarPlaneOffset");
    private static readonly int MatrixIVp = Shader.PropertyToID("_MATRIX_I_VP");
    private static readonly int ScreenTexture = Shader.PropertyToID("_ScreenTexture");
    private static readonly int BlitTexture = Shader.PropertyToID("_BlitTexture");
    private static readonly int CopyColorTexture = Shader.PropertyToID("_CopyColorTexture");
    private static readonly int ScaleBias = Shader.PropertyToID("_ScaleBias");

    public UnderWaterRenderPass()
    {
        
    }
    class PassData
    {
        public Camera camera;
        public TextureHandle mask;
        public TextureHandle depth;
        public Material material;
        public string mName;
        public string dName;
        public string wdName;
        public RendererListHandle rendererList;
        public Mesh triangleMesh;
        public TextureHandle source;
        //public TextureHandle copyColor;
        public TextureHandle waterDepth;
    }
    
    public void Setup(UnderWaterRenderFeature.Settings mSettings)
    {
        renderPassEvent = mSettings.m_RenderPassEvent;
        m_Settings = mSettings;
    }
    
    public override void RecordRenderGraph(RenderGraph renderGraph, ContextContainer frameData)
    {
        const string passName = "Under Water Mask";

        var resourceData = frameData.Get<UniversalResourceData>();
        var cameraData = frameData.Get<UniversalCameraData>();
        var renderingData = frameData.Get<UniversalRenderingData>();
        var lightData = frameData.Get<UniversalLightData>();

        //也可以使用RenderGraph的函数，直接在graph代码内创建TextureHandle
        TextureHandle underWaterMask = renderGraph.ImportTexture(m_Settings.m_UnderWaterMask);
        TextureHandle underWaterDepth = renderGraph.ImportTexture(m_Settings.m_UnderWaterDepth);
        TextureHandle waterDepth = renderGraph.ImportTexture(m_Settings.m_WaterDepth);

        //添加一个不安全的的渲染pass，因为我们要手动切换target
        using (var builder = renderGraph.AddUnsafePass<PassData>(passName, out var passData))
        {
            TextureHandle source = resourceData.activeColorTexture;
            //TextureHandle sourceDepth = resourceData.cameraDepthTexture;
            //var desc = renderGraph.GetTextureDesc(source);
            //TextureHandle copyColor = renderGraph.CreateTexture(desc);
            if (!underWaterMask.IsValid() || !underWaterDepth.IsValid() || !source.IsValid()) return;
            passData.material = m_Settings.m_Water;
            passData.mask = underWaterMask;
            passData.depth = underWaterDepth;
            passData.mName = UnderWaterRenderFeature.Settings.m_UnderWaterMaskName;
            passData.dName = UnderWaterRenderFeature.Settings.m_UnderWaterDepthName;
            passData.camera = cameraData.camera;
            passData.source = source;
            //passData.copyColor = copyColor;
            passData.waterDepth = waterDepth;
            
            ShaderTagId passID = new ShaderTagId("UnderWaterVFace");
            SortingCriteria sortingCriteria = SortingCriteria.CommonTransparent;
            DrawingSettings drawSettings = RenderingUtils.CreateDrawingSettings(passID, renderingData, cameraData, lightData, sortingCriteria);
            FilteringSettings filteringSettings = new FilteringSettings(RenderQueueRange.transparent, -1);
            RendererListParams param = new RendererListParams(renderingData.cullResults, drawSettings, filteringSettings);
            param.filteringSettings.batchLayerMask = uint.MaxValue;
            passData.rendererList = renderGraph.CreateRendererList(param);
            
            builder.UseTexture(underWaterMask, AccessFlags.ReadWrite);
            builder.UseTexture(underWaterDepth, AccessFlags.ReadWrite);
            builder.UseTexture(resourceData.activeColorTexture);
            //builder.UseTexture(passData.copyColor);
            builder.UseTexture(passData.waterDepth);
            //builder.AllowGlobalStateModification(true);
            builder.UseRendererList(passData.rendererList);
            
            builder.SetRenderFunc((PassData data, UnsafeGraphContext context) => ExecutePass(data, context));
        }
    }

    static void ExecutePass(PassData data, UnsafeGraphContext context)
    {
        if (data.material == null) return;
        RTHandle underWatermask = data.mask;
        RTHandle underWaterdepth = data.depth;
        Vector2 viewportScale = underWatermask.useScaling
            ? new Vector2(underWatermask.rtHandleProperties.rtHandleScale.x, underWatermask.rtHandleProperties.rtHandleScale.y)
            : Vector2.one;
        
        //CommandBuffer unsafeCmd = CommandBufferHelpers.GetNativeCommandBuffer(context.cmd);
        var cmd = CommandBufferHelpers.GetNativeCommandBuffer(context.cmd);

        cmd.SetGlobalTexture(data.mName, data.mask);
        cmd.SetGlobalTexture(data.dName, data.depth);
        //context.cmd.SetRenderTarget(underWaterdepth);
        //Blitter.BlitTexture(cmd, underWaterdepth, viewportScale, data.material, 1);
        
        context.cmd.SetRenderTarget(underWatermask, underWaterdepth);
        context.cmd.ClearRenderTarget(true, true, Color.black);
        float zBufferParamsX, zBufferParamsY;
        if (SystemInfo.usesReversedZBuffer)
        {
            zBufferParamsY = 1f;
            zBufferParamsX = data.camera.farClipPlane / data.camera.nearClipPlane - 1f;
        }
        else
        {
            zBufferParamsY = data.camera.farClipPlane / data.camera.nearClipPlane;
            zBufferParamsX = 1f - zBufferParamsY;
        }

        var farPlaneLerp = (1f - zBufferParamsY * m_Settings.m_FarPlaneMultiplier) / (zBufferParamsX * m_Settings.m_FarPlaneMultiplier);
        data.material.SetFloat(FarPlaneOffset, farPlaneLerp);
        Matrix4x4 vp = data.camera.projectionMatrix * data.camera.worldToCameraMatrix;
        data.material.SetMatrix(MatrixIVp, vp.inverse);
        context.cmd.DrawProcedural(Matrix4x4.identity, data.material, 1, MeshTopology.Triangles, 3);
       
        //context.cmd.SetRenderTarget(underWatermask, data.sourceDepth);//分辨率不一致
        
        context.cmd.DrawRendererList(data.rendererList);
        
        //cmd.CopyTexture(data.source, data.copyColor);
        context.cmd.SetRenderTarget(data.source);
        MaterialPropertyBlock materialPropertyBlock = new MaterialPropertyBlock();
        materialPropertyBlock.SetTexture(BlitTexture, underWatermask);
        //materialPropertyBlock.SetTexture(CopyColorTexture, data.copyColor);
        materialPropertyBlock.SetVector(ScaleBias, new Vector4(1, 1, 0, 0));
        context.cmd.DrawProcedural(Matrix4x4.identity, data.material, 3, MeshTopology.Triangles, 3, 1, materialPropertyBlock);
        
    }

    public override void FrameCleanup(CommandBuffer cmd)
    {
        
    }

    public void Dispose()
    {
        
    }

}

```

shader

```hlsl
Shader "Unlit/water_test"
{
    Properties
    {
        _Color("Color", color) = (1,1,1,1)
        _BlitTexture ("Texture", 2D) = "white" {}
        _WaveScale("Wave Scale", range(0, 1)) = 1
    }
    
    CGINCLUDE
    #include "UnityCG.cginc"
    struct appdata
    {
        float4 vertex : POSITION;
        float2 uv : TEXCOORD0;
        float3 normal : NORMAL;
        uint vertexID : SV_VertexID;
    };

    struct v2f
    {
        float4 vertex : SV_POSITION;
        float2 uv : TEXCOORD0;
        float3 worldNormal : TEXCOORD1;
        float3 worldPos : TEXCOORD2;
    };

    float4x4 _MATRIX_I_VP;
    float _FarPlaneOffset;
    float _WaveScale;
    float4 _WaterCenterPosWorld;

    float4 GetWave(float4 vertex)
    {
        vertex.y += sin(_Time.y*6 + (vertex.z + vertex.x)*20)*0.2 * _WaveScale;
        return vertex;
    }
    float4 GetFullScreenTriangleVertexPosition(uint vertexID, float z = UNITY_NEAR_CLIP_VALUE)
    {
        float2 uv = float2((vertexID << 1) & 2, vertexID & 2);
        return float4(uv * 2.0 - 1.0, z, 1.0);
    }
    float4 ComputeClipSpacePosition(float2 positionNDC, float deviceDepth)
    {
        float4 positionCS = float4(positionNDC * 2.0 - 1.0, deviceDepth, 1.0);
        // positionCS.y was flipped here but that is SRP specific to solve flip baked into matrix.
        return positionCS;
    }
    float3 ComputeWorldSpacePosition(float2 positionNDC, float deviceDepth, float4x4 invViewProjMatrix)
    {
        float4 positionCS  = ComputeClipSpacePosition(positionNDC, deviceDepth);
        float4 hpositionWS = mul(invViewProjMatrix, positionCS);
        return hpositionWS.xyz / hpositionWS.w;
    }
    float2 GetFullScreenTriangleTexCoord(uint vertexID)
    {
    #if UNITY_UV_STARTS_AT_TOP
        return float2((vertexID << 1) & 2, 1.0 - (vertexID & 2));
    #else
        return float2((vertexID << 1) & 2, vertexID & 2);
    #endif
    }
    ENDCG
    
    SubShader
    {
        Tags {"Queue" = "Transparent" "RenderType"="Transparent" }
        LOD 100

        
        Pass
        {
            //water 0
            Tags{"RenderQueue" = "Transparent"}
            
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
        
            sampler2D _BlitTexture;
            float4 _MainTex_ST;
            half4 _Color;

            v2f vert (appdata v)
            {
                v2f o;
                 v.vertex = GetWave(v.vertex);
                float3 worldPos = mul(UNITY_MATRIX_M, v.vertex).xyz;

                o.worldPos = worldPos;
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
             
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_BlitTexture, i.uv) * _Color;
                return col;
            }
            ENDCG
        }
        
         Pass
        {
            //Horizontal mask 1
            Tags { "LightMode"="UnderWaterHorizontal"}
            
            ZWrite off
            ZTest Always
            Cull off
            
            
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            sampler2D _BlitTexture;

            float4 _MainTex_ST;
            half4 _Color;
            float _MaskBelowSurface;


            
            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = GetFullScreenTriangleVertexPosition(v.vertexID);
                o.uv = GetFullScreenTriangleTexCoord(v.vertexID);
             
                return o;
            }

            half4 frag (v2f i) : SV_Target
            {
                float3 positionWS = ComputeWorldSpacePosition(i.uv, _FarPlaneOffset, _MATRIX_I_VP);
                return positionWS.y > _WaterCenterPosWorld.y ? 1 : -1;
            }
            ENDCG
        }
        
         Pass
        {
            //vface mask 2
            Tags { "LightMode"="UnderWaterVFace"}
            
            Cull off
            ZTest LEqual
            
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            sampler2D _BlitTexture;
            sampler2D _CameraDepthTexture;
            float4 _MainTex_ST;
            half4 _Color;

            v2f vert (appdata v)
            {
                v2f o;
                v.vertex = GetWave(v.vertex);
                float3 worldPos = mul(UNITY_MATRIX_M, v.vertex).xyz;

                //float floor = tex2Dlod(_CameraDepthTexture, float4(v.uv, 0, 0)).r;
                //worldPos.y += floor;
                o.vertex = UnityWorldToClipPos(worldPos);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
             
                return o;
            }

            struct FragemtnOutput
            {
                half4 color : SV_Target0;
                float depth : SV_Depth;
            };
            
            fixed4 frag (v2f i, bool vface : SV_IsFrontFace) : SV_Target
            {
                FragemtnOutput output = (FragemtnOutput)0;
                
                if (LinearEyeDepth(i.vertex.z) < (_ProjectionParams.y + 0.001))
                {
                   discard;
                }

                if (!vface)
                {
                   return -1;
                }
                else
                {
                   return 1;
                }
            }
            ENDCG
        }
        Pass
        {
            //draw screen mesh 3
            Tags {"RenderQueue" = "Transparent"}
            Blend SrcAlpha OneMinusSrcAlpha
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
           
            sampler2D _BlitTexture;
            sampler2D _ScreenTexture;
            sampler2D _CameraDepthTexture;
            sampler2D _CopyColorTexture;

            float4 _MainTex_ST;
            float4 _ScaleBias;
            half4 _Color;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = GetFullScreenTriangleVertexPosition(v.vertexID);
                o.uv = GetFullScreenTriangleTexCoord(v.vertexID);
             
                return o;
            }
 
            half4 frag (v2f i) : SV_Target
            {
                // sample the texture
                float depth = tex2D(_CameraDepthTexture, i.uv);
                float4 scene = tex2D(_CopyColorTexture, i.uv);
                float3 positionWS = ComputeWorldSpacePosition(i.uv, _FarPlaneOffset, _MATRIX_I_VP);
                half4 col = tex2D(_BlitTexture, i.uv) * _Color;
                //col.rgb = lerp(scene.rgb, half3(0,0,1), 1-col.r);
                col.a = 1 - (col.r);//可能为-1，所以需要映射或截断
                col.rgb = half3(0, 0, 1);
                return col;
            }
            ENDCG
        }
    }
}
```


[back](../../coding-page.html)


