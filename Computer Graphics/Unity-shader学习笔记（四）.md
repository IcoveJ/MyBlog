[TOC]



# Unity-shader学习笔记（四）

这里我们将会聊聊Unity中的基础光照及其有关知识

## 11 Unity中的基础光照

### 11.1 光源及辐照度

Unity中的光源想必大家都不陌生，所有的光都是从这个光源发射出来的。

在实时渲染中，我们通常将光源视为一个没有体积的点，并用l来表示它的方向。在光学中，我们也使用**辐照度（irradiance）**来量化光。

平行光与物体表面垂直，通过计算在垂直于l的单位面积上单位时间内穿过的能量得到其辐照度；

平行光与物体表面非垂直，假设入射光与物体表面法线的夹角为θ，平行光光线间的距离为d，那么辐照度就是与照射到物体表面时光线之间的距离d/cosθ成反比，即与cosθ成正比。而cosθ又可以使用光源方向l与表面法线n的点积来得到。

### 11.2 吸收与散射

光线由光源发出与物体相交后有两个结果：**散射（scattering）**和**吸收（absorption）**。

吸收不改变光线的方向，但要改变光线的密度和颜色；

散射只改变光线的方向，不会改变光线的密度和颜色，与吸收是相反的。

光线在物体表面经过散射后，有两种方向：①折射或投射-----光线将会散射到物体内部；②反射-----光线将会散射到物体外部。

对于不透明物体，通过折射进入物体内部的光线还会继续与物体内部的颗粒进行相交，其中一些光线最后会重新发射出物体表面，而另一些则会被物体吸收。对于那些从物体表面重新发射出去的光线，它们的方向分布和颜色与入射光线是不同的。

于是，我们在渲染物体模型时，在光照模型中会使用不同的部分来计算这两种散射方向：①**高光反射（specular）**用来表示物体表面是如何反射光线的；②**漫反射（diffuse）**表示有多少光线会被折射、吸收和散射出表面。我们根据入射光线的数量和方向可以计算出射光线的数量和方向，用**出射度**来描述它。辐照度与出射度之间的比值就是材质的漫反射和高光反射的属性。

### 11.3 着色

**着色（shading）**指的是根据材质属性（如漫反射属性等）、光源信息（如光源方向、辐照度等），使用一个灯饰去计算沿某个观察方向的出射度的过程，也称之为**光照模型（Lighting Model）**。

光照模型的目的包括描述不同材质物体的表面等。

## 12 标准光照模型

什么是标准光照模型？

它只关心直接光照，也就是那些直接从光源发射出来照到物体表面后的、经过物体表面一次反射就直接进入摄像机的光线。

它的基本方法是：把进入摄像机内的光线分为四部分，每个部分用一种方法来计算它的贡献度：

①**自发光（emissive）**部分：用于描述当给定一个方向时，一个表面本身会向该方向发射多少的辐射量。

②**高光反射（specular）**部分：用于描述当光线从光源照射到模型表面时，该表面会在完全镜面方向散射多少辐射量。

③**漫反射（diffuse）**部分：用于描述当光线从光源照射到模型表面时，该表面会向每个方向散射多少辐射量。

④**环境光（ambient）**部分：用于描述其他所有的间接光照。

### 12.1 环境光

间接光照指的是光线通常会在多个物体之间反射，最后进入摄像机。通俗的说，进入摄像机的光线经过了不止一次物体反射。

在计算时，环境光变量通常是一个全局变量，意思就是说场景中的所有物体都是用这个环境光。计算公式是：
$$
c_{ambient} = g_{ambient}
$$

### 12.2 自发光

这一部分主要算的就是直接由光源发射进入摄像机的光线的贡献度，直接使用的是该材质的自发光颜色：
$$
c_{emissive} = m_{emissive}
$$
但要注意，尽管我们称之为自发光，但它并不能照亮周围模型的表面，也就是说这个物体并不会被视为一个光源。

### 12.3 漫反射

主要用于对那些被物体表面随机散射到各个方向的辐射度进行建模的。在漫反射中，视角的位置是不重要的，因为反射完全是随机的，因此可以认为在任何反射方向上的分布都是一样的，这也就说明，入射角度的光线很重要。

