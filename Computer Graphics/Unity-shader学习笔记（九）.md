[TOC]



# Unity-shader学习笔记（九）

## 20 非真实感渲染

### 20.1卡通风格的渲染

实现卡通渲染的方法很多，其中之一是使用基于色调的着色技术。在实现中，会使用漫反射系数对一张一维纹理进行采样，以控制漫反射的色调。此外卡通风格通常还需要在物体边缘部分绘制轮廓，这不同于之前的屏幕后处理技术的对屏幕图像的描边，这是基于模型的描边方法，这种方法的实现更加的简单。

#### 20.1.1渲染轮廓线

一共有五种类型用于绘制模型轮廓线：

##### ①基于观察角度和表面法线的轮廓线绘制。

这种方法是使用视角方向和表面法线的点乘结果来得到轮廓线的信息。这种方法简单快速，可以在一个Pass中就得到渲染结果，但局限性很大，而且有些情况下模型渲染出来的描边效果不尽如人意。

##### ②过程式几何轮廓线渲染。

这种方法是使用两个Pass渲染。第一个Pass渲染背面的面片，并使用某些技术让它的轮廓可见；第二个Pass再正常渲染正面的面片。这种方法的优点在快速有效，并且适用于绝大多数表面平滑的模型，但缺点也很明显，不适合立方体这种平整的模型。

##### ③基于图像处理的轮廓线渲染。

前面所讲的边缘检测就是基于这种方法。优点在于可以适用于任何种类的模型。但局限性也很大，一些深度和法线变化很小的轮廓无法被检测出来，例如桌面上的纸张。

##### ④基于轮廓边检测的轮廓线渲染

前三种渲染的最大问题是无法控制轮廓线的风格渲染。在一些需求下我们会被要求渲染出独特风格的轮廓线，例如水墨风格等。为此我们需要精确检测出轮廓线，然后对它们渲染。检测一条边是否是轮廓边的方法是：检测和这条边相邻的两个三角面片是否满足这个公式：
$$
(n_{0}\cdot v > 0) ≠(n_{1}\cdot v > 0)
$$
其中n0和n1分别表示两个相邻三家面片的法向，v是从视角到该边任意顶点的方向。这个公式的本质是在于检测两个相邻的三角片面是否一个朝正面、一个朝背面。这个检测可以在几何着色器（Geometry Shader）下完成。当然这种方法是有缺点的：实现上相对复杂，以及会出现动画连贯性问题，也就是由于是逐帧单独提取轮廓，就会在帧与帧之间出现跳跃性。

##### ⑤第五种其实是前面四种的混合。

首先找到精确的轮廓线，把模型和轮廓边渲染到纹理中，再使用图像处理的方式识别出轮廓线，并在图像空间下进行风格化渲染。

先使用过程式几何轮廓线渲染的方法来对模型进行轮廓描边：使用两个Pass渲染模型——在第一个Pass中使用轮廓线颜色渲染整个背面的面片，并在视角空间下把模型顶点沿着法线方向向外扩张一段距离，以此来让背部轮廓线可见：

```c#
viewPos = viewPos + viewNormal * _Outline;
```

但是不能直接使用顶点法线进行扩展，因为对一些内凹的模型，就可能发生背面面片遮挡正面面片的情况。因此在扩张背面顶点之前，我们首先对顶点法线的z分量进行处理，使它们等于一个定值，然后把法线归一化后再对顶点进行扩张，这样做的好处在于扩展后的背面更加扁平化，从而降低了遮挡正面面片的可能性。

```c#
viewNoemal.z = -0.5;
viewNormal = normalize(viewNormal);
viewPos = viewPos + viewNormal * _Outline;
```

#### 20.1.2 添加高光

为了在模型上实现卡通风格的一块块分界明显的纯色区域，我们就要实现Blinn-Phong模型并进行相应的改进，我们原本计算高光反射是：

```c#
float spec = pow(max(0, dot(normal, halfDir)), _Gloss);
```

在卡通渲染中我们需要将normal与halfDir的点乘结果与一个阈值进行比较，如果小于该阈值，则高光反射系数为0，否则返回1。

```c#
float spec = dot(worldNormal, worldHalfDir);
spec = step(threshold, spec);
```

我们使用step函数来实现和阈值比较的目的。step函数接受两个参数，一个参数是参考值，第二个参数是待比较的数值。如果第二个参数大于等于第一个参数，则返回1，否则返回0.

但这种粗暴的判断方法会在高光区域的边界造成锯齿。这是因为高光区域的边缘不是平滑渐变，而是由0突变到1。所以我们就需要在边界很小的一块区域内，进行平滑处理。

```c#
float spec = dot(worldNormal, worldHalfDir);
spec = lerp(0, 1, smoothstep(-w, w, spec - threshold));
```

