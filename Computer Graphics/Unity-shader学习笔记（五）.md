[TOC]



# Unity-shader学习笔记（五）

这里我们会聊聊基础纹理和透明效果的知识。

## 13 基础纹理

我们用纹理干什么呢？使用它来控制模型的外观，使用的是纹理映射技术来让纹理与模型表面贴住，在逐纹素地控制模型颜色。

在建模时，美术人员会将纹理映射坐标存储在每一个顶点上，它定义了该顶点在纹理中对应的2D坐标。通常这些坐标使用一个二位变量(u,v)来表示，u为横向坐标，v是纵向坐标。这也是我们所称的UV坐标。

我们通常将UV坐标的范围归一化到[0,1]范围内，但在纹理采样时使用的纹理坐标不一定在[0,1]内，因为在纹理的平铺模式下，[0,1]之外的纹理坐标与[0,1]之内的纹理坐标的纹理采样是不同的。

但似乎你还是不知道纹理是什么？纹理坐标又代表什么？

### 13.1 纹理与纹理坐标

纹理可以理解为一个物体身上颜色以及物体表面的光滑和粗糙情况。

而纹理坐标呢？

它实际上是一个二维数组，它的元素是一些颜色值。单个颜色值被称为**纹理元素或纹素**，每一个纹理像素在纹理中都有一个唯一的地址，也就是我们之前所说UV坐标。

当我们将一个纹理应用于一个图元时，它的纹理像素地址必须要映射到对象坐标中，然后再平移到屏幕坐标系或像素位置上去。

### 13.2 单张纹理

在这个例子中，我们使用Blinn-Phong光照模型进行计算：

①在高光反射的Properties中多增加一项纹理属性：

```c#
Properties{
	_Color ("Color Tint", Color) = (1,1,1,1)
	_MainTex ("Main Tex", 2D) = "white" {}
	_Specular("Specular", Color) = (1,1,1,1)
	_Gloss("Gloss", Range(8.0, 256)) = 20
}
```

我们声明了一个名为_MainTex的纹理，使用一个字符串和一个花括号作为它的初值。

②在CG代码块中声明与上述属性相匹配的变量：

```c#
fixed4 _Color;
sampler2D _MainTex;
float4 _MainTex_ST;
fixed4 _Specular;
float _Gloss;
```

对第三个变量_MainTex_ST，这是为纹理类型的属性声明的。

_MainTex_ST.xy存储的是纹理的缩放值；

_MainTex_ST.zw存储的是纹理的偏移值。

这两个值都可以在材质面板的纹理属性中调节。

③定义顶点着色器的输入和输出结构体：

```c#
struct a2v
{
	float4 vertex : POSITION;
	float3 normal : NORMAL;
	float4 texcoord : TEXCOORD0;
};

struct v2f
{
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
	float2 uv : TEXCOORD2;
};
```

在a2v中使用TEXCOORD0语义声明变量texcoord，使Unity将模型的第一组纹理坐标存储到该变量中。

在v2f中添加用于存储纹理坐标变量的uv，以便在片元着色器中使用该坐标进行纹理采样。

有的朋友可能会说，为什么worldMormal和worldPos要声明成纹理坐标？因为这整个shader都是为纹理采样服务的，我们将纹理坐标映射到顶点上去后，依然是要对顶点的值进行反射处理。

④定义顶点着色器：

```c#
v2f vert(a2v v)
{
	v2f o;
					
	o.pos = UnityObjectToClipPos(v.vertex);
					
	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
	o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;

	return o;
}
```

先使用缩放属性_MainTex_ST.xy将纹理坐标映射到顶点坐标上去，再使用偏移属性 _MainTex_ST.zw对结果进行偏移。这样之后纹理就能显示在屏幕坐标系中了。

⑤最后在片元着色器中计算光照反射模型，这里使用的就不再是顶点的原始坐标，而是纹理映射后的坐标。