漫反射我们会使用兰伯特定律（Lambert's law）：反射光线的强度与表面法线和光源方向之间夹角的余弦值称正比。那么漫反射部分的计算就是：
$$
c_{diffuse} = (c_{light}\cdot m_{diffuse})max(0,n\cdot I)\\
其中，n是表面法线，I是指光源的单位矢量，m_{diffuse}是材质的漫反射颜色，c_{light}是光源颜色
$$
在计算法线点乘光源方向时，其结果很可能是负数，所以我们此时就需要使用max()函数，人为将可能产生的负数结果截取为0，这样就能防止物体被从后面来的光源照亮。

### 12.4 高光反射

这里的高光反射与真实世界中的高光反射现象不完全符合，它只是一种经验模型。可用于计算那些沿着完全镜面反射方向被反射的光线，可以让物体看起来更有光泽。

计算高光反射需要的信息有很多，比如：表面法线、视觉方向、光源方向、反射方向等。

对这四个矢量，我们只要知道其中三个就能求得剩下那个。例如已知表面法线n、视角方向v、光源方向I，求反射方向r，其计算公式为：
$$
r = 2(n \cdot I)n -I
$$
接着我们就能利用Phong模型来计算高光反射：
$$
c_{specular} = (c_{light} \cdot m_{specular})max(0, v \cdot r)^{m_{gloss}}\\
其中m_{gloss}是材质的光泽度，也被称为反光度，它用于控制高光区域的亮点有多宽，m_{gloss}越大，亮点就越小。\\
m_{specular}是材质的高光反射颜色，用于控制该材质对于高光反射的强度和颜色。\\
c_{light}则是光源的颜色和强度
$$
另有学者Blinn提出了一个简单的修改方法来得到类似的效果：避免计算反射方向。

他引入了一个新的适量：h，是通过对v合I去平均后再归一化得到的，即：
$$
h = \frac{v+I}{|v+I|}
$$
然后，Blinn模型的计算公式是：
$$
c_{specular} = (c_{light} \cdot m_{specular})max(0, n \cdot h)
$$
这两者的区别在于：当v和I都是确定的时候，Blinn模型会快于Phong模型；反之Phong会快于Blinn。同时，这两种模型都是经验模型。

### 12.5 在何处计算上述四个部分

我们有两个选择：①在片元着色器中计算，也被称为**逐像素光照（per-pixel lighting）**；②在顶点着色器中计算，也被称为**逐顶点光照（per-vertex lighting）**。

在逐像素光照中，我们以每个像素为基础，获得它的法线（可以来自于法线纹理，也可以来自对顶点法线的插值），然后进行光照模型的计算。这种插值技术被称为**Phong着色（Phong shading）**，也被称为Phong插值或法线插值着色技术。

在逐顶点光照中（也被称为**高洛德着色（Gouraud shading）**）。我们是在每个顶点上计算光照，然后在渲染图元内部进行线性插值，最后输出成像素颜色。

大家应该都很清楚，顶点数目远少于像素数目，所以当在光照模型中有非线性计算时，例如高光反射的计算，著顶点光照就会出错。同时，由于逐顶点光照是在渲染图元内部对顶点进行的插值，这就会导致渲染图元内部的颜色总是暗于顶点处的最高颜色值，在一些情况下就会产生明显的棱角现象。

### 12.6 环境光和自发光的计算

在Unity中，场景中的环境光可以在Window->Lighting->Ambient Source/Ambient Color/Ambient Intensity中进行控制（在Unity5以上的版本中，是在Window->Rendering->Linghting Settings中进行设置）。在Shader中，可以通过Unity内置变量UNITY_LIGHTMODEL_AMBIENT来得到环境光的颜色和强度信息。

### 12.7 漫反射光照模型

#### 12.7.1 逐顶点光照

首先，我们已经知道了基本光照模型中漫反射部分的计算公式：
$$
c_{diffuse} = (c_{light}\cdot m_{diffuse})max(0,n\cdot I)
$$
我们一共需要四个参数：入射光线的颜色和强度、材质的漫反射系数、表面法线、光源方向。

①先在Properties中声明一个Color类型的属性：

```c#
Properties{
    _Diffuse ("Diffuse", Color) = (1,1,1,1)
}
```