smoothstep函数中w参数的意义是：当spec-threshold小于-w时，返回0，大于w时，返回1，否则在0~1之间进行插值。这样就可以在高光区域的边界处（[-w,w]区间内）得到一个从0到1的平滑处理的spec值，从而实现抗锯齿的目的。w值可以由你自己定，但最好的是通过fwidth函数来得到邻域像素之间的近似导数值。

#### 20.1.3 实现

①声明需要使用的各个属性

```c#
Properties{
    _Color ("Color Tint", Color) = (1,1,1,1)
    _MainText ("Main Tex", 2D) = "White" {}
    _Ramp ("Ramp Texture", 2D) = "White" {}
    _Outline ("Outline", Range(0, 1)) = 0.1
    _OutlineColor ("Outline Color", Color) = (0,0,0,1)
    _Specular ("Specular", Color) = (1,1,1,1)
    _SpecularScale ("Specular Scale", Range(0, 0.1)) = 0.01
}
```

_Ramp用于控制漫反射色调的渐变纹理； _Outline用于控制轮廓线宽度； _OutlineColor对应着轮廓线颜色； _Specular是高光反射颜色； _SpecularColor用于控制计算高光反射时使用的阈值。

②定义渲染轮廓线需要的Pass，这个Pass只渲染背面的三角面片

先设置正确的渲染状态

```c#
Pass{
    NAME "OUTLINE"
    Cull Front
}
```

使用Cull指令先剔除正面的三角面片，而只渲染背面。

③定义描边需要的顶点着色器和片元着色器

```c#
v2f vert (a2v v) 
{
	v2f o;
			
	float4 pos = mul(UNITY_MATRIX_MV, v.vertex); 
	float3 normal = mul((float3x3)UNITY_MATRIX_IT_MV, v.normal);  
	normal.z = -0.5;
	pos = pos + float4(normalize(normal), 0) * _Outline;
	o.pos = mul(UNITY_MATRIX_P, pos);
				
	return o;
}
			
float4 frag(v2f i) : SV_Target 
{ 
	return float4(_OutlineColor.rgb, 1);               
}
```

在顶点着色器中我们首先把顶点和法线变换到视角空间下，以让描边可以在观察空间达到最好的效果。然后设置法线的z分量，对其归一化后再将顶点沿其方向进行扩张，得到扩张后的顶点坐标。对法线的处理是为了尽可能避免背面扩张后的顶点挡住正面的面片。最后再将顶点从视角空间变换到裁剪空间。

片元着色器就只干一件事——用轮廓线颜色渲染整个背面

④接下来就是光照模型所在的Pass，以渲染模型的正面。由于光照模型需要使用Unity提供光照等信息，我们需要为Pass进行相应的设置，并添加相应的编译指令。

```c#
Pass{
    Tags {"LightMode" = "ForwardBase"}
    
    Cull Back
        
    CGPROGRAM
        
    #pragma vertex vert
    #pragma fragment frag
        
    #pragma multi_compile_fwdbase
    ......;
}
```

⑤定义顶点着色器

```c#
struct v2f{
    float4 pos : POSITION;
    float2 uv : TEXCOORD0;
    float3 worldNormal : TEXCOORD1;
    float3 worldPos : TEXCOORD2;
    SHADOW_COORDS(3);
};

v2f vert(a2v v){
    v2f o;
    
    o.pos = UnityObjectToClipPos( v.vertex);
	o.uv = TRANSFORM_TEX (v.texcoord, _MainTex);
	o.worldNormal  = UnityObjectToWorldNormal(v.normal);
	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
	TRANSFER_SHADOW(o);
				
	return o;
}
```

⑥片元着色器中包含了计算光照模型的代码：

```c#
float4 frag(v2f i) : SV_Target 
{ 
    //对后面计算需要的方向矢量进行归一化处理
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
	fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
	fixed3 worldHalfDir = normalize(worldLightDir + worldViewDir);
	
    //对纹理采样，并计算材质的反射系数
	fixed4 c = tex2D (_MainTex, i.uv);
	fixed3 albedo = c.rgb * _Color.rgb;
	
    //计算环境光照
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
			
    //计算当前世界坐标下的阴影值
	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				
    //半兰伯特光照模型计算漫反射系数
	fixed diff =  dot(worldNormal, worldLightDir);
	diff = (diff * 0.5 + 0.5) * atten;
    //使用这个漫反射系数对渐变纹理_Ramp进行采样，在去计算最后的漫反射光照
	fixed3 diffuse = _LightColor0.rgb * albedo * tex2D(_Ramp, float2(diff, diff)).rgb;
				
    //计算高光反射，使用fwidth函数来精确w值。
	fixed spec = dot(worldNormal, worldHalfDir);
	fixed w = fwidth(spec) * 2.0;
	fixed3 specular = _Specular.rgb * lerp(0, 1, smoothstep(-w, w, spec + _SpecularScale - 1)) * step(0.0001, _SpecularScale);//使用step(0.0001, _SpecularScale)是为了在_SpecularScale为0时，实现完全消除高光反射的光照
				
	return fixed4(ambient + diffuse + specular, 1.0);
}
```

