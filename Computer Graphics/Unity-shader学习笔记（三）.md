[TOC]



# Unity-shader学习笔记（三）

前面介绍了渲染流水线，那么这里就详细说说每一部分的编写。

## 9 顶点/片元着色器

### 9.1 基本结构

```c#
Shader "MyShaderName" {
    Properties{
        //属性
    }
    
    SubShader{
        //针对显卡A的SubShader
        Pass{
            //设置渲染状态和标签
            ......;
            //开始CG代码片段
            CGPROGRAM
            //该代码片段的编译指令，例如：
            #pragma vertex vert
            #pragma fragment frag
            //上面这两句一般是shader已经自建好了的
            //CG代码的编写处
            ......;
            ENDCG
            
            //其他设置
        }
        //其他需要的Pass
    }
    SubShader{
        //针对显卡B的SubShader
    }
    
    //上述SubShader都失败后用于回调的Unity Shader 
    Fallback "VertexLit"
}
```

根据上述内容，就应该发现，最重要的就是Pass语义块。接下来逐个解读每一部分：

**①**代码第一行的命名：这个命名非常重要，并不是单单说就直接命名，而是要以"a/b/.../z"的路径形式命名。命好名后，在Material的shader中可以进行选择：a是一级目录，z是最后的目录，也是最重要决定使用的shader。这样命名可以帮助我们更快速的shader。

**②**Properties属性：这里定义的属性是会直接显示在Material的面板上，例如：

```c#
Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}
```

上面这个例子就会在Material面板中增加一项名为"Texture"的属性，它有三个部分：Offset（指明使用贴图的起始位置）、Tilling（指明从offset位置处的大小区域，区域的取值范围一般为(-1,1)，超过的话部分会按比例生成新的区域拼接上原先的）、未使用贴图时的Material颜色（由"="后面的颜色决定，这里是白色）

（Offset和Tilling较多得用于制作帧序列动画）

**③SubShader**：shader在执行过程中回一次检测SubShader，直到找到一个可执行的。

**④**在Pass之前其实还有一个Tags：

```c#
Tags{"TagName1" = "Value1""TagName2"="Value2"}
```

这些Tags会被应用于当前的所有Pass，一共有如下几种Tag：

|          Queue           |                设置渲染队列顺序                |
| :----------------------: | :--------------------------------------------: |
|      **RenderType**      |                **区别渲染对象**                |
|   **DisableBatching**    |          **是否要保存模型空间的信息**          |
| **ForceNoShadowCasting** | **用于渲染半透明物体时是否接受其他物体的阴影** |
|   **IgnoreProjector**    |    **是否接受贴图投影的影响（非光照照成）**    |
|  **CanUseSpriteAtlas**   |                **是否启用图集**                |
|     **PreviewType**      | **指明材质面板将如何预览该材质（默认为球形）** |

主要介绍其中的Queue、RenderType和PreviewType：

**Queue：**（实际上Queue的每个option在内部都被解释为固定的数值索引，从上到下以此为：1000，2000，2450，3000，4000；其次天空盒在所有不透明和透明对象之间绘制。）

|   Background    |              最先被调用的渲染，通常用来渲染背景              |
| :-------------: | :----------------------------------------------------------: |
|  **Geometry**   |     **默认值，大多数物体使用此队列，用来渲染不透明物体**     |
| **Alpha Test**  | **透明度测试，在Geometry之后进行透明测试，透明物体会被放弃渲染** |
| **Transparent** | **在Geometry和Alpha Test之后，从后往前渲染。任何Alpha混合（即不写入深度缓冲的shader）都在这里** |
|   **Overlay**   |  **用于叠加效果，任何最后呈现的东西都在这里，例如镜头光晕**  |

**RenderType：**（这些类型名称其实是可以自定义的，但是需要自己区别场景中不同渲染对象使用的Shader的RenderType的类型名称不同，也就是说RenderType使用自定义的名称并不会对该Shader的使用和着色效果产生影响。）