②再在Pass语义块中指明该Pass的光照模式：

```C#
SubShader {
    Pass{
        Tags{"LightMode" = "ForwarBase"}
    }
}
```

之前在聊Tags的时候忘记说了，在Pass语义块中也是能够写Tags的，作用是告诉渲染引擎如何来渲染。就像上述中，它是告诉引擎去正向渲染。在后面我们再来细说！

③因为是渲染的光线，会用到_LightColor0，所以要使用Lighting.cginc

④在Properties语义块中需要定义一个与该属性相匹配的变量

```c#
fixed4 _Diffuse;
```

这样我们就能得到漫反射公式中需要的参数之一-----材质的漫反射属性。

⑤之后我们就能够定义定义顶点着色器输入输出结构体了

```c#
struct a2v
{
	float4 vertex : POSITION;
	float3 normal : NORMAL;
};

struct v2f
{
	float4 pos : SV_POSITION;
	fixed3 color : COLOR;//这里也可以使用float4 texcoord : TEXCOORD0
};
```

⑥接下来就是顶点着色器了，因为只实现逐顶点的漫反射光照，因此漫反射的所有计算都在顶点着色器中

```c#
v2f vert (a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
    
	fixed3 worldNormal = normalize(mul(v.normal,(float3x3)unity_WorldToObject));
	fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));//
	o.color = ambient + diffuse;
	return o;
}
```

我们之前重复过很多次，顶点着色器的任务就是将顶点位置从模型空间转换到裁剪空间，所以使用

UnityObjectToClipPos()函数（因为笔者使用的是Unity的2018版本，如是5.x版本，则是使用mul()函数和UNITY_MATRIX_MVP参数）。

然后我们使用Unity的内置变量UNITY_LIGHTMODEL_AMBIENT得到环境光部分。

 接下来才是真正的漫反射计算阶段：目前我们已经知道了材质的漫反射颜色_Diffuse和顶点法线v.normal，也就是说我们还需要去求解光源的颜色、强度信息以及光源的方向。

光源的颜色、强度信息可以通过unity的内置变量_LightMode()来访问获得（注意，想要得到正确的值需要定义合适的LightMode标签）；

光源方向可以通过_WorldSpaceLightPos()来访问获得，但要注意，我们这儿只是使用了一个光源，所以使用该函数不会出现任何错误；若场景中含有多个光源时，直接使用该函数会得到错误的数据。在Unity中还有很多内置函数可以实现这个，将在后面进行说明。

最后是表面法线和光源方向之间的点积，这两个值必须要在同一坐标系下求得的点积才有作用这里我们选择的是世界坐标空间。这里要注意一下mul()函数参数的位置，参数位置不同，得到的结果也不同。

⑦最后通过片元着色器将最终的顶点颜色输出。

```c#
fixed4 frag (v2f i) : SV_Target
{
	return fixed4(i.color, 1.0);
}
```

有细心的朋友应该会发现，在时间渲染出的模型中，当我们将光源设置为绕着模型自动旋转时，在亮暗交接处会有一些锯齿，这也是逐顶点光照可能出现的一些视觉问题，为了解决这些问题，可以使用逐像素的漫反射光照。

#### 12.7.2 逐像素光照 

逐像素光照的代码与逐顶点光照的代码大体上相同，主要区别在于将漫反射光照模型的计算从顶点着色器转移到了片元着色器。

①修改顶点着色器的输出结构v2f

```c#
struct v2f
{
	float4 pos : SV_POSITION;
	fixed3 worldNormal : TEXCOORD0;
};
```

②修改顶点着色器，删除漫反射计算有关内容

```c#
v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
	return o;
}
```

③修改片元着色器，加入漫反射计算有关内容

```c#
fixed4 frag(v2f i) : SV_Target
{
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
	fixed3 color = ambient + diffuse;
	return fixed4(color, 1.0);
}
```

尽管在这样设置了逐像素的光照模型后，其与逐顶点光照模型还是有一个缺陷：但我们将视线移至物体背面后就会发现更像是一个2D平面而不是一个3D的模型了（你也会发现两种光照模型下背面颜色的不同）。于是就提出了一种改善技术-----**半兰伯特光照模型（Half Lambert）**。