### 20.2 素描风格的渲染

通过提前生成的素描纹理来实现实时的素描风格渲染，这些纹理组成了一个色调艺术映射。

这就要求使用多张素描纹理来进行渲染，也可以使用多级渐远纹理的生成。

①声明需要的各个属性（我们这里直接选择使用六张素描纹理来实现素描风格的渲染）

```c#
Prperties{
    _Color ("Color Tint", Color) = (1,1,1,1)
    _TileFactor ("Tile Factor", Float) = 1
    _Outline ("Outline", Rnage(0,1)) = 0.1
    _Hatch0 ("Hatch 0") = "white" {}
    _Hatch1 ("Hatch 1") = "white" {}
    _Hatch2 ("Hatch 2") = "white" {}
    _Hatch3 ("Hatch 3") = "white" {}
    _Hatch4 ("Hatch 4") = "white" {}
    _Hatch5 ("Hatch 5") = "white" {}
}
```

_Color适用于控制模型颜色的属性； _TileFactor是纹理的平铺系数； _TileFactor越大，模型上的素描线条越密。 _Hatch0~5对应着六张素描纹理

②素描也会使用轮廓线的渲染，所以会使用到20.1的渲染轮廓线的Pass

```c#
SubShader {
    Tags {"RenderType" = "Opaque" "Queue" = "Geometry"}
    UsePass ".../.../.../Toon Shading/OUTLINE"
}
```

注意Unity内部会把Pass的名称全部转成大写格式，所以就需要我们在UsePass中使用大写格式的Pass名称。

③定义光照模型所在的Pass

```c#
Pass{
    Tags {"LightMode" = "ForwardBase"}
    
    CGPROGRAM
        
    #pragma vertex vert
    #pragma fragment frag
        
    #pragma multi_compile_fwdbase
}
```

④在顶点着色器中计算六张纹理的混合权重，在v2f结构体中添加相应的变量

```c#
struct v2f{
    float4 pos : SV_POSITION;
    float2 uv : TEXCOORD0;
    fixed3 hatchWeights0 : TEXCOORD1;
    fixed3 hatchWeights1 : TEXCOORD2;
    float3 worldPos : TEXCOORD3;
    SHADOW_COORDS(4);
}
```

我们声明了六张纹理，也就是我们需要6个混合权重，我们将它们存储在两个fixed3类型的变量（hatchWeights0和hatchWeights1）中。声明worldPos和使用SHADOW_COORDS宏声明阴影纹理的采样坐标是为了添加阴影效果。

⑤关键的顶点着色器

```c#
v2f vert(a2v v){
    v2f o;
    
    o.pos = UnityObjectToClipPos(v.vertex);
    o.uv = v.texcoord.xy * _TileFactor;
    
    fixed3 worldLightDir = normalize(WorldSpaceLightDir(v.vertex));
    fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
    fixed diff = saturate(dot(worldLightDir, worldNormal));
    
    o.hatchWeights0 = fixed3(0,0,0);
    o.hatchWeights1 = fixed3(0,0,0);
    
    float hatchFactor = diff * 0.7;
	if (hatchFactor > 6.0) {
		// Pure white, do nothing
	} else if (hatchFactor > 5.0) {
		o.hatchWeights0.x = hatchFactor - 5.0;
	} else if (hatchFactor > 4.0) {
		o.hatchWeights0.x = hatchFactor - 4.0;
		o.hatchWeights0.y = 1.0 - o.hatchWeights0.x;
	} else if (hatchFactor > 3.0) {
		o.hatchWeights0.y = hatchFactor - 3.0;
		o.hatchWeights0.z = 1.0 - o.hatchWeights0.y;
	} else if (hatchFactor > 2.0) {
		o.hatchWeights0.z = hatchFactor - 2.0;
		o.hatchWeights1.x = 1.0 - o.hatchWeights0.z;
	} else if (hatchFactor > 1.0) {
		o.hatchWeights1.x = hatchFactor - 1.0;
		o.hatchWeights1.y = 1.0 - o.hatchWeights1.x;
	} else {
		o.hatchWeights1.y = hatchFactor;
		o.hatchWeights1.z = 1.0 - o.hatchWeights1.y;
	}
    
    o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
    
    TRANSFORM_SHADOW(o);
    
    return o;
}
```

使用_TileFactor得到纹理采样坐标，再计算逐顶点光照得到漫反射系数diff。我们一开始先将权重值初始化为0，并将diff缩放至[0,7]范围得到hatchFacor，并将其均匀划分为7个子区间，判断hatchFactor所处的子区间来计算对应的纹理混合权重。最后计算顶点的世界坐标并使用TRANSFORM_SHADOW宏来计算阴影纹理采样坐标。

⑥片元着色器部分

