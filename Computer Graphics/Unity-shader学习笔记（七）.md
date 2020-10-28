[TOC]



# Unity-shader学习笔记（七）

## 15  更复杂的光照

又是光照，只是比之前更复杂了，体现在哪儿呢？需要处理的光源数目更多、类型也更复杂，例如：如何处理光源和聚光灯，如何处理光照的衰减等等，这些都是接下来会聊到的。

### 15.1  Unity的渲染路径

渲染路径决定了光照是如何应用到UnityShader中的，所以当我们要使用光源时，我们要为每个Pass指定它使用的渲染光源。

渲染路径可以在Camera的Rendering Pass中进行选择，一共有四种（笔者使用的是2018.4.17f1）：

Forward、Deferred、Legacy Vertex Lit、Legacy Deferred(Light prepass)。

①Deferred：延迟渲染，具有最高照明、阴影保真度的渲染路径，如果场景中使用了许多的实时灯光，这是最适合的，但需要一定的硬件支持；

②Forward：前向渲染（或者正向渲染）。最传统的渲染路径，支持所有典型的Unity图形功能；

③Legacy Vertex Lit：正向渲染路径的一个子集，具有最低照明保真度的渲染路径，并且不支持实时阴影；

④Legacy Deferred(Light prepass)：传统延迟（轻量级预备），类似于延迟渲染，只是使用的技术与权衡皆不同

下面我们主要介绍前两种。

#### 15.1.1 前向渲染路径

我们将聊聊前向渲染路径的原理以及在Unity中的实现和注意事项。

##### 15.1.1.1 前向渲染路径的原理

每进行一次完整的前向渲染，我们需要渲染该对象的渲染图元，并计算颜色缓冲区和深度缓冲区的信息。我们利用深度缓冲区来决定一个片元是否可见，如果可见，就更新颜色缓冲区中的颜色值。

```c#
Pass{
    for (each primitive in this model){
        for (each fragment covered by this primitive){
            if (failed in depth test){
                discard;
            }else{
                float4 color = Shading (materialInfo, pos, normal, lightDir, viewDir);
                writeFrameBuffer (fragment, color);
            }
        } 
    }
}
```

对于每个像素光源，我们都要进行上面一次完整的渲染流程，同时对每个Pass也要计算一个逐像素光源的光照，假设场景中有N个物体，每个物体受M个光源的影响，那么渲染整个场景一共需要N*M个Pass，如果有大量的逐像素光照，那么需要执行的Pass数目也会很大，因此渲染引擎通常会限制每个物体的逐像素光照的数目。

##### 15.1.1.2 Unity中的前向渲染

一个Pass不仅仅可以用来计算逐像素光照，也可以用来计算逐顶点光照，这些都取决于光照计算所出流水线阶段以及计算时使用的数学模型。当我们渲染一个物体时，Unity会计算哪些光源照亮了它，以及这些光源照亮该物体的方式。

在Unity中，前向渲染路径有三种处理光照的方式：逐顶点处理、逐像素处理、球谐函数处理（SH处理）。决定一个光源的的处理模式取决于它的类型和渲染模式。

光源类型有平行光源和其它类型的；渲染模式是指该光源是否是重要的（Important）。如果我们将一个光源的模式设置为Important，意味着提高它的优先级并按照逐像素光源来处理。

Unity是怎么对光源的重要度进行排序的呢？

①场景中最亮的平行光总是逐像素处理的；

②渲染模式被设置成Not Important的光源会按逐顶点或者SH处理，其中最多只有四个光源按逐顶点的方式处理；

③渲染模式被设置成Important的光源会按照逐像素处理；

④如果根据上述规则得到的逐像素光源数量小于Quality Setting中的逐像素光源数量（Pixel Light Count）,会有更多的光源以逐像素的方式进行渲染。

在哪儿进行计算呢？Pass中。在前向渲染中有两种Pass，一种是Base Pass，另一种是Additional Pass

在Base Pass中：

```c#
Tags {"LightMode" = "ForwardBase"}
#pragma multi_compile_fwdbase
    ......//一个逐像素的平行光以及多有逐顶点和SH光源
```

可实现的光照效果有：

光照纹理、环境光、自发光、平行光阴影。

在Additional Pass中：

```c#
Tags {"LightMode" = "AdditionalBase"}
Blend One One
#pragma multi_compile_fwdadd
    ......//其他影响该物体的逐像素光源，每个光源执行一次Pass
```

可实现的光照效果有：

默认情况下是不支持阴影的，但可以通过语句：#pragma multi_compile_fwdadd_fullshadow来开启阴影。

##### 15.1.1.3 内置的光照变量和函数

根据物品们使用的渲染路径（LightMode的取值），Unity会将不同的光照变量传递给Shader。

前向渲染的的光照变量有：

|         _LightColor0         |   float4   |                 该Pass处理的逐像素光源的颜色                 |
| :--------------------------: | :--------: | :----------------------------------------------------------: |
|   **_WorldSpaceLightPos0**   | **float4** | **_WorldSpaceLightPos0.xyz是该Pass处理的逐像素光源的位置，至于其w分量，若是平行光则为0，其他光则为1** |
|      **_LightMatrix0**       | **float4** | **从世界空间到光源空间的变换矩阵，可用于采样cookie和光强衰减纹理** |
| **unity_ 4LightPosX0/Y0/Z0** | **float4** |   **仅用于Base Pass。前4个重要的点光源在世界空间中的位置**   |
|   **unity_ 4LightAtten0**    | **float4** |   **仅用于Base Pass。存储了前4个非重要的点光源的衰减因子**   |
|    **unity_ LightColor**     | **half4**  |     **仅用于Base Pass。存储了前4个非重要的点光源的颜色**     |

前向渲染的的光照函数有：

|     float3 WorldSpaceLightDir (float4 v)      | **仅可用于前向渲染中。输入一个模型空间中的顶点位置，返回世界空间中 从该点到光源的光照方向。内部实现使用了UnityWorldSpaceLightDir函数。没有被归一化** |
| :-------------------------------------------: | :----------------------------------------------------------: |
| **float3 UnityWorldSpaceLightDir (float4 v)** | **仅可用于前向渲染中。输入一个世界空间中的顶点位置，返回世界空间中 从该点到光源的光照方向。没有被归一化** |
|    **float3 ObjSpaceLightDir (float4 v)**     | **仅可用于前向渲染中。输入一个模型空间中的顶点位置，返回模型空间中 从该点到光源的光照方向。没有被归一化** |
|      **float3 Shade4PointLights (...)**       | **仅可用于前向渲染中。计算四个点光源的光照，它的参数是已经打包进矢量的光照数据，通常就是上表中的内置变量，如unity_ 4LightPosX0、unity_ 4LightPosY0、unity_ 4LightPosZ0、unity_ LightColor和unity_ 4LightAtten0等。前向渲染通常会使用这个函数来计算逐顶点光照** |

#### 15.1.2 延迟渲染路径

为什么要有延迟渲染？因为当场景中包含大量的实时光源时，前向渲染的性能会急速下降。具体体现在：

当我们在一块区域放置了多个光源时，这些光源影响的区域互相叠加，为了得到最终的光照效果，就需要对该区域内的每个物体执行多个Pass来计算不同光源对该物体的光照结果，然后再颜色缓冲区中进行混合得到最终的光照，这就会导致没执行一个Pass都要渲染一遍物体，重复计算。

在延迟渲染中，除了会使用到前向渲染中使用的颜色缓冲和深度缓冲外，还会利用额外的缓冲区，我们称之为G缓冲，它存储了我们所关心的表面法线、位置、材质属性等信息。

##### 15.1.2.1 延迟渲染路径的原理

延迟渲染主要包含两个Pass：在第一个Pass中，不进行任何的光照，而是仅仅计算哪些片元是可见的，这主要是通过深度缓冲技术来实现的，当发现一个片元是可见的，我们就将它的相关信息存储到G缓冲区中；在第二个Pass中，我们利用G缓冲区的各个片元信息（表面法线、视角方向、漫反射系数等）来进行真正的光照计算。

```c#
Pass 1{
    for (each primitive in this model){
        for (each fragment covered bu this primitive){
            if (failed in depth test){
                discard;
            }else{
           		writeGBuffer (materialInfo, pos, normal, lightDir, viewDir);
        	}
        }
    }
}
//利用G-Buffer中的信息进行真正的光照计算
Pass 2{
    for (each pixel in the screen){
        if (the pixel is valid){
            readGBuffer(pixel, materialInfo, pos, normal, lightDir, viewDir);
            float4 color = Shading(materialInfo, pos, normal, lightDir, viewDir);
            writeFrameBuffer(pixel, color);
        }
    }
}
```

延迟渲染的效率不依赖场景的复杂度，而是和我们使用的屏幕空间的大小有关，因为，我们需要的信息都存储在缓冲区中，可以将其理解为一张张2D图像，我们的计算实际上就是在这些图像空间中进行的。