广义的半兰伯特光照模型的公式如下：
$$
c_{diffuse} = (c_{light}\cdot m_{diffuse})(\alpha(n\cdot I)+\beta)
$$
半兰伯特光照模型就没有再使用max操作来避免点积为负的情况，而是引入了一个缩放系数α和一个偏移系数β，绝大多数情况下，α和β的大小都是0.5，也即：
$$
c_{diffuse} = (c_{light}\cdot m_{diffuse})(0.5*(n\cdot I)+0.5)
$$
这样就将n•I的结果范围从[-1,1]映射到[0,1]内，同时将背光面的点积结果从原兰伯特光照模型的0值，变换成了不同的值，不在固定为0。

半兰伯特光照模型是没有任何物理依据的，它仅仅是一个视觉加强技术。

渲染出来的效果就是背光区的阴影是一个渐变的过程。

### 12.8 高光反射光照模型

首先，我们已经知道了基本光照模型中高光反射部分的计算公式：
$$
\begin{align}
c_{specular} &= (c_{light} \cdot m_{specular})max(0, v \cdot r)^{m_{gloss}} & r = 2(n \cdot I)n - I
\end{align}
$$
一共需要四个参数：入射光线的颜色和强度、材质的高光反射系数、视角方向、反射方向。

反射方向的计算，CG提供了内置函数reflct(i,n)，i为入射方向；n为法线方向。

#### 12.8.1 逐顶点光照

①在Properties语义块中声明控制高光反射的属性

```c#
Properties{
    _Diffuse ("Diffuse", Color) = (1,1,1,1)
    _Specular ("Specular", Color) = (1,1,1,1)//控制材质的高光反射颜色
    _Gloss ("Gloss", range(8.0, 256)) = 20//控制高光区域的大小
}
```

②Pass语义块中的次要部分就不再谈论，直接讨论最重要顶点着色器的输入输出结构体和顶点着色器的内容。

```c#
struct a2v
{
	float4 vertex : POSITION;
	float3 normal : NORMAL;
};

struct v2f
{
	float4 pos : SV_POSITION;
	fixed3 color : COLOR;
};

v2f vert (a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
    
    
	fixed3 worldLightnormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
	fixed3 diffuse = _LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldLightnormal, worldLightDir));
    //以上部分为漫反射的计算
    
	fixed3 reflectDir = normalize(reflect(-worldLightDir, worldLightnormal));
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
	fixed3 specular = _LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir, viewDir)), _Gloss);
	o.color = ambient + diffuse + specular;
	return o;
}
```

漫反射的计算与12.7.1中是一样的，就不细说了。对于高光反射部分，首先计算是反射方向的矢量reflectDir，由于reflect函数的入射方向是由光源指向交点处，所以计算时要对worldLightDir进行取反

推导过程如下：
$$
设光源为点B，模型表面的点为A，摄像机的位置为C，B在x轴上的投影点为E，向量AB与法线间的夹角为\theta，理想化|AB| = |AC|\\
在我们通常的书写习惯中，光线是B\rightarrow A\rightarrow C，也就是说yizhi入射方向矢量为\vec{BA},法线矢量为\vec{N}(单位矢量)，反射方向矢量为\vec{AC}\\
根据上述前提，我们延长BA至D使|BA| = |AD|，那么根据三角形法则就有：\\
\begin{align}
\vec{AC} &= \vec{AD} + \vec{DC}\\
&= -\vec{AB}+2\vec{ED}\\
\vec{ED} &= \vec{N}*|ED|\\
&= \vec{N}*|AB|cos\theta\\
&= \vec{N}*\vec{N}\cdot \vec{AB}\\
所以\vec{AC} &= -\vec{AB}+2（\vec{N}*\vec{N}\cdot \vec{AB}）
\end{align}
$$
去翻的主要原因就是角度θ，因为我们一般是把入射光线与法线的夹角称为入射角θ，注意这里是光线，不是矢量，而我们进行的所有计算都是以矢量在进行计算，若不取反，那么角度值就会增加90°。

接下来获取视角方向：分别获得谁先那估计和点在世界空间下的坐标，再前者减后者就得到视角方向。

四个参数获取完毕，就再根据公式就能够得到高光反射的光照部分。