|          Opaque           | 用于大多数着色器，告诉系统应该在渲染非透明物体时调用我们 |
| :-----------------------: | :------------------------------------------------------: |
|      **Transparent**      |                   **用于半透明着色器**                   |
|   **TransparentCutout**   |                    **蒙皮透明着色器**                    |
|      **Background**       |             **Skybox shaders. 天空盒着色器**             |
|        **Overlay**        |                **光晕着色器、闪光着色器**                |
|      **TreeOpaque**       |                   **地形引擎中的树皮**                   |
| **TreeTransparentCutout** |                   **地形引擎中的树叶**                   |
|     **TreeBillboard**     |                 **地形引擎中的广告牌树**                 |
|         **Grass**         |                    **地形引擎中的草**                    |
|    **GrassBillboard**     |                **地形引擎何中的广告牌草**                |

**PreviewType：**

|    SkyBox    |      材质展示为天空盒      |
| :----------: | :------------------------: |
|  **Plane**   |     **材质展示为平面**     |
|   **Cube**   |    **材质展示为正方体**    |
|  **Sphere**  | **材质展示为球体**（默认） |
| **Cylinder** |    **材质展示为圆柱体**    |

**⑤Pass**：我们从外到内的剖析Pass版块

最外面是CGPROGRAM和ENDCG，紧接着是：

```c#
#pragma vertex vert
#pragma fragment frag
//它们将告诉Unity哪个函数包含了顶点着色器的代码，哪个函数包含了片元着色器的代码
```

看看下面这个例子：

```C#
struct vertOut {
	float4 pos : SV_POSITION;
	float4 scrPos : TEXCOORD0;
};

vertOut vert(appdata_base v) {
	vertOut o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.scrPos = ComputeScreenPos(o.pos);
	return o;
}

fixed4 frag(vertOut i) : SV_Target{
	float2 wcoord = (i.scrPos.xy / i.scrPos.w);
	return fixed4(wcoord, 0.0, 1.0);
}
```

vert函数的输入v包括了这个顶点的位置、法线和纹理坐标（是一个appdata_base的对象），返回的是一个vertOut的对象，这个对象包含了它在裁剪空间中的坐标和第一纹理坐标，是通过CG/HLSL的语义：SV_POSOTION和TEXCOORD0（Unity内置结构支持到6）

frag函数的输入也是一个vertOut的对象，输出的是一个fixed4类型的变量，并用语义SV_Target进行限定。SV_Target的作用是告诉渲染器，把用户的输出颜色存入到一个渲染目标中去。片元着色器输出的颜色的每个分量范围在[0,1]，其中(0.0,0.0,0.0)表示黑色，(1.0,1.0,1.0)表示白色。

### 9.2 顶点着色器数据的传入

在上面介绍了整个shader的结构，那么我们开始看看一个shader是怎么传入数据的。

在第一次介绍顶点着色器的时候，提到过在UnityCG.cginc中，官方已经为我们建好了几种顶点结构体的输入：appdata_base、appdata_tan、appdata_full、appdata_img。

那我们自己怎么建一个自己需要的顶点结构体？仿照官方的。

```c#
struct a2v {
    float4 vertex : POSITION;//POSITION语义告诉Unity，用模型空间的顶点坐标填充certex向量
    float3 normal : NORMAL;//NORMAL语义告诉Unity，用模型空间的法线方向填充normal向量
    float4 texcoord : TEXCOORD0;//TEXCOORD0语义告诉Unity，用模型的第一套纹理坐标填充texcoord变量
    ......
};
```

就会发现对一个顶点结构体，语义非常重要，根据这些语义Unity就会知道从哪里读取数据、并把数据输出到哪里。这些语义有：

①模型到顶点着色器：

|        POSITION         |     顶点位置     |
| :---------------------: | :--------------: |
|       **NORMAL**        |   **顶点法线**   |
|       **TANGENT**       |   **顶点切线**   |
| **TEXCOORD0-TEXCOORDn** | **顶点纹理坐标** |
|        **COLOR**        |   **顶点颜色**   |

②顶点着色器到片段着色器

|  SV_POSITION  | 裁剪空间中的顶点坐标（也可以用POSITION，但最好使用SV_POSITION） |
| :-----------: | :----------------------------------------------------------: |
|  **COLOR0**   |                         **顶点颜色**                         |
|  **COLOR1**   |                         **顶点颜色**                         |
| **TEXCOORDn** |                         **纹理坐标**                         |

③片段着色器输出语义

SV_Target：输出值直接用于渲染

### 9.3 顶点着色器到片元着色器的通信