```c#
fixed4 frag(v2f i) : SV_Target 
{			
	fixed4 hatchTex0 = tex2D(_Hatch0, i.uv) * i.hatchWeights0.x;
	fixed4 hatchTex1 = tex2D(_Hatch1, i.uv) * i.hatchWeights0.y;
	fixed4 hatchTex2 = tex2D(_Hatch2, i.uv) * i.hatchWeights0.z;
	fixed4 hatchTex3 = tex2D(_Hatch3, i.uv) * i.hatchWeights1.x;
	fixed4 hatchTex4 = tex2D(_Hatch4, i.uv) * i.hatchWeights1.y;
	fixed4 hatchTex5 = tex2D(_Hatch5, i.uv) * i.hatchWeights1.z;
	fixed4 whiteColor = fixed4(1, 1, 1, 1) * (1 - i.hatchWeights0.x - i.hatchWeights0.y - i.hatchWeights0.z - i.hatchWeights1.x - i.hatchWeights1.y - i.hatchWeights1.z);
				
	fixed4 hatchColor = hatchTex0 + hatchTex1 + hatchTex2 + hatchTex3 + hatchTex4 + hatchTex5 + whiteColor;
				
	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
								
	return fixed4(hatchColor.rgb * _Color.rgb * atten, 1.0);
}
```

在顶点着色器中得到6张纹理的混合权重后，我们在片元着色器中对每张纹理采样并和它们对应的权重值相乘得到每张纹理的采样颜色，包括了纯白在渲染中的贡献度，这是通过从1中减去所有6张纹理的权重来得到的。最后的结果是混合各个颜色值并和阴影值、模型颜色相乘后的结果。

## 21 噪声

噪声的用处很多，比如纹理的溶解、火焰、波纹等。

### 21.1 消融效果

在游戏中你很经常见到地图被烧毁的的效果，这就是消融，还被用于人物的死亡等。

消融的原理是：噪声纹理+透明度测试。我们将对噪声纹理采样的结果和某个控制消融程度的阈值比较，如果小于阈值，就是用clip函数把它对应的像素裁剪掉，这些部分就是被烧毁的区域。而镂空区域边缘的烧焦效果则是将两种颜色混合，再用pow函数处理后，与原纹理颜色混合后的结果。

①声明各个属性

```c#
Properties{
    _BurnAmount ("Burn Amount", Range(0.0, 1.0)) = 0.0
    _LineWidth ("Burn Line Width", Range(0.0, 2.0)) = 0.1
    _MainTex ("Base (RGB)", 2D) = "White"{}
    _BumpMap ("Normal Map", 2D) = "bump" {}
    _BurnFirstColor("Burn First Color", Color) = (1, 0, 0, 1)
	_BurnSecondColor("Burn Second Color", Color) = (1, 0, 0, 1)
	_BurnMap("Burn Map", 2D) = "white"{}
}
```

_BurnAmount用于控制消融程度，当值为0时，物体为真常效果，当值为1时，物体就完全消融。 _LineWidth用于控制模拟烧焦效果时的线宽，它的值越大，火焰边缘的蔓延范围越广。 _MainTex和 _BumpMap分别对应了物体原本的漫反射纹理和法线纹理。 _BurnFirstColor和 _BurnSecondColor对应了火焰边缘的两种颜色。 _BurnMap则是关键的噪声纹理。

②定义消融所需的Pass

```c#
Pass{
    Tags {"LightMode" = "ForwardBase"}
    
    Cull Off
    
    CGPROGRAM
        
    #include "Linghting.cginc"
    #include "AutoLight.cginc"
        
    #pragma multi_compile_fwdbase
}
```

关键是使用Cull命令关闭了面片剔除，也就是模型的正面和背面都会被渲染。这是因为消融会导致裸露模型内部的构造，如果只渲染正面就会出现错误的结果。

③定义顶点着色器

```C#
struct a2v{
    float4 vertex : POSITION;
    float3 normal : NORMAL;
    float4 tangent : TANGENT;
    float4 texcoord : TEXCOORD0;
};

struct v2f{
    float4 pos : SV_POSITION;
    float2 uvMainTex : TEXCOORD0;
    float2 uvBumpMap : TEXCOORD1;
    float2 uvBurnmap : TEXCOORD2;
    float3 lightDir : TEXCOORD3;
    float3 worldPos : TEXCOORD4;
    SHADOW_COORDS(5);
};

v2f vert(a2v v){
    v2f o;
    o.pos = UnityObjectToClipPos(v.vertex);
    
    o.uvMainTex = TRANASFORM_TEX(v.texcoord, _MainTex);
    o.uvBumpMap = TRANASFORM_TEX(v.texcoord, _BumpMap);
    o.uvBurnMap = TRANASFORM_TEX(v.texcoord, _BurnMap);
    
    TANGENT_SPACE_ROTATION;
    o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
    
    o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
    
    TRANSFER_SHADOW(o);
    
    return o;
}
```

使用TRANSFORM_TEX计算三张纹理对应的纹理坐标，再把光源方向从模型空间变换到切线空间。最后为了得到阴影信息，使用TRANSFER_SHADOW来计算世界空间下的顶点位置和阴影纹理的采样坐标。