```c#
fixed4 frag(v2f i) : SV_Target
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));

	fixed3 aldebo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
	
    //自然光
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * aldebo;
	//漫反射
	fixed3 diffuse = _LightColor0.rgb * aldebo * max(0, dot(worldNormal, worldLightDir));
	//高光反射
	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
	fixed3 halfDir = normalize(worldLightDir + viewDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

	return fixed4(ambient + diffuse + specular, 1.0);
}
```

最主要的区别就是使用tex2D函数对纹理进行采样：第一个参数是需要被采样的纹理，第二个参数是存储返回计算得到的纹素值的变量。

材质最终反射出的颜色就是采样的结果与颜色属性_Color的乘积。在计算环境光时，就不仅仅像之前的只是一个UNITY_LIGHTMODEL_AMBIENT.xyz了，也要乘上材质反射出的颜色。

高光反射的计算依然没有变化，只是每个值的内容变化了而已。

当然最后别忘了Fallback。

### 13.3 纹理的属性

这样写完之后你是不是会感慨：原来纹理映射是如此的简单。但实际上，在渲染流水线中，纹理映射的实现比我们所想象的要复杂得多。接下俩就会聊聊关于纹理属性的知识。

其实也主要是了Wrap Mode和Filter Mode的内容，它决定了当纹理坐标超过[0,1]范围后将会如何被平铺。

（1）在笔者使用的Unity2018.4.27中Wrap Mode一共有五种类型：

①Repeat重复模式

在这种模式下，如果纹理坐标超过了1，那么它的整数部分就会被舍弃而直接使用小数部分进行采样，结果就是纹理会不断重复；

②Clamp边缘拉伸模式

在这种模式下，如果纹理坐标超过了1，就会截取到1，如果小于0，就会截取到0；

③Mirror镜像模式

超出的部分将以镜像的方式展开

④Mirror Once一次镜像

字面理解起来是镜像一次，但不知道为什么它的效果跟Camp的效果是一样的；

⑤Per-axis平铺

分别对U、V方向超出部分分开做平铺处理。

（2）Filter Mode属性，它决定了当纹理由于变换而产生拉伸时将会采用哪种滤波器。它有三种模式：Point、Bilinear、Trilinear，它们得到的图片的滤波效果一依次提升，但耗费的性能也依次增大，

有朋友可能不理解滤波是干嘛的。你回想自己在放大或缩小一张图片时，总是会影响图片的清晰度（质量），这就是由滤波影响的。

既然提到了图片的缩小和放大，那就扩展地说说吧。

缩放就是原纹理中多个像素会对应一个目标像素，它比放大更加复杂一点，主要就在处理抗锯齿问题。处理它的主要办法就是**多级渐远纹理技术**。

它的原理是：将原纹理提前用滤波处理得到很多更小的图片，形成一个图像金字塔，每一层就是对上一层图像降采样的结果。

通过这种技术，就能快速得到结果像素。例如，当物体远离摄像机时就可以直接使用较小的纹理。当然缺点也很明显，就是需要额外的空间来存储每层的较小的纹理。（非常典型的用空间换时间的方法）

Point使用的是nearest neighbor滤波（最邻近滤波），在放大或缩小时它的采样像素数目通常只有一个，这也就导致放大或缩小后看起来会有像素风格；

Bilinear使用的是Linar滤波（线性滤波），对于每个目标像素，它会找到四个邻近像素，找到后会对它们进行线性插值混合，最后的效果看起来像是被模糊了；

Trilinear使用的也几乎时线性滤波，但后者只是会多出在多级渐远纹理之间的进行混合。如果一个纹理没有使用多级渐远纹理技术，那么这两种模式的结果是一样的。

### 13.4 凹凸映射

正如其名，这种映射会让模型看起来凹凸不平——使用一张纹理来修改模型表面的法线，以便为模型提供更多的细节。要注意，这种映射并不会改变模型的顶点位置。

有两种方法：

①高度映射：使用一张高度纹理来模拟表面位移，然后得到一个修改后的法线值；

②法线映射：使用一张法线纹理来直接存储表面法线。

#### 13.4.1 高度映射

高度图中存储的是强度值，它用于表示模型表面局部的海拔高度。于是：

颜色越浅表明该位置的表面越向外凸起，颜色越深表明该位置的表面越向内凹陷。