##### 15.1.2.2 Unity中的延迟渲染

第一个Pass用于渲染G缓冲。在这个Pass中，我们会将物体的漫反射颜色、高光反射颜色、平滑度、法线、自发光和深度等信息渲染到屏幕空间的G缓冲区中。对于每个物体而言，这个Pass仅执行一次。

第二个Pass用于计算真正的光照模型。使用的是上一个Pass中渲染的数据来计算最终的光照颜色，在存储到帧缓冲中。

默认的G缓冲区包含了以下几个渲染纹理(Render Texture，RT)：

①RT0：格式是ARGB32，RGB通道用于存储漫反射颜色，A通道没有被使用；

②RT1：格式是ARGB32，RGB通道用于存储高光反射颜色，A通道用于高光反射的指数部分；

③RT2：格式是ARGB2101010，RGB通道用于存储法线，A通道没有被使用；

④RT3：格式是ARGB32（非HDR）或ARGBHalf（HDR），用于存储自发光+lightmap+反射探针（reflection probes）。

⑤深度缓冲和模板缓冲

当在第二个Pass中计算光照时，默认情况下可以使用Unity内置的Standard光照模型。若想使用其他的光照模型，就需要替换原有的Internal-DefferredShading.shader文件。

##### 15.1.2.3 内置变量和函数

这些变量的使用需要头文件UnityDeferredLibrary.cginc的支持

|    _LightColor    |    float4    |                           光源颜色                           |
| :---------------: | :----------: | :----------------------------------------------------------: |
| **_LightMatrix0** | **float4*4** | **从世界空间到光源空间的变换矩阵，可用于采样cookie和光强衰减纹理** |

### 15.2 Unity的光源类型

Unity一共支持四种光源类型：平行光、点光源、聚光灯、面光源（仅仅在烘培时才会发挥作用）。

其中面光源我们这里不进行描述。

#### 15.2.1 不同的光源会产生怎样的影响

我们将从光源的位置、方向、颜色、强度和衰减这5个属性来聊。

##### 15.2.1.1 平行光

前面用得太多了，就不重复了。

##### 15.2.1.2 点光源

顾名思义，所有光都来自于一个点，它所照亮的空间是有限的，由空间的一个球体所定义。

在Scene中还需要开启光照才能看到点光源是如何影响场景中的物体的。

球体的半径可以由属性面板中的Range来调整，或者直接在Scene视图中进行拖拉放缩；点光源的方向是由其位置减去某点的位置来得到；点光源的颜色和强度可以在Light组件面板中调整。同时，点光源也是会衰减的，随着物体逐渐远离点光源，它接受到的光照强度也会逐渐较小，点光源球心处的光照强度最强，球体边界处的最弱，值为0.其中间的衰减值可以由一个函数来定义。

##### 15.2.1.3 聚光灯

聚光灯是最复杂的一种，它照亮的空间同样有限，但不再是简单的球体，而是由空间中的一块锥形区域定义。可以表示为从一特定位置出发、向特定方向延伸的光。只有一点属性与其他的不同，就是从光源中心向光源边界的光源衰减计算公式更加的复杂，因为要判断一个点是否在锥体中。

#### 15.2.2 在前向渲染中处理不同的光源

在使用前向渲染的基础上，在Unity Shader中访问光源的5个属性：位置、方向、颜色、强度以及光照衰减。

①定义第一个Pass——Base Pass

```c#
Pass{
    Tags {"LightMode" = "ForwardBase"}
    
    CGROGRAM
        
    #pragma multi_compile_fwdbase//确保我们在Shader中使用光照衰减等光照变量可以被正确赋值。
}
```

②在Base Pass的片元着色器中，首先计算场景中的环境光

```c#
fixed3 ambient = UNITY_LIGHTMODE_AMBIENT.xyz;
```

与之相同的还有物体的自发光，都只需要计算一次就好了。

③然后我们在Base Pass中计算场景最重要的平行光。当场景中含有多个平行光时，Unity会选择最亮的平行光传递给Base Pass进行逐像素处理，其他平行光会按照逐顶点或在Additional Pass中按逐像素的方式处理。如果场景中没有任何的光，那么Base Pass会当成全黑的光源处理。对于Base Pass而言，它处理的逐像素光源类型一定是平行光，可以使用_WorldSpaceLightPos0来得到这个平行光方向（位置对于平行光而言是没有任何意义的），使用 _LightColor0来得到它的颜色和强度（ _LightColor0已经是颜色和强度相乘后的结果），由于是平行光，所以我们认为是没有衰减的，因此我们直接令衰减值为1.0，代码为：

```c#
fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));

......;

fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(dot(worldNormal, halfDir), _Gloss);

......;

fixed atren = 1.0;

return fixed4(ambient + (diffuse + specular) * atten, 1.0);
```

④为场景中其他逐像素光源定义Additional Pass

```c#
Pass{
    Tags {"LightMode" = "ForwardAdd"}
    
    Blend One One
        
    #pragma multi_compile_fwadd
}
```

除之前所说的那些内容外，我们还使用Blend命令开启和设置了混合模式，这是因为我们希望Additional Pass计算得到的光照结果可以在帧缓存中与之前的光照结果进行叠加。

⑤通常来说，Additional Pass的光照处理与Base Pass的处理方式是一样的，略微的区别在于：Additional Pass中是没有Base Pass中环境光、自发光、逐顶点光照、SH光照的部分，并添加一些对不同光源类型的支持。我们通过使用#ifdef来进行不同光源类型的计算：

```c#
#ifdef USING_DIRECTIONAL_LIGHT
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
#else
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPosition.xyz);
#endif
```

⑥最后处理不同光源的衰减

```c#
#ifdef USING_DIRECTIONAL_LIGHT
    fixed3 atten = 1.0;
#else
    float3 lightCoord = mul(_LightMatrix0, float4(i.worldPosition, 1)).xyz;
    fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
#endif
```

### 15.3 Unity光照的衰减

在15.2.2中的第六步，你可能会奇怪为什么衰减会用tex2D函数来表示。

如果是平行光，我们当然可以定量的将其设定为1.0，如果是其他的光源类型，那么处理起来会更加的复杂，尽管可以使用诸如开方、出发等数学表达式来计算给定点相对于点光源和聚光灯的衰减，但计算量相对较大，因此Unity选择了使用一张纹理作为查找表（LUT），一再片元着色器中得到光源的衰减。我们首先得到在光源空间下的坐标，然后使用该坐标对衰减纹理进行采样得到衰减值。

这样做可以在一定程度上提升性能，而且得到的效果在大部分情况下都是良好的。

但这样做也是有弊端的：

①需要预处理得到采样纹理，而且纹理的大小也会影响衰减的精度；

②不直观，同时也不方便，因为将衰减值存储到查找表中，我们就无法使用其他数学公式来计算衰减值了。

#### 15.3.1 用于光照衰减的纹理

Unity在内部使用了一张名为_LightTexture0的纹理来计算光源衰减为了对 _LightTexture0纹理采样得到给定点到该光源的衰减值，我们首先需要得到该点在光源空间中的位置，这是通过 _LightMatrix0变换矩阵得到的。将这个矩阵与世界空间中的顶点坐标相乘，再将乘后的坐标的模的平方对衰减纹理进行采样，得到衰减值：

```c#
float3 lightCoord = mul(_LightMatrix0, float4(i.worldPosition, 1)).xyz;
fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
```

#### 15.3.2 使用数学公式计算衰减

由于基于纹理采样的光照衰减存在种种不利，所以有时候我们还是要使用数学公式来计算光照衰减。例如线性衰减的代码为：

```c#
float distance = length(_WorldSpaceLightPos0.xyz - i.worldPosition.xyz);
atten = 1.0 / distance;
```

当然，这种方式就需要很高超的数学功底了，尤其是聚光灯，需要根据它的朝向、张开角度等来进行计算。

### 15.4 Unity的阴影

#### 15.4.1 阴影是如何实现的

当一个光源发射一条光线遇到一个不透明的物体时，这条光线就不可以再继续照亮其他物体（暂时不考虑光线反射）。因此这个物体就会向它旁边的物体投射阴影，那些阴影区域的产生是因为光线无法到达这些区域。

在实时渲染中，我们最常使用的是一种名为Shadow Map的技术，它会首先把摄像机的位置放在与光源重合的位置上，那么场景中该光源的阴影区域就是那些摄像机看不到的地方。

在前向渲染中，如果场景中最重要的平行光开启了阴影，Unity就会为该光源计算它的阴影映射纹理，这张纹理的本质上也是一张深度图，它记录了从该光源的位置出发，能看到的场景中距离它最近的表面位置（深度信息）。

那么在计算阴影映射纹理时，我们如何判定距离它最近的表面位置？

第一种方法：先把摄像机放在光源位置上，然后按照正常的渲染流程，即调用Base Pass和Additional Pass来更新深度信息，得到阴影映射纹理，但这种方法会照成性能上的一定的浪费，因为我们仅仅只需要深度信息而已，但在Base Pass和Additional Pass中往往还计算了其他量。于是就有了第二种方法。

