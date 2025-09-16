---
layout: default
---

### 真机着色出现 NaN/Infinite
```glsl
A *= B / C
//改
A = A * B / C

1
//改
1.0
```

| a / 0        | 除0               |
|cross(a , b)   , a(0, 0, 0)|与模为0的向量进行叉积|
| |在俯视角为0的情况下进行投影矩阵运算|
|tan(a)  , a = 0|用0去做正切（tan）运算|
| |一个特别大的数和一个特别小的数进行浮点数运算但是给的浮点数精度不够|
|pow(-a, b)   (a > 0)| |


### Toggle关键字断合批
Toggle生成 property_ON关键字，打断srp需要在打包时清理关键字或自定义toggle

### 循环无法展开
```hlsl
//方案1，设置展开最大次数
[unroll(88)]
fixed2 center = fixed2(0.5,0.5);
fixed2 uv = i.uv - center;
for (fixed j = 0; j < _Strength; j++) {
	c1 += tex2D(_MainTex,uv*(1 - 0.01*j) + center).rgb;
}

//方案2，去掉采样lod计算
fixed4 center = fixed4(.5,.5,0,0);
fixed4 uv = fixed4(i.uv,0,0) - center;
			
for (fixed j = 0; j < _Strength; j++) {
	c1 += tex2Dlod(_MainTex,uv*(1 - 0.01*j) + center).rgb;
}
```

### shader在三星galaxy上发绿
>可能half的寄存器超量了，很可能因为过多的if分支导致

### SAMPLE_TEXTURE2D_LOD 闪退
>采样没有mip贴图避免使用


### 着色高光出现阶梯光环
>计算时需要float的精度，尤其在移动端

### time动画因为运行时间久而抽搐
>移动端小数位精度只到后五位精确，超过即可出现截断，建议 time.y % 120 做循环



***

[back](../../question-page.html)