优点：非常直观；

缺点：计算相当复杂，并且在实时计算的时候不能直接得到表面法线，而是需要由像素的灰度值计算得到，要消耗大量的额外性能。

#### 13.4.2 法线映射

法线映射就是要利用存储的法线，但法线的分量范围在[-1,1]，像素的分量范围[0,1]，所以要做一个映射：
$$
pixel = \frac{normal + 1}{2}
$$
可我们纹理采样得到的像素值，所以还要对上述的映射过程进行反映射，及求解逆函数：
$$
normal = pixel*2-1
$$
此时就会涉及到坐标空间的选择问题，pixel是像素空间也就是纹理空间中的，normal是模型空间中的。

有一种直接的想法是将修改后的模型空间中的表面法线存储在一张纹理中，这种纹理我们称之为模型空间的法线纹理。但在实际应用中，我们选择的时顶点的切线空间。对每个顶点都与属于自己的切线空间：空间原点为该顶点本身，z轴时顶点的法线方向，x轴是顶点的切线防线，y轴可有法线与切线叉积得到，也称y轴为副切线。这种纹理我们称之为切线空间的法线纹理。

但法线贴图有一点缺陷，就是，模型空间下看法线纹理是五颜六色的，但在切线空间下看是有一点偏蓝色的，为什么呢？

因为法线扰动问题。

一个模型有多个面构成，这些面在高模的细节中会出现凹凸起伏，但是在低模中这些凹凸细节就会被忽略，从而就会改由法线贴图记录的切线空间的法线信息来替代这些凹凸信息被记录。

在模型空间下，每个点存储的法线方向是各异的，比如(0,1,0)和(0,-1,0)两条法线，在映射后变换为(0.5,1.0,0.5)和(0.5,0.0,0.5)，一个浅绿色，一个紫色，所以会呈现为五颜六色；

而切线空间下，每条切线都是(0,0,1)，转换后为(0.5,0.5,1.0)，就为浅蓝色。

这两者各有什么优点呢？

（1）模型空间

①在纹理坐标的缝合处和尖锐的边角部分，可见的突变（缝隙）较少，及可以提供平滑的边界。

因为在模型空间下的法线纹理存储的是同一坐标下的法线信息，因此在边界处通过插值得到的发现就能够平滑变换。

相反，

在切线空间下的法线纹理在法线信息是依靠纹理坐标的方向得到的结果，就可能会导致在边缘处或尖锐的部分造成更多可见的缝合迹象。

②实现上的简单。

计算量相当得少，甚至不需要模型原始的发现和切线等信息。

相反，

切线空间下，由于模型的切线一般是和UV方向相同的，也就是说要想得到效果很好的法线映射就要要求纹理映射也是连续的。

（2）切线空间

①很高的自由度。

怎么理解这个自由度呢？模型空间下的法线纹理记录的是绝对的法线信息，只能用于创建的那个模型，换个模型就错了。

相反，

在切线空间下，法线纹理记录的是相对法线信息，也就是说即便把该纹理应用到一个完全不同的网格上也能得到一个合理的结果。

②可进行UV动画。

不知道朋友们看见游戏里面那些高质量的流动的水、喷发的火山岩浆、狂涌的大海有没有一种冲动？其中很多就用到了UV动画。

③可压缩。

由于切线空间下的法线纹理中法线的Z方向总是正方向，所以就只需要存储XY方向而推导得到Z方向。在模型空间下的法线纹理由于每个方向都是可能的，所以急需要存储三个方向的值。

在现实工业生产中，使用最多的还是切线空间。

#### 13.4.3 实践

基于上述内容，有两种计算方法：一种是选择在切线空间下进行光照计算，此时我们需要将光照方向、视角方向变换到切线空间下；另一种选择是在世界空间下进行光照计算，需要把采样得到的法线方向变换到世界空间下，再和世界空间下的光照方向和视角方向进行计算。

##### 13.4.3.1 在切线空间下计算

在切线空间下计算光照模型的基本思路是：

在片元着色器中通过纹理采样得到切线空间下的法线，然后再与切线空间下的视角方向、光照方向等进行计算，得到族中的光照结果。所以，第一步是在顶点着色器中把视角方向和光照方向从模型空间变换到切线空间。