④在片元着色器中来模拟消融效果

```c#
fixed4 frag(v2f i) : SV_POSITION{
    fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
    
    clip(burn.r - _BurnAmount);
    
    float3 tangentLightDir = normalize(i.lightDir);
    fixed3 tangentNormal = UnpackedNormal(tex2D(_BumpMap, i.uvBumpMap));
    
    fixed3 albedo = tex2D(_MainTex, i.uvMainTex).rgb;
    
    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
    
    fixed3 diffuse = _LightColor0.rgb * albedo * saturate(dot(tangentNormal, tangentLightDir));
    
    fixed t = 1 - smoothstep(0.0, _LineWidth, burn.r - _BurnAmount);
    fixed3 burnColor = lerp(_BurnFirstColor, _BurnSecondColor, t);
    burnColor = pow(burnColor, 5);
    
    UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
    fixed3 finalColor = lerp(ambient + diffuse * atten, burnColor, t * step(0.001, _BurnAmount));
    
    return fixed4(finalColor, 1);
}
```

先对噪声纹理进行采样，并将采样结果与用于控制消融程度的_BurnAmount相减，将结果传递给clip函数。当结果小于0时就剔除该像素，从而不会显示这个像素。先根据漫反射纹理得到材质的反射率；再计算环境光，从而计算得到，漫反射光照。再去计算烧焦颜色——在宽度为 _LineWidth的范围内模拟一个烧焦的颜色变化：第一步是利用前面的边界检测，使用smoothStep函数来计算混和系数t，当t值为1时，表明该像素位于消融的边界，当t值为0时，表明该像素为正常的模型颜色，平滑的步长为burn.r - _BurnAmount，表示需要模拟的一个烧焦效果。之后用这个t值来混和两种火焰的颜色： _BurnFirstColor和 _BurnSecondColor，为了更加逼真的模拟出烧焦痕迹，使用pow函数来对结果进行处理。再使用t值来混和正常光照和烧焦颜色，其中使用step函数来保证当 _BurnAmount为0时，不显示任何消融效果，最后再返回混合的颜色值finalColor。

⑤除上述的Pass以外，我们还定义了一个用于投射阴影的Pass，以避免被剔除的区域仍能想其他物体投射阴影。

```c#
Pass{
    Tags {"LightMode" = "ShadowCaster"}
    
    CGPROGRAM
        
    #pragma vertex vert
	#pragma fragment frag
			
	#pragma multi_compile_shadowcaster
}
```

需要声明的属性：

```c#
fixed _BurnAmount;
sampler2D _BurnMap;
float4 _BurnMap_ST;
```

顶点着色器和片元着色器：

```c#
struct v2f 
{
	V2F_SHADOW_CASTER;
	float2 uvBurnMap : TEXCOORD1;
};
			
v2f vert(appdata_base v) 
{
	v2f o;
			
	TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				
	o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				
	return o;
}
			
fixed4 frag(v2f i) : SV_Target 
{
	fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				
	clip(burn.r - _BurnAmount);
				
	SHADOW_CASTER_FRAGMENT(i)
}
```

阴影投射的重点是我们需要按照正常Pass的处理来剔除片元或进行顶点动画，以便阴影可以和物体正常渲染的结果相匹配。在自定义的阴影投射的Pass中，我们使用Unity提供的内置宏V2F_SHADOW_CASTER、TRANSFER_SHADOW_CASTER_NORMALOFFSET和SHADOW_CASTER_FRAGMENT来计算阴影投射时需要的各种变量。在上述代码中，我们首先在v2f结构体中利用V2F_SHADOW_CASTER来定义阴影投射需要定义的变量。接着在顶点着色器中计算噪声纹理的采样坐标uvBurnMap，然后在片元着色器中按之前所说的处理方法使用噪声纹理的采样结果来剔除片元，最后再利用SHADOW_CASTER_FRAGMENT来让Unity完成阴影投射的部分，把结果输出到深度图和阴影映射纹理中。

通过Unity的这三个内置宏我们可以非常方便的自定义需要的阴影投射的Pass，但不便的是这些宏需要使用一些特定的输入变量，例如：TRANSFER_SHADOW_CASTER_NORMALOFFSET会使用v作为输入结构体，v中需要包含顶点位置v.vertex和顶点法线v.normal，这个可以直接使用内置的appdata_base结构体。若需要顶点动画，就在顶点着色器中修改v.vertex，再传递给TRANSFER_SHADOW_CASTER_NORMALOFFSET。

### 21.2 水波效果

在模拟是实时水面效果的过程中我们也会使用到噪声纹理：将噪声纹理用作一个高度图，以不断修改水面的法线信息。同时为了实现模拟出水不断流动的效果，会使用与时间相关的变量来对噪声纹理进行采样，得到法线信息后，再进行正常的反射+折射计算，得到最后的水面波动效果。一般情况下，我们是实现的菲涅尔发射。