第二种方法：使用LightMode标签被设置为ShadowCaster的Pass。这个Pass的渲染目标不是帧缓存，而是阴影映射纹理。它首先会将摄像机放置在光源的位置上，然后调用该Pass，通过对顶点变换后得到光源空间下的位置，并据此来输出深度信息到阴影映射纹理中

总结性的说，一个物体接受来自其他物体的阴影，以及它向其他物体投射阴影是两个过程：

①如果我们想要一个物体接受来自其他物体的阴影，就必须在Shader中对阴影映射纹理继续采样（包括屏幕空间的阴影图），把采样结果和最后的光照结果相乘来产生阴影效果；

②如果我们想要一个物体向其他物体投射阴影，就必须把该物体加入到光源的阴影映射纹理的计算中，从而让其他物体在对阴影映射纹理采样时可以得到该物体的相关信息。再Unity中这个过程时通过为该物体执行LightMode为ShadowCaster的Pass来实现的。如果使用了屏幕空间的投影映射技术，Unity还会使用这个Pass产生一张摄像机的深度纹理。

#### 15.4.2 具体实现

##### 15.4.2.1 让物体产生投影

是否让一个物体产生阴影，是通过设置Mesh Renderer组件中的Cast Shadows和Receive Shadows属性来实现的。

如果开启了Cast Shadows属性，那么Unity就会将该物体加入到光源的阴影映射纹理的计算中，从而让其他物体在对阴影映射纹理进行采样时可以得到该物体的相关信息。

Receive Shadows则可以选择是否让物体接受来自其他物体的阴影，如果不开启这个，那么当我们调用Unity的内置宏和变量计算阴影时，这些宏通过判断物体没有开启接受阴影的功能，就不会在内部为我们计算阴影。

代码的使用依然是使用的前向渲染的代码。

##### 15.4.2.2 让物体接收投影

如果你试过。当在场景中加入两个正方体，按照上述方法，你会成功在平面上产生投影，但将一个正方体靠近另一个正方体时，无法在另一个正方体上产生投影。因为上述的代码和方法只是让物体能够产生阴影，尽管你开启了那两个属性，但依然没法接受到。

在上述代码的基础上，我们进行一些改进：

①在Base Pass中新加一个头文件：

```c#
#include "AutoLight.cginc"
```

计算阴影时所用的宏都是在这个文件中声明的

②在顶点着色器的输出结构体v2f中添加一个内置宏SHADOW_COORDS：

```c#
struct v2f
{
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
	SHADOW_COORDS(2)
};
```

这个宏的作用就是声明一个用于对阴影纹理采样的坐标，需要注意的是，这个宏的参数需要是下一个可用的插值寄存器的索引值。

③在顶点着色器返回之前添加另一个内置宏TRANSFER_SHADOW：

```c#
v2f vert(a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);

	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

	TRANSFER_SHADOW(o);

	return o;
}
```

这个宏用于计算上一步中声明的阴影纹理坐标

④在片元着色器在使用内置宏SHADOW_ATTENUATION来计算阴影值

```c#
fixed4 frag(v2f i) : SV_Target
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
	fixed3 halfDir = normalize(worldLightDir + viewDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

	fixed atten = 1.0;

	//计算阴影
	fixed shadow = SHADOW_ATTENUATION(i);

	return fixed4(ambient + (diffuse + specular) * atten * shadow, 1.0);
}
```

总体来说我们就是使用了SHADOW_COORDS、TRANSFER_SHADOW、SHADOW_ATTENUATION这三个宏来实现了物体对阴影的接收。需要注意的是，这些宏是使用的上下文的变量来进行计算，例如：TRANSFER_SHADOW会使用v.vertex或a.pos来计算坐标。因此，我们就必须保证自定义的变量名个这些宏中使用的变量名相匹配。也就是要保证：a2f结构体中的顶点坐标变量名必须是vertex，顶点着色器的输出结构体v2f必须命名为v，且v2f中的顶点位置变量必须命名为pos。

#### 15.4.3 统一管理光照衰减和阴影

通过上述所有的学习，大家应该都明白我们最终的算然结果是怎样得来的——从最初漫反射时的ambient+diffuse，到高光反射ambient+diffuse+specular，再到后来的透明度时将输出的第二个参数设置为透明度值，到现在的ambient + (diffuse + specular) * atten * shadow。我们可以发现我们都是将光照结果与其他影响因子做计算得到的，环境光、漫反射、高光反射需要我们一个一个去求解，那么后面的影响因子呢？就比如现在的衰减值和阴影值？

好消息是，Unity为我们提供了同时计算这两个信息的内置宏：UNITY_LIGHT_ATTENUATION。

①头文件同时包含

#include "Lighting.cginc"

#include "AutoLight.cginc"

②SHADOW_COORDS、TRANSFER_SHADOW的使用与前面的时一样的；

③在片元着色器中使用UNITY_LIGHT_ATTENUATION进行计算

```c#
fixed4 frag(v2f i) : SV_Target
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));

	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
	fixed3 halfDir = normalize(worldLightDir + viewDir);
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

	//计算阴影
	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);

	return fixed4(ambient + (diffuse + specular) * atten, 1.0);
}
```

UNITY_LIGHT_ATTENUATION有三个参数：它会将光照衰减值和阴影值相乘后的结果存储在第一个参数中。你会发现我们并没有声明第一个参数atten，这是因为UNITY_LIGHT_ATTENUATION会帮我们声明这个变量；第二个参数是结构体v2f的对象；用来计算阴影值；第三个参数是世界空间的坐标，这个参数会用于计算光源空间下的坐标，再对光照衰减纹理采样得到光照衰减。

但要注意，在AutoLight.cginc中，Unity针对不同光源类型、是否启用cookie等不同的情况声明了多个版本的UNITY_LIGHT_ATTENUATION。

#### 15.4.4 透明物体的阴影

在讲透明物体的阴影之前想先提另一个东西，这也是我在初学阴影的时候的疑问点——就是你在刚使用Unity做3D类型的Demo的时候，你会发现，创建一个Cube或其他的，也依然会在其他物体上产生阴影，俺么我们为什么要专门写个shader来展示阴影呢？那是因为：

当你创建一个Cube或其他的3D模型时，它自带了一个叫Standard的shader，也叫标准着色器，它里面提供了各种各样的Pass，其中就有阴影的，同时，它的FallBack时VertexLit，这个Shader是干嘛的？

VertexLit.shader只有一个Pass，查看它的源码：

```c#
Pass {
    Nmae "ShadowCaster"
    Tags { "LightMode"="ShadowCaster" }
    
    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    #pragma multi_compile_shadowcaster
    #include "UnityCG.cginc"
        
    struct v2f {
        V2F_SHADOW_CASTER;
    }
    
    v2f vert (appdata_base v) {
        v2f o;
        TRANSFER_SHADOW_CASTER_NORMALOFFSET(o);
        reutnr o;
    }
    
    flaot4 frag (v2f i) : SV_Target {
        SHADOW_CASETER_FRAGMENT(i);
    }
    
    ENDCG
}
```

让物体产生阴影。这就是新建物体也能产生并接收阴影的原因。

对于不透明物体，我们直接FallBack这个就能得到阴影，但对于透明物体，我们就需要非常小心地设置这些物体的FallBack了。

原因是：透明度测试需要在片元着色器中舍弃某些片元，而VertexLit并没有这样的操作。具体操作如下：

①头文件还是那两个：

#include "Lighting.cginc"

#include "AutoLight.cginc"

②在v2f中使用内置宏SHADOW_COORDS声明阴影纹理坐标

```c#
struct v2f
{
	float4 pos : SV_POSITION;
	float3 worldNormal : TEXCOORD0;
	float3 worldPos : TEXCOORD1;
    float2 uv : TEXCOORD2;
	SHADOW_COORDS(3)
};
```

因为占用了3个插值寄存器，所以SHADOW_COORDS的参数是3，也就是说我们将占用第四个插值寄存器。

③在顶点着色器中使用内置宏TRANSFER_SHADOW计算阴影纹理坐标，并传递给片元着色器

```c#
v2f vert (a2v v){
    v2f o;
    ...;
    TRANSFER_SHADOW(o);
    
    return o;
}
```

④在片元着色器中使用内置宏UNITY_LIGHT_ATTENUATION计算阴影和光照衰减：

```c#
fixed4 frag(v2f i) : SV_Target {
    ...;
    UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
    return fixed4(ambient + diffuse * atten, 1.0);
}
```

⑤对于Fallback，我们使用Transparent/Cutout/VertexLit，以使镂空区域的阴影正常。

若还是有部分光线通过了不应该透过光的部分，那么将Mesh Render组件中的Cast Shadows属性设置为Two Sided

## 16 高级纹理

接下来我们将聊聊高维的纹理。

### 16.1 立方体纹理