具体的实现如下：

①在Properties中除了熟悉的属性以外，还加入了法线纹理的属性和控制凹凸程度的属性。

```c#
Properties
{
	// 纹理贴图代替漫反射
	_Color("Color",color) = (1,1,1,1)
	_MainTex("Main Tex",2D) = "white"{}
	// 高光反射
	_Specular("Specular",color) = (1,1,1,1)
	_Gloss("Gloss",Range(0,20)) = 20
	// 凹凸映射
	_BumpMap("Bump Map",2D) = "bump"{} // bump是Unity内置的法线纹理，当没有提供任何法线纹理时，bump对应了模型自带的法线信息
	_BumpScale("Bump Scale",Float) = 0.5 // 控制凹凸程度，为0时，意味着该法线纹理不会对光照产生任何影响
}
```

bump是Unity内置的法线纹理，当没有提供任何法线纹理时，bump对应了模型自带的法线信息;

_BumpScale控制凹凸程度，为0时，意味着该法线纹理不会对光照产生任何影响。

②在CG代码块中声明属性相应的变量

```c#
fixed4 _Color;
sampler2D _MainTex;//主纹理
float4 _MainTex_ST;
sampler2D _BumpMap;//法线贴图（Normal Map）
float4 _BumpMap_ST;
float _BumpScale;
fixed4 _Specular;
float _Gloss;
```

_MainTex_ST 与 _BumpMap_ST是为了获得 _MainTex与 _BumpMap的平铺和偏移系数，其xy分量是平铺系数，zw分量是偏移系数。

③顶点着色器的输入结构体：

```c#
struct a2v
{
	float4 position : POSITION;
	float3 normal : NORMAL; // 法线
	float4 tangent : TANGENT; // 切线
	float4 texcoord : TEXCOORD0; // 第一组纹理坐标
};
```

这里要区别tangent和normal的区别：前者是法线方向，且变量类型为float4，后者是切线类型，且变量类型为float3。因为我们要使用tangent.w分量来表示那个切线空间中副切线的方向。

④顶点着色器的输出结构体：

```c#
struct v2f
{
	float4 pos:SV_POSITION; // 顶点坐标变换
	float4 uv:TEXCOORD0;
	float3 tangLightDir:TEXCOORD1; // 光照方向，从模型空间变换到切线空间
	float3 tangViewDir:TEXCOORD2; // 视角方向
};
```

增加了两个用来存储变换后的光照和视角方向的变量tangLightDir和tangViewDir。

⑤定义顶点着色器：

```c#
v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.position);

	o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
	o.uv.zw = TRANSFORM_TEX(v.texcoord, _BumpMap);
	//*TRANSFORM_TEX是拿顶点的uv和材质球的tiling和offset做运算，以确保材质球里的缩放和偏移设置是正确的
	//与o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;是等价的

	// 法线，切线得到副切线binormal
	float3 binormal = cross(normalize(v.normal), normalize(v.tangent.xyz)) * v.tangent.w;
				
	// 构建从模型空间到切线空间的矩阵
	float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
				
	// 空间变换
	o.tangLightDir = mul(rotation, ObjSpaceLightDir(v.position));
	o.tangViewDir = mul(rotation, ObjSpaceViewDir(v.position));
	return o;
}
```

顶点着色器要根据_MainTex与 _BumpMap的平铺和偏移系数来计算需要输出的UV纹理坐标：其xy分量是 _MainTex的纹理坐标，zw分量是 _BumpMap的纹理坐标。

在计算副切线时，先计算normal与tangent的叉积（注意，cross函数的两个参数只能各含有三个分量），根据右手定则确定它的方向，在叉乘tangent的第四个分量得到副切线的方向。

再根据求出的副切线和得到的法线、切线算出rotation变换矩阵，这里也可以用Unity内置宏TANGENT_SPACE_ROTATION来直接得到，其实现就是上面代码中的那个等式。