使用一张立方体纹理作为环境纹理，使用GrabPass来获取当前屏幕的渲染纹理，并使用切线空间下的法线方向对像素的屏幕坐标进行偏移，再使用该坐标对渲染纹理进行屏幕采样，从而模拟出折射效果。对发射和折射的颜色混合，我们使用菲涅尔系数来动态决定混合系数：
$$
fresnel = pow(1-saturate(v\cdot n),4)
$$
v和n分别对应了视角方向和法线方向。它们之间的夹角越小，fresnel值越小，反射越弱，折射越强，菲涅尔系数还通常用于边缘光照的计算。

①声明需要使用的各个属性

```c#
Properties {
    _Color ("Main Color", Color) = (0, 0.15, 0.115, 1)
    _MainTex ("Base (RGB)", 2D) = "White" {}
    _WaveMap ("Wave Map", 2D) = "bump" {}
    _CubeMap ("Environment CubeMap", Cube) = "_Skybox" {}
    _WaveXSpeed ("Wave Horizontal Speed", Range(-0.1, 0.1)) = 0.01
    _WaveYSpeed ("Wave Vertical Speed", Range(-0.1, 0.1)) = 0.01
    _Distortion ("Distortion", Range(0, 100)) = 10
}
```

_Color用于控制水面颜色； _MainTex是水面波纹材质纹理； _WaveMap是一个由噪声纹理生成的法线纹理； _CubeMap是用于模拟反射的立方体纹理； _Distortion则用于控制模拟折射时图像的扭曲程度； _WaveXSpeed和 _WaveYSpeed分别用于控制法线纹理在X和Y方向上的平移速度。

②使用GrabPass来获取屏幕图像

```c#
SubShader{
    Tags {"Queue" = "Transparent" "RenderType" = "Qpaque"}
    GrabPass {"_RefractionTex"}
}
```

在SubShader的标签中将渲染队列设置成Transparent，并将RenderType设置为Qpaque。将Queue设置成Transparent可以确保物体在渲染时其他所有不透明的物体都已经被渲染到屏幕上了，否则我们就不能透过水面看到图像。设置RenderType则是为了使用着色器替换时该物体可以在需要时能被正确渲染。这通常发生在需要得到摄像机的深度和法线纹理时。然后通过GrabPass来定义一个抓取屏幕图像的Pass——我们是定义的一个字符串，其名称决定了抓取得到的屏幕图像将会被存入哪个纹理中。

③定义水面所需的Pass的变量：

```c#
fixed4 _Color;
sampler2D _MainTex;
float4 _MainTex_ST;
sampler2D _WaveMap;
float4 _WaveMap_ST;
samplerCUBE _CubeMap;
fixed _WaveXSpeed;
fixed _WaveYSpeed;
float _Distortion;
sampler2D _RefractionTex;
float4 _RefractionTex_TexelSize;
```

_RefractionTex_TexelSize是我们通过GrabPass抓取到的纹理的大小，我们将在对屏幕图像的采样坐标进行偏移时使用该变量。

④定义顶点着色器

```c#
struct v2f {
    float4 pos : SV_POSITION;
    float4 scrPos : TEXCOORD0;
    flaot4 uv : TEXCOORD1;
    float4 TtoW0 : TEXCOORD2;
    float4 TtoW1 : TEXCOORD3;
    float4 TtoW2 : TEXCOORD4;
};

v2f vert (a2v v){
    v2f o;
    o.pos = UnityObjectToClipPos(v.vertex);
    
    o.srcPos = ComputeGrabScreenPos(o.pos);
    
    o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
    o.uv.zw = TRANSFORM_TEX(v.texcoord, _WaveMap);
    
    float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
    fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
    fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
    fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w;
    
    o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
    o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
    o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);   
    
    return o;
}
```

在进行顶点变换后，通过调用ComputeGrabScreenPos来得到对应被抓取屏幕图像的采样坐标。再计算_MainTex和 _WaveMap的采样坐标，并将它们存储在float4变量的xy和zw分量中。然后按照之前所学的方法求解从切线空间到世界空间的变换矩阵。

⑤定义片元着色器

```c#
fixed4 frag(v2f i) : SV_Target 
{
	float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
	float2 speed = _Time.y * float2(_WaveXSpeed, _WaveYSpeed);
				
	//再切线空间中获得法线信息
	fixed3 bump1 = UnpackNormal(tex2D(_WaveMap, i.uv.zw + speed)).rgb;
	fixed3 bump2 = UnpackNormal(tex2D(_WaveMap, i.uv.zw - speed)).rgb;
	fixed3 bump = normalize(bump1 + bump2);
				
	//在切线空间中计算偏移量
	float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;
	i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy;
	fixed3 refrCol = tex2D( _RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;
				
	// 将法线转移到世界空间
	bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
	fixed4 texColor = tex2D(_MainTex, i.uv.xy + speed);
	fixed3 reflDir = reflect(-viewDir, bump);
	fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb * _Color.rgb;
				
	fixed fresnel = pow(1 - saturate(dot(viewDir, bump)), 4);
	fixed3 finalColor = reflCol * fresnel + refrCol * (1 - fresnel);
				
	return fixed4(finalColor, 1);
}
```