在图形学中，立方体纹理是环境映射的一种实现方法。环境映射可以模拟物体周围的环境，使用了环境映射的物体可以看起来像是镀了层金属一样发射出周围的环境。

立方体纹理一共包含六张图片，对应着一个立方体的六个面。对立方体纹理进行采样，需要一个三维的纹理坐标，这个三维纹理坐标表示了我们在世界空间下的一个3D方向。这个方向矢量从立方体的中心出发，当它向外部延伸时，会与六个纹理之一相交，而采样得到的结果就是由该点计算而来。

立方体纹理在实时渲染中有很多应用：天空盒、环境映射。

#### 16.1.1 天空盒

天空盒，用于模拟游戏背景。正如其名：用来模拟天空，并且是个盒子。当我们使用了一个天空盒，整个场景就被包含在一个立方体内，而这个立方体的每个面使用的技术就是立方体纹理映射技术。

天空盒的创建我们就不说了，这东西的创建教程似乎已经满大街都是了。

#### 16.1.2 创建用于环境映射的立方体纹理

用这种方式，可以模拟出金属质感的材质。创建环境映射的立方体纹理分方法有三种：

①直接由一些特殊布局的纹理创建；

②手动创建一个Cubemap资源，再把6张图赋给它；

③有脚本生成

前两种方法都需要我们提前准备好立方体纹理的图像，它们得到的立方体纹理往往是被场景中的物体所共用的。，但我们所希望的是根据物体在场景中位置的不同，生成它们各自不同的立方体纹理，这种情况下就是使用脚本来创建。这是通过利用Unity提供的Camera.RenderToCubemap函数来实现。这个函数可以把从任意位置观察到的场景图像存储到6张图像中，从二创建出该位置上对应的立方体纹理。

使用它的代码为如：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEditor;

public class CubeMap_Try01 : ScriptableWizard
{
    public Transform renderFromPosition;
    public Cubemap cubemap;

    void OnWizardUpdate()
    {
        helpString = "Select transform to render from and cubemap to render into";
        isValid = (renderFromPosition != null) && (cubemap != null);
    }

    void OnWizardCreate()
    {
        GameObject go = new GameObject("CubemapCamera");
        go.AddComponent<Camera>();
        go.transform.position = renderFromPosition.position;
        go.GetComponent<Camera>().RenderToCubemap(cubemap);
        DestroyImmediate(go);
    }