真正用这个进行渲染过的朋友都知道渲染出来的结果我们看起来很糟糕，这是因为高光反射部分的计算是非线性的，而顶点着色器中的插值过程使线性的，这两者就冲突了，就会导致视觉问题。处理方法就是使用逐像素。

#### 12.8.2 逐像素光照

同漫反射一样，高光反射的逐像素光照与逐顶点光照相比，主要区别就在于将光照的计算放在了片元着色器中。

①先修改顶点着色器的输出结构体

```c#
struct v2f
{
	float4 pos : SV_POSITION;
	fixed3 worldNormal : TEXCOORD0;
	fixed3 worldPos : TEXCOORD1;
};
```

②顶点着色器下就只需要计算世界空间下的法线方向和顶点坐标，并把它们传递给片元着色器中即可：

```c#
v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
	return o;
}
```

③在片元着色器中进行关键计算

```c#
fixed4 frag(v2f i) : SV_Target
{
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

	fixed3 diffuse = _LightColor0.rgb*_Diffuse.rgb*saturate(dot(worldNormal, worldLightDir));
				
	fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));

	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
				
	fixed3 specular = _LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir, viewDir)), _Gloss);
	return fixed4(ambient + diffuse + specular, 1.0);
}
```

这样渲染后的结构就明显会比逐顶点光照的效果要好，更加的平滑。

#### 12.8.3 Blinn-Phong光照模型

通过在漫反射中的介绍我们知道还有一种叫做Blinn-Phong的模型的渲染效果会更好：
$$
\begin{align}
c_{specular} &= (c_{light} \cdot m_{specular})max(0, n \cdot h)\\
h &= \frac{v+I}{|v+I|}
\end{align}
$$
需要在片元着色器中额外写一句代码并更改计算高光反射时的参数：

```c#
fixed3 halfDir = normalize(worldLightDir + viewDir);
fixed3 specular = _LightColor0.rgb*_Specular.rgb*pow(saturate(dot(reflectDir, halfDir)), _Gloss);
```

然后就会发现Blinn-Phong模型的效果也要比逐像素光照更好一点。

### 12.9 关于Unity的内置函数

聊完上述的内容，你就会发现在我们求解光源方向和视角方向时，使用了大量的内置函数：

normalize(_WorldSpaceLightPos0.xyz)来得到光源方向（这种只适合平行光）；

normalize(_WorldSpaceCameraPos.xyz - i.worldPosition.xyz)来得到视角方向。

其余相似的内置函数还有：

|                       函数名                       |                             描述                             |
| :------------------------------------------------: | :----------------------------------------------------------: |
|       **float3 WorldSpaceViewDir(float4 v)**       | **输入模型空间中的顶点位置，返回世界空间中从该点到摄像机的观察方向。内部实现使用了UnityWorldSpaceViewDir函数** |
|    **float3 UnityWorldSpaceViewDir(float4 v)**     | **输入世界空间中的顶点位置，返回世界空间中从该点到摄像机的观察方向** |
|        **float3 ObjSpaceViewDir(float4 v)**        | **输入模型空间中的顶点位置，返回模型空间中从该点到摄像机的观察方向** |
|      **float3 WorldSpaceLightDir(float4 v)**       | **仅用于前向渲染。输入模型空间中的顶点位置，返回模型空间中从该点到光源的光照方向，内部实现使用了UnityWorldSpaceLightDir函数，没有被归一化。** |
|    **float3 UnityWorldSpaceLightDir(float4 v)**    | **仅用于前向渲染。输入世界空间中的顶点位置，返回模型空间中从该点到光源的光照方向，没有被归一化** |
|       **float3 ObjSpaceLightDir(float4 v)**        | **仅用于前向渲染。输入模型空间中的顶点位置，返回模型空间中从该点到光源的光照方向，没有被归一化** |
| **float3 UnityObjectToWorldNormal(float4 normal)** |           **把法线矢量从模型空间变换到世界空间中**           |
|    **float3 UnityObjectToWorldDir(float4 dir)**    |            **把方向矢量从模型空间变换到世界空间**            |
|    **float3 UnityWorldToObjectDir(float4 dir)**    |            **把方向矢量从世界空间变换到模型空间**            |

注意，在使用这些函数的结果使一定要对其进行归一化。