先利用TtoW的w分量来得到世界坐标，并利用该值得到该片元对应的视角方向。此外使用Unity内置的_Time.y分量和 _WaveXSpeed、 _WaveYSpeed属性来计算法线纹理的当前偏移量，并利用该值对法线纹理来进行采样——以模拟出两层交叉的水面波动效果；然后使用该值和 _Distortion、 _RefractionTex_TexelSize来对屏幕图像的采样坐标进行偏移，以模拟折射效果。 _Distortion值越大，偏移量越大，水面背后的物体看起来变形程度最大。将偏移量与屏幕坐标的z值相乘是为了模拟深度越大、折射程度越大的效果，当然也是可以直接将偏移值叠加到屏幕坐标上。随后对屏幕坐标进行透视除法，再使用该坐标对抓取的屏幕图像 _RefractionTex进行采样以得到模拟的折射颜色。

最后将法线方向从切线空间变换到世界空间下，并据此得到视角方向相对于法线方向的反射方向，并用这个反射方向对Cubemap进行采样，并将结果与主纹理颜色相乘后得到反射颜色。我们也对主纹理进行了纹理动画，以模拟水波效果。

为了混合折射和反射颜色，我们是计算的菲涅尔系数。

### 21.3 使用噪声实现全局雾效

我们前面是通过深度纹理来实现的基于屏幕后处理技术的全局雾效——通过深度纹理来重建每个像素在世界空间下的位置，再使用一个基于高度的公式来计算雾效的混合系数，最后使用该系数来混合雾的颜色和原屏幕的颜色。

这实际上是一个基于高度的均匀雾效——即在同一个高度上，雾的浓度是相同的。但我们很多时候所想的是模拟出一种不均匀的、在不断飘动的雾。 

这就需要使用一张噪声纹理来实现。实现的主体内容与之前的基本一样，只是在Shader的片元着色器中对高度的计算加入了对噪声的影响。

（1）C#脚本

①继承于之前的PostEffectsBase基类

②声明该效果需要的shader，并创建相应的材质：

```c#
public Shader fogShader;
private Material fogMaterial = null;

public Material material{
    get{
        fogMaterial = CheckShaderAndCreateMaterial(fogShader, fogMaterial);
        return fogMaterial;
    }
}
```

③我们需要获取摄像机的相关参数，包括近裁剪平面的距离、FOV等，同时还需要获取摄像机在世界空间下的前方、上方、右方等方向，因此声明两个变量来存储摄像机的Camera和Transform组件。

```c#
private Camera myCamera;
public Camera camera{
    get{
        if (myCamera == null){
            myCamera = GetComponent<Camera>();
        }
        return myCamera;
    }
}

private Transform myCameraTransform;
public Transform cameraTransform{
    get{
        if (myCameraTransform == null){
            myCameraTransform = camera.transform;
        }
        return myCameraTransform;
    }
}
```

④定义模拟雾效是需要的参数

```c#
[Rnage (0.1f, 3.0f)]
public float fogDensity = 1.0f;

public Color fogColor = Color.white;

public float fogStart = 0.0f; 
public float fogEnd = 2.0f;

public Texture noiseTexture;

[Range (-0.5f, 0.5f)]
public float fogXSpeed = 0.1f;

[Range (-0.5f, 0.5f)]
public float fogYSpeed = 0.1f;

[Range (0.0f, 3.0f)]
public float noiseAmount = 1.0f;
```

fogDensity用于控制雾的浓度，fogColor用于控制雾的浓度，fogStart和fogEnd分别控制雾效的起始和终止高度，noiseTexture就是我们将会使用的噪声纹理，fogXSpeed和fogYSpeed用于模拟雾的飘动效果。noiseAmount用于控制噪声程度，当其值为0时，表示不应用任何噪声，即得到一个均匀的完全基于高度的全局雾效。

⑤OnEnable函数中设置摄像机的相应状态

```c#
void OnEnable(){
    camera.depthTextureMode |= DepthTextureMode.Depth;
}
```

⑥OnRenderImage函数