    [MenuItem("GameObject/Render into Cubemap")]
    static void RenderCubemap()
    {
        ScriptableWizard.DisplayWizard<CubeMap_Try01>(
            "Render cubemap", "Render!");
    }
}
```

在这段代码中，在renderFromPosition（这个由用户指定）位置处动态创建一个摄像机，再调用RenderToCubemap函数把从当前位置观察到的图像渲染到用户指定的立方体纹理cubemap中，完成后再销毁临时摄像机。注意：携带有这块代码的脚本文件应该放在Editor文件夹下才能正确执行

具体步骤为（我这儿默认你的代码写好并放在了Editor文件夹下面）：

①创建一个空的GameObject；

②创建一个用于存储的立方体纹理（Create -> Legacy -> Cubemap），并勾选其属性面板中的Readable（还有一个名为Face size的属性，其大小决定了渲染出来的立方体的纹理，成正比）；

③在菜单栏选择并打开GameObject -> Render into Cubemap，将步骤①②中的对象放入窗口中，然后点击右下角的Render

④在Cubemap窗口中就能看到从GameObject位置观察到的世界空间下的6张图像，及立方体纹理。

#### 16.1.3 反射

使用反射效果的物体通常看起来就像是镀了一层金属——通过入射光线的方向和表面法线方向来计算反射方向，再利用反射方向对立方体纹理采样。

①在Properties中声明与反射有关的属性

```c#
_Color("Color Tint", Color) = (1, 1, 1, 1)
_ReflectColor("Reflection Color", Color) = (1, 1, 1, 1)
_ReflectAmount("Reflect Amount", Range(0, 1)) = 1
_Cubemap("Reflection Cubemap", Cube) = "_Skybox" {}
```

_ReflectColor用于控制反射颜色， _ReflectAmount用于控制这个材质的反射程度， _Cubemap用于模拟反射的环境映射纹理。

②在顶点着色器中计算该顶点的反射方向，通过CG的reflect函数来实现：

```c#
v2f vert(a2v v) 
{
	v2f o;

	o.pos = UnityObjectToClipPos(v.vertex);

	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

	o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);

	o.worldRefl = reflect(-o.worldViewDir, o.worldNormal);

	TRANSFER_SHADOW(o);

	return o;
}
```

至于o.worldViewDir的值取负，前面讲过就不再复述。

③在片元着色器中，利用反射方向来对立方体纹理采样

```c#
fixed4 frag(v2f i) : SV_Target 
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
	fixed3 worldViewDir = normalize(i.worldViewDir);

	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 diffuse = _LightColor0.rgb * _Color.rgb * saturate(dot(worldNormal, worldLightDir));

	fixed3 reflection = texCUBE(_Cubemap, i.worldRefl).rgb * _ReflectColor.rgb;

	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);

	fixed3 color = ambient + lerp(diffuse, reflection, _ReflectAmount) * atten;

	return fixed4(color, 1.0);
}
```

与之前我们所写过的代码的区别点在使用了texCUBE函数，对立方体进行采样。要注意的是，在进行立方体采样时，i.worldPos并不是归一化后的坐标值。这是因为：用于采样的参数仅仅是作为方向变量传递给texCUBE函数的，它的作用仅仅是提供一个方向而已，归一化与否并不影响。然后，我们使用 _ReflectAmount来混和漫反射颜色和反射颜色，并和环境光相加后返回这个和。

（这里提一下lerp函数，无论是CG、Mathf，还是Vector3，lerp函数都是三个参数，第三个参数是一个混合值，取值范围在0~1，返回的是第二个参数乘以第三个参数，再传给第一个参数）

最后就是将立方体纹理拖进模型中。

#### 16.1.4 折射

折射定律（斯涅尔定律）是啥我们就不说了，直接上公式：
$$
\eta_{1}sin\theta_{1} = \eta_{2}sin\theta_{2}
$$
*η1*和*η2*分别是两个介质的折射率。

通常，当我们得到折射方向后我们就会直接使用它来对立方体纹理进行采样，但这是不符合物理现实的。因为对一个透明物体而言，一种更准确的模拟方法需要计算两次折射——一次是当光线进入它内部时，另一次是从它内部射出时。但是想要实时渲染中模拟出第二次折射方向很复杂，也很难，虽然有方法。但是仅仅只模拟一次得到的效果从视觉上看起来也挺像那么回事的，毕竟正如图形学所说“如果它看起来是对的，那么它就是对的”，所以我们通常只模拟一次。

①在Properties中声明四个新的属性：

```c#
_Color("Color Tint", Color) = (1, 1, 1, 1)
_RefractColor("Refraction Color", Color) = (1, 1, 1, 1)
_RefractAmount("Refraction Amount", Range(0, 1)) = 1
_RefractRatio("Refraction Ratio", Range(0.1, 1)) = 0.5
_Cubemap("Refraction Cubemap", Cube) = "_Skybox" {}
```

相对于反射的属性，折射中我们增加了_RefractRatio属性，用于得到不同介质的透射比，以此来计算折射方向。

②在顶点着色器中计算折射方向：

```c#
v2f vert(a2v v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);

	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

	o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);

	o.worldRefr = refract(-normalize(o.worldViewDir), normalize(o.worldNormal), _RefractRatio);

	TRANSFER_SHADOW(o);

	return o;
}
```

使用CG的refract函数来计算折射方向。它的第一个参数是为入射光线的方向，必须是归一化后的矢量；第二个参数是表面法线，同样需要是归一化后的矢量；第三个参数是入射光线所在介质的折射率与折射光线所在介质的折射率之间的比值。例如光从空气射到玻璃表面，那么这个值就是1/1.5。函数的结果是折射方向，它的模等于入射光线的模。

③在片元着色器中使用折射方向对立方体纹理进行采样

```c#
fixed4 frag(v2f i) : SV_Target 
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
	fixed3 worldViewDir = normalize(i.worldViewDir);

	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	fixed3 diffuse = _LightColor0.rgb * _Color.rgb * saturate(dot(worldNormal, worldLightDir));
    
	fixed3 refraction = texCUBE(_Cubemap, i.worldRefr).rgb * _RefractColor.rgb;

	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);

	fixed3 color = ambient + lerp(diffuse, refraction, _RefractAmount) * atten;

	return fixed4(color, 1.0);
}
```

方法与反射是一致的。

#### 16.1.5 菲涅尔反射

在实时渲染中，我们常用菲涅尔反射来根据视角方向控制反射程度。通俗地讲，当光线照射到物体表面上时，一部分发生反射，一部分进入物体内部发生折射或散射。被反射的光和入射光之间存在一定的比率关系，这个比率关系可以通过菲涅尔等式进行计算。菲涅尔反射在生活中最常见的是：你在湖边，你可以很清楚的看见脚边的水面以及看到水底的状况，当看向远处时，会发现几乎看不见水下的情景，只能看见水面反射的环境。

事实上，反光物体都会有菲涅尔效果。我们一般使用菲涅尔等式来实现菲涅尔反射，但由于其过于复杂，在实时渲染中，我们通常会使用一些解释公式来计算，有两个：

①Schlick菲涅尔近似等式：
$$
F_{Schlick}(v,n) = F_{0}+(1-F_{0})(1-v\cdot n)^{5}
$$
F0是一个反射系数，用于控制菲涅尔反射的强度，v是视角方向，n是表面法线

②Empricial菲涅尔近视等式：
$$
F_{Empricial}(v,n) = max(0,min(1,bias+scale*(1-v\cdot n)^{power}))
$$
bias、scale、power是控制项。

具体的实现（使用Schlick菲涅尔近似等式）：

①在Properties中声明相关属性：

```c#
_Color("Color Tint", Color) = (1, 1, 1, 1)
_FresnelScale("Fresnel Scale", Range(0, 1)) = 0.5
_Cubemap("Reflection Cubemap", Cube) = "_Skybox" {}
```

②在顶点着色器中计算世界空间下的法线方向、视角方向和反射反射方向，与反射的顶点着色器是一样的

```c#
v2f vert(a2v v) 
{
	v2f o;

	o.pos = UnityObjectToClipPos(v.vertex);

	o.worldNormal = UnityObjectToWorldNormal(v.normal);

	o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

	o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);

	o.worldRefl = reflect(-o.worldViewDir, o.worldNormal);

    TRANSFER_SHADOW(o);

    return o;
}
```

③在片元着色器在计算菲涅尔反射，并使用结果混合漫反射光照和反射光照

```c#
fixed4 frag(v2f i) : SV_Target 
{
	fixed3 worldNormal = normalize(i.worldNormal);
	fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
	fixed3 worldViewDir = normalize(i.worldViewDir);

	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);

	fixed3 reflection = texCUBE(_Cubemap, i.worldRefl).rgb;

	fixed fresnel = _FresnelScale + (1 - _FresnelScale) * pow(1 - dot(worldViewDir, worldNormal), 5);

	fixed3 diffuse = _LightColor0.rgb * _Color.rgb * saturate(dot(worldNormal, worldLightDir));

	fixed3 color = ambient + lerp(diffuse, reflection, saturate(fresnel)) * atten;

	return fixed4(color, 1.0);
}
```

对比反射的代码，相当于就是将反射中的_ReflectAmount替换成了fresnel， _ReflectAmount取值在0~1，计算菲涅尔反射时我们也是用saturate函数将fresnel取值在0~1。

在计算fresnel时，_FresnelScale我们是设为的public，所以说，当我们将 _FresnelScale调为1时，物体就是完全反射；调为0时，物体就是一个具有边缘光照效果的漫反射物体。

### 16.2 渲染纹理

在之前的内容中，一个摄像机的渲染结果会输出到颜色缓冲中，并显示在我们的屏幕上。现代GPU允许我们将整个三维场景渲染到一个中间缓冲中，即渲染目标纹理（Render Target Texture，RTT），而不是传统的帧缓冲或后备缓冲。与之相关的是多重渲染目标（Multiple Render Target，MRT）。这种技术指的是GPU允许我们把场景同时渲染到多个渲染目标纹理中，而不再需要为每个渲染目标纹理单独渲染完整的场景。延迟渲染就是使用多重渲染目标的一个应用。

Unity为渲染目标纹理定义了一种专门的纹理类型——渲染纹理（Render Texture）。Unity中渲染纹理的使用通常有两种方式：一种是在Project目录下创建一个渲染纹理，然后将某个摄像机的渲染目标设置成该渲染纹理，这样就能将该摄像机的渲染结果实时更新到渲染纹理中。另一种是在屏幕后处理时使用GrabPass命令或OnRenderImage函数来获取当前屏幕图像，Unity会把这个屏幕图像放到一张和屏幕分辨率等同的渲染纹理中。

#### 16.2.1 镜子效果

我们使用渲染纹理来模拟镜子效果。

①在Properties中声明一个纹理属性

```c#
_MainTex("Main Tex", 2D) = "white" {}
```

②在顶点着色器中计算纹理坐标

```c#
v2f vert(a2v v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);

	o.uv = v.texcoord;

	o.uv.x = 1 - o.uv.x;

	return o;
}
```

因为镜子里显示的图像都是左右相反的，所以我们将x分量的纹理坐标进行了左右翻转。

③在片元着色器中对渲染纹理进行采样和输出

```c#
fixed4 frag(v2f i) : SV_Target 
{
	return tex2D(_MainTex, i.uv);
}
```

代码完成后我们将携带有这个shader的Material赋给被我们视为镜子的Quad，创建渲染纹理Render Texture，同时赋给Quad中的 _MainTex和一个新的Camera。

要注意的是：我们将渲染纹理的分辨率设置为的256*256，这样做有时会使图像模糊不清，此时就可以提高分辨率或更多的抗锯齿采样，但提高分辨率的后果就是会影响带宽和性能。

#### 16.2.2 玻璃效果

我们还可以通过在Shader中使用GrabPass来完成获取屏幕图像的目的。当我们在Shader中定义了一个GrabPass后，Unity就会把当前屏幕的图像绘制在一张纹理中，以便我们在后续的Pass中访问它。我们通常使用GrabPass来实现诸如玻璃等透明材质的。与使用简单的透明混合相比，使用GrabPass可以让我们对该物体后面的图象做进一步的处理，例如使用法线来模拟折射效果，而不再是简单地和原屏幕的颜色进行混合。

但使用GrabPass就需要额外小心物体的渲染队列设置，我们需要将包含GrabPass的渲染队列设置为透明队列，即"Queue" = "Transparent"。这样才能保证当渲染该物体时，所有的不透明物体都已经绘制在屏幕上了，从而获取正确的屏幕图像。

如何使用GrabPass来模拟一个玻璃效果呢？

先使用一张法线纹理来修改模型的法线信息，再通过一个Cubemap来模拟玻璃的反射而在模拟折射时，则使用了GrabPass获取玻璃后面的屏幕图像，并使用空间下的法线对屏幕纹理坐标偏移后，再对屏幕图像进行采样来模拟近似的折射效果。

①在Properties中是声明各个属性

```c#
_MainTex("Main Tex", 2D) = "white" {}
_BumpMap("Normal Map", 2D) = "bump" {}
_Cubemap("Environment Cubemap", Cube) = "_Skybox" {}
_Distortion("Distortion", Range(0, 100)) = 10
_RefractAmount("Refract Amount", Range(0.0, 1.0)) = 1.0
```

_MainTex是该玻璃的材质纹理，我们默认设置为白色； _BumpMap是玻璃的法线纹理； _Cubemap是用于模拟反射的环境纹理； _Distortion用于控制模拟折射时图像的扭曲程度； _RefractAmount用于控制折射程度，取值为0时该玻璃只包含反射效果，取值为1时该玻璃只包含折射效果。

②定义相应的渲染队列，并使用GrabPass来获取屏幕图像

```c#
SubShader
{
	Tags { "Queue" = "Transparent" "RenderType" = "Opaque" }

	GrabPass { "_RefractionTex" }
    ......;
}
```

z怎么理解这两个渲染队列？

Queue设置为Transparent，意味着在确保该物体被渲染之前，其他所有不透明的物体已经被渲染到屏幕上了；RenderType设置为Opaque，是为了在是哦也能够着色器替换时，该物体可以在被需要时被正确渲染。接下来的就是对GrabPass的定义，_RefractionTex是纹理名称，决定了抓取到的屏幕图像将会被存入哪个纹理。虽然省略声明该字符串，但直接声明纹理名称的方法往往可以得到跟高的性能。

③声明需要的变量

```c#
sampler2D _MainTex;
float4 _MainTex_ST;
sampler2D _BumpMap;
float4 _BumpMap_ST;
samplerCUBE _Cubemap;
float _Distortion;
fixed _RefractAmount;
sampler2D _RefractionTex;
float4 _RefractionTex_TexelSize;
```

_RefractionTex_TexelSize可以让我们得到该纹理的纹素大小。

④顶点着色器

```c#
v2f vert(a2v v) 
{
	v2f o;

    o.pos = UnityObjectToClipPos(v.vertex);

    o.scrPos = ComputeGrabScreenPos(o.pos);

    o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
	o.uv.zw = TRANSFORM_TEX(v.texcoord, _BumpMap);

    float3 worldPos = mul(_Object2World, v.vertex).xyz;
	fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
	fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
	fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w;

    o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
	o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
	o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);

	return o;
}
```

顶点坐标转换后，调用内置的ComputeGrabScreenPos函数来得到对应被抓取的屏幕图像的采样坐标。然后分别计算_MainTex和 _BumpMap的采样坐标，分别存储在一个float4类型变量的xy和zw分量中。为了对Cubemap进行采样，我们就需要在片元着色器中把法线方向从切线空间变换到世界空间下，所以我们就需要计算出从切线空间到世界空间的变换矩阵，还是熟悉的那个计算方法，就不再多说了。

⑤片元着色器

```c#
fixed4 frag(v2f i) : SV_Target 
{
	float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
	fixed3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));

	fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));

	float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;
	i.scrPos.xy = offset + i.scrPos.xy;
	fixed3 refrCol = tex2D(_RefractionTex, i.scrPos.xy / i.scrPos.w).rgb;

	bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
	fixed3 reflDir = reflect(-worldViewDir, bump);
	fixed4 texColor = tex2D(_MainTex, i.uv.xy);
	fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb;

	fixed3 finalColor = reflCol * (1 - _RefractAmount) + refrCol * _RefractAmount;

	return fixed4(finalColor, 1);
}
```

先通过TtoW0的w分量得到世界坐标，再利用这个值得到该片元对应的视角方向。接着对法线纹理采样，，得到切线空间下的法线方向，然后与_Distortion和 _RefractionTex_TexelSize来对屏幕坐标的采样坐标进行偏移，模拟折射效果。 _Distortion的值越大，偏移量越大，玻璃背后的物体看起来变形程度就越大。此处，我们选择使用切线空间下的法线方向来进行偏移，是因为该空间下的法线可以反映局部空间下的法线方向。随后我们对scrPos透视除法得到真正的屏幕坐标，再使用该坐标对抓取到的屏幕图像 _RefractionTex进行采样，得到模拟的折射颜色。然后我们再把法线方向从切线空间变换到世界空间下，并据此反射方向，再使用反射方向对Cubemap进行采样，并将结果与主纹理颜色相乘后得到反射颜色。

#### 16.2.3 渲染纹理 VS GrabPass

明显你会发现同样是抓取屏幕图像，GrabPass比渲染纹理要简单许多，但效率上，渲染纹理要好许多，尤其是移动设备上。因为GrabPass往往需要CPU直接读取后备缓冲中的数据，就会破坏CPU与GPU之间的并行性，这就会耗费许多额外的时间，在一些移动设备上是不支持的。

### 16.3 程序纹理

程序纹理指的是那些由计算机生成的图像，我们通常使用一些算法来创建个性化图案或非常真实的自然元素，例如树木、石头等。好处是我们可以使用各种参数来控制纹理的外观，就不再仅仅是那些颜色的属性，可以是那些完全不同类型的图案属性。

#### 16.3.1 在Unity中实现简单的程序纹理

这里我们使用一个普通的计算纹理高光反射的shader，并赋给一个Material，但不加入任何的纹理图。

接着我们使用一个脚本来生成一个纹理，命名为ProceduralTextureGeneration.cs。

①这个脚本时要在编辑器模式下运行，所以需要在类的开头添加：

```c#
[ExecuteInEditMode]
public class ProceduralTextureGeneration : MonoBehaviour {
```

②声明一个材质，这个材质将使用该脚本中生成的程序纹理

```c#
public Material material = null;
```

③生命程序纹理使用的各种参数

```c#
private int m_textureWidth = 512;
public int textureWidth {
		get {
			return m_textureWidth;
		}
		set {
			m_textureWidth = value;
			_UpdateMaterial();
		}
	}

private Color m_backgroundColor = Color.white;
public Color backgroundColor {
		get {
			return m_backgroundColor;
		}
		set {
			m_backgroundColor = value;
			_UpdateMaterial();
		}
	}

private Color m_circleColor = Color.yellow;
public Color circleColor 
    {
		get {
			return m_circleColor;
		}
		set {
			m_circleColor = value;
			_UpdateMaterial();
		}
	}

private float m_blurFactor = 2.0f;
public float blurFactor
	{
	get 
     	{
			return m_blurFactor;
		}
		set 
        {
			m_blurFactor = value;
			_UpdateMaterial();
		}
	}
```

我们声明了四个属性变量：纹理的大小——通常是2的整数幂；纹理的背景颜色；纹理颜色；模糊因子——用来模糊边界。

④为了保存生成的程序纹理，声明一个Texture2D类型的纹理变量：

```c#
private Texture2D m_generatedTexture = null;
```

⑤编写纹理的生成函数

```C#
void Start () 
{
	if (material == null) 
    {
		Renderer renderer = gameObject.GetComponent<Renderer>();
		if (renderer == null) 
    	{
			Debug.LogWarning("Cannot find a renderer.");
			return;
		}
		material = renderer.sharedMaterial;
	}
    _UpdateMaterial();
}
```

我们需要在程序一启动瞬间就开始生成纹理，所以就放在Start函数中：先检测纹理是否为空，为空的话，就尝试从该脚本所在的物体上得到相应的材质（因为创建一个物体后它自身是携带初始材质的）。完成后，再调用 _UpdateMaterial来为其生成程序纹理。

⑥_UpdateMaterial函数

```c#
private void _UpdateMaterial() 
{
	if (material != null) 
    {
		m_generatedTexture = _GenerateProceduralTexture();
		material.SetTexture("_MainTex", m_generatedTexture);
	}
}
```

在material不为空的前提下，调用_GenerateProceduralTexture函数来生成一张程序纹理，并赋给m_generatedTexture变量，再调用material.SetTexture函数将该纹理赋给材质中为 _MainTex的属性。

⑦_GenerateProceduralTexture函数

```c#
private Texture2D _GenerateProceduralTexture()
{
	Texture2D proceduralTexture = new Texture2D(textureWidth, textureWidth);

	// 相邻圆之间的间隔
	float circleInterval = textureWidth / 4.0f;
	// 圆的半径
	float radius = textureWidth / 10.0f;
	// 边缘的模糊因子
	float edgeBlur = 1.0f / blurFactor;
	for (int w = 0; w < textureWidth; w++) 
    {
		for (int h = 0; h < textureWidth; h++) 
        {
			// 用背景颜色初始化像素
			Color pixel = backgroundColor;
			// 绘制9个圆
			for (int i = 0; i < 3; i++) 
            {
				for (int j = 0; j < 3; j++) 
                {
					// 计算圆心坐标
					Vector2 circleCenter = new Vector2(circleInterval * (i + 1), circleInterval * (j + 1));
					// 计算每一个像素到圆界的距离，以绘制圆的边界
					float dist = Vector2.Distance(new Vector2(w, h), circleCenter) - radius;
					// 模糊圆的边界
					Color color = _MixColor(circleColor, new Color(pixel.r, pixel.g, pixel.b, 0.0f), Mathf.SmoothStep(0f, 1.0f, dist * edgeBlur));
					// 混合颜色得到每一个像素的颜色
					pixel = _MixColor(pixel, color, color.a);
				}
			}
			proceduralTexture.SetPixel(w, h, pixel);
		}
	}
	proceduralTexture.Apply();
	return proceduralTexture;
}
```

因为这里我们是以均匀绘制9个圆为目标，所以我们先提前设置好与圆有关的变量——圆心间距、圆的半径，圆的边界。接着一个两层循环计算每一个像素，在每一个像素下，通过一个两层循环绘制9个圆，每循环一次，调用一次Texture2D的内置函数SetPixel，所有循环结束调用Texture2D的内置函数Apply。

⑧_MixColor混合颜色的函数

```c#
private Color _MixColor(Color color0, Color color1, float mixFactor) 
{
	Color mixColor = Color.white;
	mixColor.r = Mathf.Lerp(color0.r, color1.r, mixFactor);
	mixColor.g = Mathf.Lerp(color0.g, color1.g, mixFactor);
	mixColor.b = Mathf.Lerp(color0.b, color1.b, mixFactor);
	mixColor.a = Mathf.Lerp(color0.a, color1.a, mixFactor);
	return mixColor;
}
```

很简单，就不多说。

#### 16.3.2 Unity的程序材质

在Unity中，有一类专门使用程序纹理的材质，叫做程序材质，也就是程序材质的纹理不再是我们之前使用的普通贴图纹理，而是用16.3.1的方法生成的程序纹理。

实际上，我们也一般不会在Unity中生成程序纹理及程序材质，而是使用一个叫做Substance Designer的软件在Unity外部生成。

## 17 让画面动起来

这里我们将聊聊通过引入时间变量，以实现各种动画效果，例如流动的河流、广告牌等。

### 17.1 有关时间的Unity Shader中的内置变量

|        名称         |    类型    |                             描述                             |
| :-----------------: | :--------: | :----------------------------------------------------------: |
|      **_Time**      | **float4** | **设t是自该场景加载开始所经过的时间，4个分量的值分别是(t/20,t,2t,3t)** |
|    **_SinTime**     | **float4** |   **设t是时间的正弦值，4个分量的值分别是(t/8,t/4,t/2,t)**    |
|    **_CosTime**     | **float4** |   **设t是时间的余弦值，4个分量的值分别是(t/8,t/4,t/2,t)**    |
| **Unity_DeltaTime** | **float4** | **设dt为时间增量，4个分量的值分别是(dt,1/dt,smoothDt,1/smoothDt)** |

### 17.2 纹理动画（UV动画）

让模型顶点对应的uv随时间变化，使其对应纹理像素不断随时间变化。但是无法达到让物体“变形”的效果

#### 17.2.1 序列帧动画

序列帧动画是最常见的纹理动画之一，它的原理也很简单，就像电影一样，依次播放一系列关键帧图像，当播放速度达到一定数值时，看起来就是一个连续的动画。它的优缺点都很明显，优点是——具有很高的灵活性，缺点是——由于序列帧中每张关键帧图像都不一样，就会导致制作一张出色的序列帧纹理所需要的美术工程量巨大。

基于上述，我们要制作序列帧动画，我们要先提供一张包含关键帧的图像，再对这张图像进行shader处理。

①在Properties中设置有关序列帧动画的相关参数：

```c#
Properties 
{
	_Color ("Color Tint", Color) = (1, 1, 1, 1)
	_MainTex ("Image Sequence", 2D) = "white" {}
   	_HorizontalAmount ("Horizontal Amount", Float) = 4
   	_VerticalAmount ("Vertical Amount", Float) = 4
   	_Speed ("Speed", Range(1, 100)) = 30
}
```

_Speed属性就不谈了，就如其名； _HorizontalAmount和 _VerticalAmount分别表示了该图像在水平方向和竖直方向包含的关键帧图像的个数。

②由于序列帧图像通常是透明纹理，我们就需要设置Pass的渲染状态，以渲染透明效果。

```c#
SubShader 
{
	Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
	
	Pass 
    {
		Tags { "LightMode"="ForwardBase" }
			
		ZWrite Off
		Blend SrcAlpha OneMinusSrcAlpha
```

由于序列帧图像通常包含了透明通道，因此我们将其视为一个半透明对象。所以我们使用半透明的标签配置来设置序列帧的SubShader标签：将Queue和RenderType都设置为Transparent，IgnoreProjector设置为True，并在Pass中使用Blend命令来开启并设置混合模式，同时关闭深度写入。

③我们在顶点着色器中只进行基本的顶点变换，并把顶点纹理坐标存储在v2f结构体中

```c#
v2f vert (a2v v) 
{  
	v2f o;  
	o.pos = UnityObjectToClipPos(v.vertex);  
	o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);  
	return o;
}  
```

④片元着色器才是最重要的，所有与动画有关的计算全在片元着色器中。

```c#
fixed4 frag (v2f i) : SV_Target 
{
	float time = floor(_Time.y * _Speed);  
	float row = floor(time / _HorizontalAmount);
	float column = time - row * _HorizontalAmount;
    
    //half2 uv = float2(i.uv.x /_HorizontalAmount, i.uv.y / _VerticalAmount);
	//uv.x += column / _HorizontalAmount;
	//uv.y -= row / _VerticalAmount;
	half2 uv = i.uv + half2(column, -row);
	uv.x /=  _HorizontalAmount;
	uv.y /= _VerticalAmount;
				
	fixed4 c = tex2D(_MainTex, uv);
	c.rgb *= _Color;			
	return c;
}
```

要播放帧动画，从本质上来说，我们需要计算出每个时刻需要播放的关键帧在纹理中的位置。而由于序列帧纹理都是按照行列排列的，因此这个位置可以认为是该关键帧所在的行列索引数。查看_Time的参数信息可以得知其y分量就是自该场景加载后所经过的时间。我们将其与speed相乘后得到模拟时间，并用floor函数取整。再用time去除 _HorizontalAmount，整除部分就是行索引，余数就是列索引。

接着使用行列索引值来构建真正的采样坐标：将采样坐标映射到每个关键帧图像的坐标范围内——先把原纹理坐标按行列数进行等分，得到每个子图像的纹理坐标范围，然后使用当前的行列数对上述的结果进行偏移，得到当前子图像的纹理坐标。

需要注意的是：由于在Unity中，纹理坐标竖直方向的顺序和序列帧纹理中的顺序是相反的，前者时从下到上逐渐增大，后者的播放顺序是从上到下的，所欲对竖直方向的坐标偏移需要使用减法。

⑤最后的是Fallback，可以设置为Transparent/VertexLit，但也可不设（即关闭）。

#### 17.2.2 滚动的背景

滚动的背景在2D游戏中是用了很多，用来模拟游戏角色在场景中的穿梭，在一些影视动画中也用了很多。这些场景往往包含了多个层（layers）来模拟一种视差效果。这里我们将实现一个包含两层的无限滚动的2D游戏背景。

①在Properties中声明与滚动背景有关的属性

```c#
Properties 
{
	_MainTex ("Base Layer (RGB)", 2D) = "white" {}
	_DetailTex ("2nd Layer (RGB)", 2D) = "white" {}
	_ScrollX ("Base layer Scroll Speed", Float) = 1.0
	_Scroll2X ("2nd layer Scroll Speed", Float) = 1.0
	_Multiplier ("Layer Multiplier", Float) = 1
}
```

与之前不同的是我们这儿使用了两个与纹理有关的属性_MainTex和 _DetailTex，分别是第一层（较远）和第二层（较近）的背景纹理， _ScrollX和 _Scroll2X分别是第一层和第二层的水平滚动速度。 _Multiplier用于控制纹理的整体亮度。

②顶点着色器依然很简单

```c#
v2f vert (a2v v)
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
				
	o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex) + frac(float2(_ScrollX, 0.0) * _Time.y);
	o.uv.zw = TRANSFORM_TEX(v.texcoord, _DetailTex) + frac(float2(_Scroll2X, 0.0) * _Time.y);
				