以一个实例来进行说明：

```c#
struct vertOut {
	float4 pos : SV_POSITION;
	float4 scrPos : TEXCOORD0;
};

vertOut vert(appdata_base v) {
	vertOut o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.scrPos = ComputeScreenPos(o.pos);
	return o;
}

fixed4 frag(vertOut i) : SV_Target{
	float2 wcoord = (i.scrPos.xy / i.scrPos.w);
	return fixed4(wcoord, 0.0, 1.0);
}
```

在这里我们可以看到顶点着色器的函数vert传入的是appdata_base的对象v，片元着色器传入的是vertOut的对象i。顶点着色器到片元着色器的通信就是通过vertOut这个结构体来实现的。

通过将vert的类型声明为vertOut类型，再在vert中i定义verOut的对象，将传入的点的信息进行提取处理后输出给这个对象。

也就是说，我们要有一个结构体来定义顶点着色器的输入；一个结构体来定义顶点着色器的输出。

同时这里也再次说明官方的顶点着色器的输入/出，也即UnityCG.cginc中顶点着色器的输入/出：

|       名称       |   描述   |                     包含的变量                     |
| :--------------: | :------: | :------------------------------------------------: |
| **appdata_base** | **输入** |       **顶点位置、顶点法线、第一组纹理坐标**       |
| **appdata_tan**  | **输入** |  **顶点位置、顶点切线、顶点法线、第一组纹理坐标**  |
| **appdata_full** | **输入** | **顶点位置、顶点切线、顶点法线、最多六组纹理坐标** |
| **appdata_img**  | **输入** |            **顶点位置、第一组纹理坐标**            |
|   **v2f_img**    | **输出** |           **裁剪空间中的位置、纹理坐标**           |

### 9.4 Properties的使用

正如我们在Shader的结构中所介绍的那样，Properties中的内容会直接显示在该材质的面板中，我们可以在shader外面进行调控。我们也知道，  在编写Unity的脚本时，我们可以在脚本中对物件身上的属性进行调用，shader也是，可以调用Properties中的属性内容。例如：

```c#
Properties{
    _Color ("Color Tint", Color) = (1.0,1.0,1.0,1.0)
}

......

fixed4 _Color;
			
struct vertin 
{
	float4 vertex : POSITION;
	float4 normal : NORMAL;
	float4 texcoord : TEXCOORD0;
};

struct vertOut
{
	float4 pos : SV_POSITION;
	fixed3 color : COLOR0;
};

vertOut vert (vertin v)
{
	vertOut o;
	o.pos = UnityObjectToClipPos(v.vertex);
	o.color = v.normal*0.5 + fixed3(0.5, 0.5, 0.5);
    return o;
}

fixed4 frag (vertOut i) : SV_Target
{
	fixed3 c = i.color;
	c *= _Color.rgb;
	return fixed4(c, 1.0);
}
```

为了在CG代码中能够访问Properties中的内容，还需要在CG代码片段中提前定义一个新的变量，要注意，这个变量必须与Properties中的属性定义相匹配。

具体的与之匹配的变量类型如下：

|  Color、Vector   | float4、half4、fixed4  |
| :--------------: | :--------------------: |
| **Range、Float** | **float、half、fixed** |
|      **2D**      |     **sampler2D**      |
|     **Cube**     |    **samplerCube**     |
|      **3D**      |     **sampler3D**      |

## 10 关于ShaderLab、DX、OpenGL

因为接下来我们会聊到有关渲染的知识，而渲染在这三个平台的差异（主要是DX和OpenGL上的差异）很大，所以在渲染前，先说说DX和OpenGL。

### 10.1 坐标差异

之前其实也提到过DX和OpenGL在坐标上的差异：在水平方向上，两者的数值变化方向是相同的；在竖直方向上，两者是相反的。这是因为DX的屏幕坐标原点在左上角，OpenGL的屏幕坐标原点在左下角。这就会影响渲染纹理。

什么是渲染纹理？

我们在输出渲染结果的时候，不仅仅可以输出到屏幕上，还可以输出到不同的渲染目标，后者的渲染结果所保存的地方就是渲染纹理。当我们要将屏幕图像渲染到一张渲染纹理中时，如果不对坐标差异进行处理，就会出现纹理翻转的情况。所幸，Unity自身为我们解决了这个问题，就是为了能够在不同平台上达到一致性。