最后再根据计算得到的rotation，使用Unity内置函数ObjSpaceLightDir和ObjSpaceViewDir得到的模型空间下的光照和视角方向分别计算得到切线空间吓得我光照和视角方向。

⑥在片元着色器中采样得到切线空间下的法线空间

```c#
fixed4 frag(v2f i) :SV_Target
{
	// 归一化
	fixed3 tangLightDir = normalize(i.tangLightDir);
	fixed3 tangViewDir = normalize(i.tangViewDir);
				
    // 纹理采样
	fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
	fixed3 tangentNormal;
				
	// 如果纹理图不是 normal map
	// tangentNormal.xy = (packedNormal.xy * 2 - 1) *_BumpScale;
	// tangentNormal.z = sqrt(1-saturate(dot(tangentNormal.xy, tangentNormal.xy))); 
	// 或者标识为 Normal map
	tangentNormal = UnpackNormal(packedNormal);
				
	// 如果没有 _BumpScale， 下面两步可以不执行
	tangentNormal.xy *= _BumpScale;
	tangentNormal.z = sqrt(1 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

	// 反射率
	fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
	// 环境光
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
	// 漫反射
	fixed3 diffuse = _LightColor0.rgb * albedo * saturate(dot(tangentNormal, tangLightDir));
				
	// 高光反射
	fixed3 halfDir = normalize(tangViewDir + tangLightDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(tangentNormal, halfDir)), _Gloss);
				
	return fixed4(ambient + diffuse + specular, 1.0);
}
```

归一化处理就不再说了，以后也基本上不会再提。

首先利用tex2D对_BumpMap在输入点上进行采样，并将其颜色赋给packedNormal。

可我们要得到的是法线啊，毕竟我们是要根据法线来进行插值的？还记得13.4.2的一开始吗？

是的，进行反映射。就为代码块中被注释部分。

我没有使用这种方法，也是因为直接将图像设置为了Normal Map，但这种思想理解是要有的，毕竟万一哪天你忘了将图片设置为Normal Map呢，是吧。

接下来计算环境光、漫反射和高光反射的与之前的相同就不再复述

##### 13.4.3.2 在世界空间下的计算

在世界空间下计算光照模型的基本思路是：

在顶点着色器中计算从切线空间到世界空间的变换矩阵，并传递给片元着色器。变换矩阵的计算可以由顶点的切线、副切线和法线在世界空间下的表示来得到。最后再在片元着色器中将法线纹理中的法线方向从切线空间变换到世界空间。

①修改顶点着色器的输出结构体

```c#
struct v2f
{
	float4 pos : SV_POSITION; // 顶点坐标变换
	float4 uv : TEXCOORD0;
	float4 tanx : TEXCOORD1;
	float4 tany : TEXCOORD2; // 光照方向，从模型空间变换到切线空间
	float4 tanz : TEXCOORD3; // 视角方向
};
```

相比于切线空间，我们去除了光照方向和视角方向的变量，引入三个tan，xyz的tan，依次存储了从切线空间到世界空间的变换矩阵的每一行。你可能会问，为什么不是float3，因为在切线空间中我们所求的变换矩阵是float3*3？这是为了充分利用插值寄存器的存储空间，一个插值寄存器最多只能存储float4大小的变量，w分量用来存储世界空间下的顶点位置。

②修改顶点着色器，在顶点着色器中计算从切线空间到世界空间的变换矩阵

```c#
v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.position);

	o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
	o.uv.zw = TRANSFORM_TEX(v.texcoord, _BumpMap);

	float3 WorldPos = mul(unity_ObjectToWorld, v.position).xyz;
	fixed3 WorldTan = UnityObjectToWorldDir(v.tangent.xyz);
	fixed3 WorldNormal = UnityObjectToWorldNormal(v.normal);
	fixed3 WorldBinormal = cross(WorldNormal, WorldTan) * v.tangent.w;// w分量控制方向

	o.tanx = float4(WorldTan.x, WorldBinormal.x, WorldNormal.x, WorldPos.x);
	o.tany = float4(WorldTan.y, WorldBinormal.y, WorldNormal.y, WorldPos.y);
	o.tanz = float4(WorldTan.z, WorldBinormal.z, WorldNormal.z, WorldPos.z);

	return o;
}
```