	return o;
}
```

进行最基本的顶点变换，把顶点从模型空间变换到裁剪空间。然后计算两层背景纹理的纹理坐标：先用TRANSFORM_TEX来得到两张纹理的纹理坐标，再用内置的_Time.y（同序列帧动画中的 _Time）在水平方向上对纹理坐标进行偏移，以达到滚动效果。

③相对于序列帧动画，滚动背景的片元着色器就相对简单

```c#
fixed4 frag (v2f i) : SV_Target 
{
	fixed4 firstLayer = tex2D(_MainTex, i.uv.xy);
	fixed4 secondLayer = tex2D(_DetailTex, i.uv.zw);
				
	fixed4 c = lerp(firstLayer, secondLayer, secondLayer.a);
	c.rgb *= _Multiplier;
				
	return c;
}
```

在顶点着色器中我们将两张纹理坐标+偏移后的结果存储在i.uv中的，分别用的xy和zw分量来存储。在片元着色器中再根据这两个分量进行纹理采样，并使用lerp函数根据第二层纹理的a分量来混合。最后使用_Multiplier和混合后的颜色进行相乘以调整背景亮度。

### 17.3 顶点动画

原理和uv动画有相同的地方，但是顶点动画可以让物体动起来的同时，达到变形的效果。

其主要原理是在 顶点着色器中对模型的顶点进行偏移，再将偏移后的顶点位置输出到片元着色器中。

用正弦函数，以时间为变化量，输出对应的 x方向偏移，即可模拟简单的水流效果、广告牌技术、飘动的旗帜等

y = Asin(频率 * t  + w)

#### 17.3.1 流动的河流（2D版）

①根据原理使用的波形函数来设置相应的属性

```c#
Properties 
{
	_MainTex ("Main Tex", 2D) = "white" {}
	_Color ("Color Tint", Color) = (1, 1, 1, 1)
	_Magnitude ("Distortion Magnitude", Float) = 1
	_Frequency ("Distortion Frequency", Float) = 1
	_InvWaveLength ("Distortion Inverse Wave Length", Float) = 10
	_Speed ("Speed", Float) = 0.5
}
```

_MainTex是河流纹理， _Color用于控制整体的颜色， _Magnitude用于控制波形函数的振幅， _Frequency用于控制波动频率， _InvWaveLength用于控制波长的倒数（ _InvWaveLength越大，波长越小）， _Speed用于控制河流纹理的移动速度。

②设置为透明效果的SubShader标签

```c#
SubShader {
	Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True"}
```

与之前的透明效果设置不同的是，多设置了一个名为DisableBatching的标签，并设置为True。

DisableBatching是用来指明是否对该SubShader使用批处理，因为，批处理会合并所有相关模型，而这些模型各自的模型空间就会丢失。而我们在制作这个2D版的流动的河流中，我们是需要在物体的模型空间下对顶点位置进行偏移，所以就需要取消对该shader的批处理操作。

③设置Pass的渲染状态

```c#
Pass 
{
	Tags { "LightMode"="ForwardBase" }
			
	ZWrite Off
	Blend SrcAlpha OneMinusSrcAlpha
	Cull Off
```

我们就关闭了深度写入，开启并设置混合模式，并关闭了剔除功能。

④在顶点着色器中进行相关的顶点动画

```c#
v2f vert(a2v v) 
{
	v2f o;
				
	float4 offset;
	offset.yzw = float3(0.0, 0.0, 0.0);
	offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
	o.pos = UnityObjectToClipPos(v.vertex + offset);
				
	o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
	o.uv +=  float2(0.0, _Time.y * _Speed);
				
	return o;
}
```

因为是一个2D的河流流动效果，所以我们就是对顶点的x方向进行位移，因此就将yzw的位移量设置为0。然后使用_Frequency属性和内置的 _Time.y变量来控制正弦函数的频率；因为是在模型空间下，所以对上述的结果要加上模型空间下的位置分量，并乘以 _InvWaveLength来控制波长。最后对结果值乘以 _Magnitude属性来控制·波动幅度，得到最终的位移。最后将位移量加到顶点上后再进行顶点变换。

⑤在片元着色器中对纹理采样再添加颜色控制

```c#
fixed4 frag(v2f i) : SV_Target 
{
	fixed4 c = tex2D(_MainTex, i.uv);
	c.rgb *= _Color.rgb;
				
	return c;
} 
```

⑥最后的Fallback，可以设置为内置的Transparent/VertexLit，也可以选择关闭Fallback

#### 17.3.2 广告牌

广告牌技术会根据视角方向来旋转一个被纹理着色的多边形（通常就是简单的四边形，这个多边形即使广告牌），使得这些多边形总面对着摄像机的（初始布置场景的时候，可能不是的）。

广告牌技术被应用于，例如渲染烟雾、云朵、闪光效果等。

广告牌技术的本质就是构建旋转矩阵，旋转变换矩阵是需要3个基向量，广告牌技术使用的基向量通常就是表面法线（normal）、指向上的方向（up）、指向右的方向（right），此外还需要制定一个锚点，在旋转过程中这个锚点不变，所有的物体都是围绕这个锚点来旋转。

广告牌技术的重难点是：如何根据需求来构建3个相互正交的基向量。计算过程是：

先通过初始计算得到目标的表面法线（一般都是视角方向）和指向上的方向，这两者往往是不垂直的，当两者之一往往是固定的，例如：当模拟草丛时，我们希望指向上的方向是固定的，永远是(0,1,0)，法线方向应该随视角变化；当模拟粒子效果时，我们希望法线方向是固定的，永远指向视角方向，指向上的方向则可以发生变化。

我们假设法线方向是固定的，那么我们先根据初始的表面法线和指向上的方向来计算出目标方向的指向右的方向：
$$
right = up * normal
$$
对其归一化后，再由法线方向和指向右的方向计算出正交的指向上的方向：
$$
up' = normal * right
$$
这里再详细说一下为什么这么做。我们使用广告牌技术就是为了让目标始终对着我们视野，也就是摄像机，换句话说不管摄像机怎么转动，在我们看起来，目标始终是要垂直我们的视线的。在布置场景的时候这个垂直可能是没有的，我们要做的就是把目标转过来，使垂直条件满足。我们通过叉乘先求出它的向右的方向向量，最后再通过目标的向上的方向向量和向右的方向向量求出垂直于这个目标的方向向量。完毕！

①在Properties中声明有关的属性

```c#
Properties 
{
	_MainTex ("Main Tex", 2D) = "white" {}
	_Color ("Color Tint", Color) = (1, 1, 1, 1)
	_VerticalBillboarding ("Vertical Restraints", Range(0, 1)) = 1 
}
```

_VerticalBillboarding用于调整是固定法线还是固定指向上的方向。

②设置和17.3.1一样的SubShader标签和Pass的渲染状态，原由也是这个

```c#
SubShader 
{
	Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent" "DisableBatching"="True"}
    
    Pass 
    { 
		Tags { "LightMode"="ForwardBase" }
			
		ZWrite Off
		Blend SrcAlpha OneMinusSrcAlpha
		Cull Off
```

③顶点着色器依然是我们的核心，所有的计算都是在模型空间下的。

我们先选择模型空间的原点作为广告牌的锚点，并利用内置变量获取模型空间下的视角位置：

```c#
float3 center = float3(0, 0, 0);
float3 viewer = mul(unity_WorldToObject,float4(_WorldSpaceCameraPos, 1));
```

然后计算三个正交矢量。首先根据观察位置和锚点计算目标法线方向，并根据_VerticalBillboarding属性来控制垂直方向上的约束度。

```c#
float3 normalDir = viewer - center;
normalDir.y =normalDir.y * _VerticalBillboarding;
normalDir = normalize(normalDir);
```

当_VerticalBillboarding等于1时，法线方向固定为视角方向；当 _VerticalBillboarding等于0时，向上方向固定为(0,1,0)。最后再归一化得到方向矢量。

接下来是为了得到向上的方向矢量。为了避免初始场景中，目标以侧面线对着摄像机，也就是说避免法线方向与向上的方向平行，就需要对法线方向的y分量进行判断，以得到正确的向上方向。

```c#
float3 upDir = abs(normalDir.y) > 0.999 ? float3(0, 0, 1) : float3(0, 1, 0);
float3 rightDir = normalize(cross(upDir, normalDir));
upDir = normalize(cross(normalDir, rightDir));
```

当然，我们的向上的方向向量只是一个粗略的。得到三个正交基矢量后我们根据原始的位置相对于锚点的偏移量以及三个正交基矢量，以计算得到新的顶点位置，并将

```c#
float3 centerOffs = v.vertex.xyz - center;
float3 localPos = center + rightDir * centerOffs.x + upDir * centerOffs.y + normalDir * centerOffs.z;         
o.pos = UnityObjectToClipPos(float4(localPos, 1));
```

⑤片元着色器依然如河流那么简单

```c#
fixed4 frag (v2f i) : SV_Target 
{
	fixed4 c = tex2D (_MainTex, i.uv);
	c.rgb *= _Color.rgb;
				
	return c;
}
```

⑥Fallback也是一样的，可以设置为内置的Transparent/VertexLit，也可以选择关闭Fallback。

#### 17.3.3 注意事项

注意事项①：正如在设置标签时说的那样的，由于我们是在模型空间下进行的顶点动画，如果开启了批处理的话就会破坏这种动画效果，所以我们就使用DisableBatching强制关闭批处理，但这样做也有缺点：会带来一定的性能下降。因为换来的是Draw Call的增加。对广告牌的改进就是利用顶点颜色来存储每个顶点到锚点的距离值。

注意事项②：如果想在顶点动画中加入阴影，假如是像之前那样使用内置的Diffuse等包含阴影的Pass来渲染，就不会得到正确的阴影效果。因为：Unity的阴影绘制需要调用一个ShadowCasterPass，但这个Pass中并没有进行相关的顶点动画，如果直接使用这个Pass，就会使得Unity会按照原来的顶点位置来计算阴影，于是就会出错。解决问题的方法是：自定义一个ShadowCasterPass。

阴影投射的重点在于我们需要按正常Pass的处理来剔除片元或进行顶点动画。在自定义的阴影投射Pass中，我们通常会使用Unity提供的内置宏来计算阴影投射时需要的各种变量，包括：V2F_SHADOW_CASTER、TRANSFER_SHADOW_CASTER_NORMALOFFSET、SHADOW_CASTER_FRAGMENT，我们就只需要关注自定义计算的部分。

```c#
Pass 
{
	Tags { "LightMode" = "ShadowCaster" }
			
	CGPROGRAM
			
	#pragma vertex vert
	#pragma fragment frag
			
	#pragma multi_compile_shadowcaster
			
	#include "UnityCG.cginc"
			
	float _Magnitude;
	float _Frequency;
	float _InvWaveLength;
	float _Speed;
			
	struct v2f 
    { 
	    V2F_SHADOW_CASTER;
	};
			
	v2f vert(appdata_base v) 
	{
		v2f o;
			
		float4 offset;
		offset.yzw = float3(0.0, 0.0, 0.0);
		offset.x = sin(_Frequency * _Time.y + v.vertex.x * _InvWaveLength + v.vertex.y * _InvWaveLength + v.vertex.z * _InvWaveLength) * _Magnitude;
		v.vertex = v.vertex + offset;

		TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				
		return o;
	}
			
	fixed4 frag(v2f i) : SV_Target 
    {
	    SHADOW_CASTER_FRAGMENT(i)
	}
	ENDCG
}
```

在上述这个例子中，我们在顶点着色器的输出结构体中使用V2F_SHADOW_CASTER来定义阴影投射需要定义的变量；接着在顶点着色器中按之前对顶点的处理方法计算顶点的偏移量，不同的是，这里我们是直接将偏移值加到顶点位置变量中，再使用TRANSFER_SHADOW_CASTER_NORMALOFFSET让Unity帮我们完成剩下的部分。在片元着色器中，我们是直接使用SHADOW_CASTER_FRAGMENT来让Unity帮我们完成阴影投射的部分，把结果输出到深度图和阴影映射纹理中。