有一种特殊情况Unity是不会自动进行反转操作的---使用抗锯齿的同时使用渲染到纹理的技术。

在这种情况下，Unity会先渲染得到屏幕图像，再交由硬件进行抗锯齿处理，得到一张渲染纹理来供我们进行后续处理。此时，在DX平台下我们所得到的输入屏幕图像并不会被Unity翻转，就有两种情况：

①我们的屏幕特效只需要渲染一张图片时，由于在使用Graphics.Blit函数时，Unity已经对屏幕图像的采样坐标进行了处理，就不需要再去考虑纹理的翻转问题。

②如果我们同时需要处理多张渲染图象，例如需要同时处理屏幕图像和法线纹理，由于这些图像的在竖直方向的朝向就很可能不同，我们就需要在顶点着色器中翻转某些渲染纹理的纵坐标。一般需要添加这样的代码：

```c#
#if UNITUY_UV_STARTS_AT_TOP
if (_MainTex_TexelSize.y < 0)
    uv.y = 1 - uv.y;
#endif
```

其中，UNITUY_UV_STARTS_AT_TOP是用于判断当前平台是否是DX类型的平台，当在这类型平台下开启了抗锯齿后，主纹理的纹素大小在竖直方向上会变成负值。

### 10.2 shader语法的差异

这一点存在于不同平台以及统一平台的不同版本上。例如：

```c#
float4 v = float4(0.0);
```

在OpenGL中，这个代码是合法的，将得到的一个4个分量都是0.0的float4类型的变量赋值给v。

但在DX12上，这个就是非法的，必须写成：

```c#
float4 v = float4(0.0, 0.0, 0.0, 0.0);
```

除此之外还有两个语法上的差异：

①当在表面着色器的顶点函数（不是顶点着色器）中使用一个out修饰的参数时，若unity对out修饰的参数报“未初始化”的问题，则应该在顶点函数中使用UNITY_INITIALIZE_OUTPUT(Input, o)，使用UNITY_INITIALIZE_OUTPUT宏对输出结构体o进行初始化

③在OpenGL中允许使用tex2D函数对纹理进行采样，但在DX9及以上版本中，由于在顶点着色器阶段无法得到UV偏导，而tex2D函数又需要这样的骗到信息，也就导致DX9及以上版本不支持顶点阶段中的tex2D运算。如果在DX下我们确实需要在顶点着色器中访问纹理，就需要使用tex2Dlod函数来代替，同时还用使用#pragma target 3.0，因为tex2Dlod是Shader Model 3.0中的特性。也即：

```c#
#pragma target 3.0
tex2Dlod(tex, float4(uv, 0, 0));
```

### 10.3 shader的语义差异

其实这一点在前面介绍语义的时候也讲过，主要在SV_POSITION和SV_Target上：

①SV_POSITION与POSITION：使用这两个都可以用来描述顶点着色器输出的顶点位置，但SV_POSITION比POSITION更好，因为在索尼PS4、PS5或使用了细分着色器的情况下，使用的POSITION的Shader无法正常工作

②SV_Target与COLOR/0：SV_Target是用来描述片元着色器的输出颜色的，同上一个一样，在索尼PS4、PS5中，使用COLOR或者COLOR0的shader无法正常工作。

### 10.4 变量类型的选取

在上面的内容中我们也提到了shader常使用的三种数值类型，那么它们的精度呢？

|   类型    |                             精度                             |
| :-------: | :----------------------------------------------------------: |
| **float** |           **最高精度的浮点值，通常使用32位来存储**           |
| **half**  | **中等精度的浮点值，通常使用16位来存储，精度范围是-60000~+60000** |
| **fixed** | **最低精度的浮点值，通常使用11位来存储，精度范围是-2.0~+2.0** |

实际上，在现代PC的GPU中，它会将所有的计算都按照最高的浮点精度进行计算，也就是说float、half、fixed在这些平台上实际是等价的。但在移动端的GPU上，精度是非常确定的，不同精度的浮点值的运算速度也会不同，所以在进行移动端的开发要特别注意变量类型的选取。

同时为了验证shader的实际效果，应该选择真正的移动平台。