在顶点着色器中我们计算了世界空间下的顶点位置、切线、法线和副法线，与切线空间的计算并无差别。

剩下的将四个向量的xyz分别存储在相应的xyz中，这四列就是刚刚算出来的四个值。

③修改片元着色器，在世界空间下进行光照计算：

```c#
fixed4 frag(v2f i) :SV_Target
{
    //顶点在世界空间下的位置
	float3 worldPos = float3(i.tanx.w, i.tany.w, i.tanz.w);
	// 光照方向
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(worldPos));
	// 视角方向
	fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));

	// 计算法线，从切线空间变换到世界空间
	fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));
	bump.xy *= _BumpScale;
	bump.z = sqrt(1 - saturate(dot(bump.xy, bump.xy)));
	bump = normalize(half3(dot(i.tanx.xyz, bump), dot(i.tany.xyz, bump), dot(i.tanz.xyz, bump)));

	// 替代漫反射的纹理采样
	fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
	fixed3 diffuse = _LightColor0.rgb * albedo * saturate(dot(bump, worldLightDir));

	// 高光反射
	fixed3 halfDir = normalize(worldLightDir + worldViewDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(bump, halfDir)), _Gloss);

	return fixed4(ambient + diffuse + specular, 1.0);
}
```

世界空间下的光照和视角方向分别用Unity的内置函数UnityWorldSpaceLightDir和UnityWorldSpaceViewDir得到，再使用内置的UnpackNormal函数对法线纹理进行采样和解码，并使用_BumpScale对其进行缩放。

最后通过使用点乘实现矩阵的每一行与法线相乘得到。

做完之后，用过对比会发现，从视觉层面上讲两个空间下的计算光照几乎没有变化，但明显切线空间的光照计算更为方便。

#### 13.4.4 Unity中的法线纹理类型

在上述过程中我们都是将法线纹理的纹理类型标识为Normal Map的，然后使用Unity内置函数UnpackedNormal来得到正确的法线。那么我们为什么要这么设置呢？设置为Normal Map的时候究竟发生了什么呢？

让我们来看看在UnityCG.cginc中UnpackedNormal的源码：

```c#
inline fixed3 UnpackedNormalDXT5nm (fixed4 packednormal){
    fixed3 normal;
    normal.xy = packednormal.wy * 2 - 1;
    normal.z = sqrt(1 - saturate(dot(normal.xy, normal.xy)));
    return normal;
}

inline fixed3 UnpackedNormal (fixed4 packednormal){
    #if defined (UNITY_NO_DXT5nm)
        return packednormal.xyz * 2 - 1;
    #else 
        return UnpackedNormalDXT5nm(packednormal);
    #endif
}
```

其中DXT5nm是一种压缩格式，在这种格式的法线纹理中，纹素的a通道对应了法线的x分量，g通道对应了法线的y分量，而纹理的r和b通道则会被舍弃，法线的z分量可以由xy分量推导而得。

而法线就需要使用DXT5nm格式来进行压缩，正如对DXT5nm的介绍所言，能够减少法线纹理占用的内存空间。

当我们上传一张贴图并将其纹理类型设置为Normal Map后，会出现一个复选框：Create from Grayscale。正如其名：从灰度图中制造。制造啥？高度图。当勾选后会出现两个选项：Bumpiness和Filtering。

前者用于控制每次调整_BumpScale时的凹凸程度；后者决定用哪种方式来计算凹凸程度：Smooth--使生成的法线纹理比较平滑，Sharp会使用Sobel滤波器来生成法线。

简单说一下Sobel滤波器：

它是一种边缘检测时使用的滤波器，对于高度图中的每个像素，计算它与水平方向和竖直方向上的像素差，把这个差当成该点对应法线在xy方向上的位移，然后使用映射函数存储到法线纹理的r和g分量中。

### 13.5 渐变纹理

渐变纹理用来干什么呢？控制漫反射光照的结果。