```c#
void OnRenderImage(RenderTexture src, RenderTexture dest){
    if (material != null){
        Matrix4x4 frustumCorners = Matrix4x4.idensity;
        
        //计算近裁剪平面四个角对应的向量，具体的数学理论见屏幕后处理中的全局雾效
        float fov = camera.fieldOfView;
        float near = camera.nearClipPlane;
        float far = camera.farClipPlane;
        float aspect = camera.aspect;
        
        float halfHeight = near * Mathf.Tan(fov * 0.5f * Mathf.Deg2Rad);
        Vector3 toRight = cameraTransform.right * halfHeight * aspect;
        Vector3 toTop = cameraTransform.up * halfHeight;
        
        Vector3 topLeft = cameraTransform.forward * near + toTop - toRight;
        float scale = topLeft.magnitude / near;
        
        topLeft.Normalize();
        topLeft *= scale;
        
		Vector3 topRight = cameraTransform.forward * near + toRight + toTop;
		topRight.Normalize();
		topRight *= scale;

		Vector3 bottomLeft = cameraTransform.forward * near - toTop - toRight;
		bottomLeft.Normalize();
		bottomLeft *= scale;

		Vector3 bottomRight = cameraTransform.forward * near + toRight - toTop;
		bottomRight.Normalize();
		bottomRight *= scale;
		
        frustumCorners.SetRow(0, bottomLeft);
		frustumCorners.SetRow(1, bottomRight);
		frustumCorners.SetRow(2, topRight);
		frustumCorners.SetRow(3, topLeft);
        
        //将上述四个向量存储于一个矩阵类型的变量frustumCorners
        material.SertMatrix("_FrustumCornersRay", frustumCorners);
        
        material.SetFloat("_FogDensity", fogDensity);
        material.SetColor("_FogColor", fogColor);
        material.SetFloat("_FogStart", fogStart);
        material.SetFloat("_FogEnd", fogEnd);
        
        material.SetTexture("_NoiseTex", NoiseTex);
        material.SetFloat("_FogXSpeed", FogXSpeed);
        material.SetFloat("_FogYSpeed", FogYSpeed);
        material.SetFloat("_NoiseAmount", NoiseAmount);
        
        Graphics.Blit(src, dest, material);
    }else{
        Graphics.Blit(src, dest);
    }
}
```

（2）shader

①声明各个属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white" {}
	_FogDensity ("Fog Density", Float) = 1.0
	_FogColor ("Fog Color", Color) = (1, 1, 1, 1)
	_FogStart ("Fog Start", Float) = 0.0
	_FogEnd ("Fog End", Float) = 1.0
	_NoiseTex ("Noise Texture", 2D) = "white" {}
	_FogXSpeed ("Fog Horizontal Speed", Float) = 0.1
	_FogYSpeed ("Fog Vertical Speed", Float) = 0.1
	_NoiseAmount ("Noise Amount", Float) = 1
}
```

②使用CGINCLUBE来管理代码

③根据属性来声明变量

```c#
float4x4 _FrustumCornersRay;
		
sampler2D _MainTex;
half4 _MainTex_TexelSize;
sampler2D _CameraDepthTexture;
half _FogDensity;
fixed4 _FogColor;
float _FogStart;
float _FogEnd;
sampler2D _NoiseTex;
half _FogXSpeed;
half _FogYSpeed;
half _NoiseAmount;
```

④由于顶点着色器与屏幕后处理的全局雾效是一样的，所以我们就不再书写，而将注意力更多地放在有区别的片元着色器

```c#
fixed4 frag(v2f i) : SV_Target 
{
	float linearDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth));
	float3 worldPos = _WorldSpaceCameraPos + linearDepth * i.interpolatedRay.xyz;
			
	float2 speed = _Time.y * float2(_FogXSpeed, _FogYSpeed);
	float noise = (tex2D(_NoiseTex, i.uv + speed).r - 0.5) * _NoiseAmount;
					
	float fogDensity = (_FogEnd - worldPos.y) / (_FogEnd - _FogStart); 
	fogDensity = saturate(fogDensity * _FogDensity * (1 + noise));
			
	fixed4 finalColor = tex2D(_MainTex, i.uv);
	finalColor.rgb = lerp(finalColor.rgb, _FogColor.rgb, fogDensity);
		
	return finalColor;
}
```

首先还是根据深度纹理来重建该像素在世界空间中的位置。然后使用内置的_Time.y变量和 _FogXSpeed、 _FogYSpeed属性计算出当前噪声纹理的偏移量，并据此对噪声纹理进行采样得到噪声值。最将该噪声值添加到雾效浓度的计算中，得到应用噪声后的雾效混合系数fogDensity，再使用该系数将雾的颜色和原始颜色进行混合。

⑤定义雾效渲染所需要的Pass

```c#
Pass {          	
	CGPROGRAM  
			
	#pragma vertex vert  
	#pragma fragment frag  
			  
	ENDCG
}
```

### 21.4 关于噪声纹理

噪声纹理究竟是什么呢？可以视为是一种程序纹理，是由计算机利用相关算法生成的，其中Perlin噪声和Worly噪声是两种最常使用的噪声类型。Perlin噪声可以用于生成更加自然的噪声，而Worly噪声则通常用于模拟诸如石头、水、纸张等多孔噪声。这个如法线纹理、深度纹理等，也是可以通过Photoshop进行生成的，但若想更自由地控制噪声纹理的生成，就需要更加深入的去了解这两种噪声的实现原理并进行自己的设计。