在之前计算漫反射光照时，使用表面法线和光照方向的点击结果与材质的反射率相乘得到表面的漫反射光照。这样做相对死板，在1998年Valve公司的Gooch等人在名为A Non-Photorealistic Lighting Model For Automatic Technical Illustration的论文（百度文库有提供）中提出了cool-to-warm tones（冷到暖色调）的着色技术，用来得到一种插画风格的渲染技术。使用这种技术，可以保证物体的轮廓线相比于之前使用的传统漫反射光照更加明显，而且能够提供多种色调变化。很多卡通风格都使用了这种技术。

①在Properties中声明一个用来存储渐变纹理的纹理属性

```c#
Properties
{
	_Color ("Color Tint", color) = (1,1,1,1)
       _RampTex ("Ramp Tex", 2D) = "white" {}
	_Specular("Specular", Color) = (1,1,1,1)
	_Gloss("GLoss", Range(8.0, 256)) = 20
}
```

②定义顶点着色器的输入与输出：

```c#
struct a2v
{
	float4 vertex : POSITION;
	float3 normal : NORMAL;
	float4 texcoord : TEXCOORD0;
};

struct v2f
{
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
	float2 uv : TEXCOORD2;
};
```

注意各个变量的类型。

③定义顶点着色器：

```c#
v2f vert (a2v v)
{
    v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);

	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

	o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);

	return o;
}
```

使用内置函数TRANSFORM_TEX来计算经过平铺和偏移的纹理坐标

④最为关键的片元着色器

```c#
fixed4 frag (v2f i) : SV_Target
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
	            
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
	//半兰伯特光照模型
	fixed halfLambert = 0.5 * dot(worldNormal, worldLightDir) + 0.5;
	fixed3 diffuseColor = tex2D(_RampTex, fixed2(halfLambert, halfLambert)).rgb * _Color.rgb;

	fixed3 diffuse = _LightColor0.rgb * diffuseColor;

	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
	fixed3 haldDir = normalize(worldLightDir + viewDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, haldDir)), _Gloss);

	return fixed4(ambient + diffuse + specular, 1.0);
}
```

使用半兰伯特光照模型，通过对法线方向和光照方向的点积做一次0.5的缩放以及一个0.5的偏移。再使用halfLambert来构建纹理坐标并用这个纹理坐标对渐变纹理_RampTex进行采样。由于 _RampTex是一个一维纹理，纵轴方向颜色不变，只变横轴，所以纹理坐标的uv方向都使用halfLambert。再将材质颜色与从渐变纹理采样得到的颜色相乘，得到最终的漫反射颜色。接下来的高光反射和环境光就在多说了。

以上是对scene中的模型进行的渐变纹理处理，但对于渐变纹理本身的设置，需要在其中的Wrap Mode设为Clamp模式，以防止对纹理采样时由于浮点数精度而造成的问题。

### 13.6 遮罩纹理

什么是遮罩？它允许我们可以保护某些区域，使它们免于修改。

就比如：在高光反射中表面所有的像素都是使用了同样大小的高光强度和高光指数，但为了让模型表面的一些点反光强而一些点反光弱，就需要使用一张遮罩纹理来控制光照。

流程一般是：通过采样得到遮罩纹理的纹素值，然后使用其中几个通道的值来与某种表面属性进行相乘，这样，该通道的值为0时，可以保护表面不受该属性的影响。

整体的代码建立在”在切线空间下进行光照计算“的代码的基础上，区别只是：

①属性中要加入遮罩纹理及其scale的属性：

```c#
_SpecularMask("Specular Mask",2D) = "white"{}
_SpecularScale("Specular Scale",Float) = 1.0
```

②CG代码块中声明这两个属性的变量：

```c#
sampler2D _SpecularMask;
float _SpecularScale;
```

③剩下的改动就只在片元着色器中了，在计算高光反射之前，要先对_SpecularMask进行采样，再将采样结果与高光反射值叉乘

```c#
//遮罩纹理采样
fixed3 specularMask = tex2D(_SpecularMask, i.uv).r * _SpecularScale;

// 高光反射
fixed3 halfDir = normalize(tangViewDir + tangLightDir);
fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(tangentNormal, halfDir)), _Gloss) * specularMask;
```

