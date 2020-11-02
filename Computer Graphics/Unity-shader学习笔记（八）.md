[TOC]



# Unity-shader学习笔记（八）

## 18 屏幕后处理效果

屏幕后处理效果是游戏中实现屏幕特效的常见方法。

### 18.1 建立一个基本的屏幕后处理脚本系统

屏幕后处理，指的是在渲染玩整个场景得到屏幕图像后，再对这个图象进行一系列操作，实现各种屏幕特效。

通过这种技术，可以为游戏画面添加更多的艺术效果，例如景深(Depth of Field)、运动模糊(Motion Blur)。要想实现屏幕后处理，就需要得到渲染后的屏幕图像，即抓取屏幕。我们需要使用Unity为我们提供的接口——OnRenderImage函数。其声明如下：

```c#
MonoBehaviour.OnRenderImage (RenderTexture src, RenderTexture dest)
```

我们在脚本中声明此函数后，Unity会把当前渲染得到的图像存储在第一个参数对应的源渲染纹理中，通过函数中的一系列操作后，再把目标渲染纹理（即第二个参数对应的渲染纹理）显示到屏幕上。在OnRenderImage函数中，通常是利用Graphics,Blit函数来完成渲染纹理的处理，它有三种声明：

```c#
public static void Blit(Texture src, RenderTexture dest);
public static void Blit(Texture src, RenderTexture dest, Material mat, int pass = -1);
public static void Blit(Texture src, Material mat, int pass = -1);
```

参数src对应源纹理，在屏幕后处理技术中，这个参数通常就是当前屏幕的渲染纹理或上一步处理后得到的渲染纹理。

参数dest是目标渲染纹理，如果它的值为null就会直接将结果显示在屏幕上。

参数mat是我们使用的材质，其使用的shader将会进行各种屏幕后处理操作，而src纹理将会被传递给shader中名为_MainTex的纹理属性。

参数pass的默认值为-1，表示将会依次调用shader内所有的P啊水水，否则只会调用给定索引的Pass。

在默认情况下，OnRenderImage函数会在所有不透明和透明的Pass执行完毕后被调用，这样就可以对场景中所有游戏对象都产生影响。有时候你也可能会

在执行完不透明的Pass（渲染队列小于2500的Pass，包括Background、Geometry、AlphaTest）就立即执行OnRenderImage函数，以避免对透明物体产生影响。此时要做的就是在OnRenderImage函数前添加ImageEffectOpaque属性来实现这样的目的。

因此，要在Unity中实现屏幕后处理效果的过程通常是：首先在摄像机中添加一个用于屏幕后处理的脚本。在这个脚本中，我们会实现OnRenderImage函数来获取当前屏幕的渲染纹理。然后再调用Graphics.Blit函数使用特定的shader来对当前图像进行处理，再把返回的渲染纹理显示到屏幕上。对于复杂的纹理，我们可能需要多次调用Graphics.Blit。

但是，并不是任何情况都能使用屏幕后处理技术，这取决于当前平台是否支持渲染纹理和屏幕特效、是否支持当前使用的UnityShader等。所以创建一个用于屏幕后处理效果的基类：

①首先，所有屏幕后处理效果都需要绑定在某个摄像机上，并且我们希望在编辑器状态下也能够执行该脚本来查看状态

```c#
[ExecuteInEditMode]
[RequireComponent (typeof(camera))]
public class PostEffectsBase : MonaBehaviour{
    ...;
}
```

②在Start函数中调用CheckResource函数，以检测各种资源和条件是否满足

```c#
protected void CheckResources(){
    bool isSupported = CheckSupport();
    if(isSupported == false){
        NotSupported();
    }
}

protected bool CheckSupport(){
    if(SystemInfo.supportsImagEffects == false || SystemInfo.supportsRenderTextures == false){
        Debug.LogWarning("This platform does not support image effects or render texture.");
        return false;
    }
    return true;
}

protected void NotSupported(){
    enabled = false;
}

protected void Start(){
    CheckResources();
}
```

SystemInfo是一个访问硬件设备信息的类，类中的属性都只是只读属性；supportsImagEffects是个bool类型函数，用于判断是否支持图形特效；supportsRenderTextures同样是个bool类型函数，用于判断是否支持渲染纹理。

③因为每个屏幕后处理效果通常都需要指定一个Shader来创建一个用于处理渲染纹理的材质，因此这个基类也要提供这样的方法

```c#
protected Material CheckShaderAndCreateMaterial(Shader shader, Material material){
    if (shader == null){
        return null;
    }
    
    if (shader.isSupported && material && material.shader == shader)
        return material;
    
    if (!shader.isSupported){
        return null;
    }
    else{
        material = new Material(shader);
        material.hideFlags = HideFlags.DontSave;
        if (material)
            return material;
        else
            return null;
    }
}
```

HideFlags类是一个枚举类，用于控制Object对象的销毁方式及其在检视面板中的可视性。DontSave的作用是在新场景中保留对象，但不保留其子类对象。isSupported是一个只读函数，用于判断这个shader是否可以在用户终端上运行。

### 18.2 调整屏幕亮度、饱和度和对比度

需要同时有控制材质的shader和控制屏幕显示的C#脚本

（1）控制屏幕显示的C#脚本

①创建脚本，并将脚本原本继承的MonoBehaviour更改为我们上面写的，命名为PostEffectsBase；

②声明该效果需要的Shader，并据此创建相应的材质：

```c#
public Shader briSatConShader;
private Material briSatConMaterial;
public Material material {
    get{
        briSatConShader = CheckShaderAndCreateMaterial(briSatConShader, briSatConMaterial);
        return briSatConMaterial;
    }
}
```

③定义调整亮度、饱和度和对比度的参数

```c#
[Range(0.0f, 3.0f)]
public float brightness = 1.0f;
[Range(0.0f, 3.0f)]
public float saturation = 1.0f;
[Range(0.0f, 3.0f)]
public float contrast = 1.0f;
```

提供初始值，并固定其变化区间。

④定义OnRenderImage函数来进行真正的特效处理

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest){
    if (material != null){
        material.SetFloat("_Brightness", brightness);
        material.SetFloat("_Saturation", saturation);
        material.SetFloat("_Contrast", contrast);
        
        Graphics.Blit(src, dest, material);
    }
    else{
        Graphics.Blit(src, dest);
    }
}
```

（2）控制材质的shader

①在Properties中声明属性

```c#
Properties {
	_MainTex ("Base (RGB)", 2D) = "white" {}
	_Brightness ("Brightness", Float) = 1
	_Saturation("Saturation", Float) = 1
	_Contrast("Contrast", Float) = 1
}
```

声明的属性要与OnRenderImage函数中的一致，后三个属性的值会由脚本而得。

②定义用于屏幕后处理的Pass标签

```c#
SubShader {
	Pass {  
		ZTest Always Cull Off ZWrite Off
```

屏幕后处理实际上是在场景中绘制了一个与屏幕痛快通告的四边形面片，为了防止它对其他物体产生影响，我们需要设置相关的渲染状态。这里我们选择的是关闭深度测试的深度写入，是为了防止它挡住在其后面被渲染的物体。这个基本上是屏幕后处理的标配。

③屏幕特效的顶点着色器一般都很简单，只需要进行必需的顶点变换。

```c#
struct v2f {
	float4 pos : SV_POSITION;
	half2 uv: TEXCOORD0;
};
			  
v2f vert(appdata_img v) {
	v2f o;
				
	o.pos = UnityObjectToClipPos(v.vertex);
				
	o.uv = v.texcoord;
					 
	return o;
}
```

④在片元着色器中进行亮度、饱和度和对比度的调整计算

```c#
fixed4 frag(v2f i) : SV_Target {
	fixed4 renderTex = tex2D(_MainTex, i.uv);  
				  
	//亮度
	fixed3 finalColor = renderTex.rgb * _Brightness;
				
	//饱和度
	fixed luminance = 0.2125 * renderTex.r + 0.7154 * renderTex.g + 0.0721 * renderTex.b;
	fixed3 luminanceColor = fixed3(luminance, luminance, luminance);
	finalColor = lerp(luminanceColor, finalColor, _Saturation);
				
	//对比度
	fixed3 avgColor = fixed3(0.5, 0.5, 0.5);
	finalColor = lerp(avgColor, finalColor, _Contrast);
				
	return fixed4(finalColor, renderTex.a);  
}  
```

⑤这里我们需要关闭shader的Fallback，即：

```c#
Fallback Off
```

### 18.3 边缘检测

边缘检测的原理是利用一些边缘检测算子对图像进行卷积操作，没错，就是深度学习/图像处理里面的那个卷积。

#### 18.3.1 什么是卷积

在图像处理中，卷积操作指的是使用一个卷积核对一张图像中的每个像素进行一系列操作。卷积核通常是一个四方形网格结构，该区域内的每一个放个都有一个权重值。当对某个像素进行卷积时，我们将卷积核的中心放置于该像素上，翻转核之后再依次计算核中每个元素和其覆盖的图像像素值的乘积并求和，得到的结果就是该位置的新像素。

#### 18.3.2 常见的边缘检测算子

边缘检测算子，也就是我们前面所说的卷积核，它就是用于检测边缘/那条边的。我们先想想，边应该是怎样的？或者说我们是怎样确定一个物体是有边的？

我在一张黑纸上画了一个白色的正方向，你可以马上指出在正方形和其余黑色部分的交界处就是一条边，你是怎么确定的？根据颜色。

我们也是这样，根据相邻像素之间存在差别明显的颜色、亮度、纹理等属性，来确定那条边。我们也将这种相邻像素之间的差值用梯度来表示，那么边界处的梯度值肯定是最大。

常见的边缘检测算子有三种：

①Roberts：
$$
G_{x} = \left[\matrix{
-1 & 0\\
0 & 1
}\right]\\
G_{y} = \left[\matrix{
0 & -1\\
1 & 0
}\right]
$$
②Prewitt：
$$
G_{x} = \left[\matrix{
-1 & -1 & -1\\
0 & 0 & 0\\
1 & 1 & 1
}\right]\\
G_{y} = \left[\matrix{
-1 & 0 & 1\\
-1 & 0 & 1\\
-1 & 0 & 1
}\right]
$$
③Sobel：
$$
G_{x} = \left[\matrix{
-1 & -2 & -1\\
0 & 0 & 0\\
1 & 2 & 1
}\right]\\
G_{y} = \left[\matrix{
-1 & 0 & 1\\
-2 & 0 & 2\\
-1 & 0 & 1
}\right]
$$
由于梯度是一个矢量，有方向，也有大小，所以我们就不能只对水平方向或垂直方向进行卷积计算梯度值，当算出各方向的梯度值后，再进行：
$$
G = \sqrt{G_{x}^2+G_{y}^2}
$$
但由于开方操作会对性能造成一定影响，我们就会使用绝对值操作来代替开根号：
$$
G = |G_{x}|+|G_{y}|
$$

#### 18.3.3 实现

同样，我们需要一个shader和一个C#脚本

（1）用于边缘检测的C#脚本

①依然继承于PostEffectsBase

```c#
public class EdgeDetection : PostEffectsBase{
    ...;
}
```

②声明该效果需要的Shader，并据此创建相应的材质

```c#
public Shader edgeDetectShader;
private Material edgeDetectMaterial = null;
public Material material {
    get {
        edgeDetectMaterial = CheckShaderAndCreateMaterial(edgeDetectShader, edgeDetectMaterial);
        return edgeDetectMaterial;
    }
}
```

③在脚本中提供应用于调整边缘线强度、描边颜色以及背景颜色的参数

```c#
[Rnage(0.0f, 1.0f)]
public float edgesOnly = 0.0f;

public Color edgeColor = Color.black;

public Color backgroundColor = Color.white;
```

当edgesOnly值为0时，边缘将会叠加在原渲染图像上；edgesOnly值为1时，则会只显示边缘，不显示原渲染图象。其中，背景颜色由backgroundColor指定，边缘颜色由edgeColor指定。

④定义OnRenderImage函数来进行真正的特效：

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest){
    if (material != null){
        material.SetFloat("_EdgeOnly", edgeOnly);
        material.SetFloat("_EdgeColor", edgeColor);
        material.SetFloat("_BackgroundColor", backgroundColor);
        
        Graphics.Blit(src, dest, material);
    }
    else{
        Graphics.Blit(src, dest);
    }
}
```

（2）shader部分

①在Properties中声明需要的属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white"{}
    _EdgeOnly ("Edge Only", Float) = 1.0
    _EdgeColor ("Edge Color", Color) = (0,0,0,1)
    _BackgroundColor ("Background Color", Color) = (1,1,1,1)
}
```

②为屏幕后处理的Pass设置相关的渲染状态

```c#
SubShader {
    Pass{
        ZTest Always Cull Off ZWrite Off
        ......;    
    }
}
```

③声明对应的变量

```c#
sampler2D _MainTex;
half4 _MainTex_TexelSize;
fixed _EdgeOnly;
fixed4 _EdgeColor;
fixed4 _BackgroundColor;
```

新声明的_MainTex_TexelSize变量是为了访问 _MainTex对应的每个纹素的大小。xxx_TexelSize是Unity为客户提供的访问xxx纹理对应的每个纹素的大小。

④在顶点着色器中计算边缘检测时需要的纹理坐标

```c#
struct v2f
{
	float4 pos : SV_POSITION;
	half2 uv[9] : TEXCOORD0;
};
			  
v2f vert(appdata_img v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
				
	half2 uv = v.texcoord;
				
	o.uv[0] = uv + _MainTex_TexelSize.xy * half2(-1, -1);
	o.uv[1] = uv + _MainTex_TexelSize.xy * half2(0, -1);
	o.uv[2] = uv + _MainTex_TexelSize.xy * half2(1, -1);
	o.uv[3] = uv + _MainTex_TexelSize.xy * half2(-1, 0);
	o.uv[4] = uv + _MainTex_TexelSize.xy * half2(0, 0);
	o.uv[5] = uv + _MainTex_TexelSize.xy * half2(1, 0);
	o.uv[6] = uv + _MainTex_TexelSize.xy * half2(-1, 1);
	o.uv[7] = uv + _MainTex_TexelSize.xy * half2(0, 1);
	o.uv[8] = uv + _MainTex_TexelSize.xy * half2(1, 1);
					 
	return o;
}
```

我们将采用Sobel算子进行，所以我们就在顶点着色器的输出结构体中定义了一个维数为9的纹理数组。在片元着色器中也是可以进行行纹理坐标的采样的，但后果就是运算增加，性能降低。

⑤重要的片元着色器

```c#
fixed4 fragSobel(v2f i) : SV_Target 
{
	half edge = Sobel(i);
				
	fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv[4]), edge);
	fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
	return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
}
```

我们首先调用Sobel函数计算当前像素的梯度值edge，并利用该值分别计算背景为原图和纯色下的颜色值，然后利用_EdgeOnly在两者之间插值得到最终的像素值。Sobel函数将利用Sobel算子对原图进行边缘检测。

```c#
fixed luminance(fixed4 color) 
{
	return  0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b; 
}
			
half Sobel(v2f i) 
{
	const half Gx[9] = {-1,  0,  1,
						-2,  0,  2,
						-1,  0,  1};
	const half Gy[9] = {-1, -2, -1,
						0,  0,  0,
						1,  2,  1};		
				
	half texColor;
	half edgeX = 0;
	half edgeY = 0;
	for (int it = 0; it < 9; it++) 
    {
		texColor = luminance(tex2D(_MainTex, i.uv[it]));
		edgeX += texColor * Gx[it];
		edgeY += texColor * Gy[it];
	}
				
	half edge = 1 - abs(edgeX) - abs(edgeY);
				
	return edge;
}
```

要先定义水平方向和竖直方向使用的卷积核Gx和Gy；然后我们依次对9个像素进行采样，计算它们的亮度值，再与卷积核Gx和Gy中对应的权重相乘后，叠加到各自的梯度值上。最后用1减去这两个梯度值的绝对值，就得到edge，值越小，表明该位置就越可能是一个边缘点。

⑥关闭Fallback

```c#
Fallback Off
```

### 18.4 高斯模糊

高斯模糊也使用了卷积操作。

最常用的模糊有两种：均值模糊和中值模糊。

均值模糊：使用的卷积核各个元素值都相等，且相加等于1，即卷积后得到的像素值是其领域内各个像素值的平均值。

中值模糊：选择领域内对所有像素排序后的中值替换掉原颜色

高斯模糊则比这两者更高级。

#### 18.4.1 高斯滤波

高斯模糊同样使用了卷积计算，它使用的卷积核名为高斯核。高斯核是一个正方向大小的滤波核，其中每个元素的计算都基于下面这个高斯方程：
$$
G(x,y)=\frac{1}{2\pi \sigma^{2}}e^{-{\frac{x^2+y^2}{2\sigma ^{2}}}}
$$
σ是标准方差，一般取值为1，x和y分别对应了当前位置到卷积核中心的整数距离。要构建高斯核，我们就只需要计算高斯核中各个位置对应的高斯值。为避免滤波后的图像不会变暗，我们需要对高斯核中的权重进行归一化，即让每个权重除以所有权重的和。

使用高斯方程的好处是：很好的模拟了领域每个像素对当前处理像素的影响程度——距离越大，影响越大。

高斯核的维数越高，模糊程度越大。但是，用一个N*N的高斯核对图像进行卷积滤波，就需要N * N * W * H次纹理采样（W和H分别是图像的宽和高）。当N的大小不断增加时，采样次数会变得非常巨大。我们采用的方法就是将这个二维高斯函数拆分成两个一维函数，也即使用两个一维的高斯核先后对图像进行滤波，它们得到的结果和直接使用二维高斯核是一样的，但采样次数降低至2 * N * W * H。

并且二维高斯核是具有旋转对称性的

#### 18.4.2 实现

对于两个一维的高斯核，我们使用两个Pass分别从两个方向对图像进行滤波，并同时通过调整高斯滤波的应用次数来控制模糊程度。

（1）继承于PostEffectsBase的C#脚本：

```c#
public class GaussianBlur : PostEffectsBase{
    ...;
}
```

①声明该效果需要的shader

```c#
public Shader gaussianBlurShader;
private Material gaussianBlurMaterial = null;

public Material material{
    get{
        gaussianBlurMaterial = CheckShaderAndCreateMaterial(gaussianBlurShader, gaussianBlurMaterial);
        return gaussianBlurMaterial;
    }
}
```

②声明调整高斯模糊迭代次数、模糊范围和缩放系数的参数

```c#
[Range(0, 4)]
public int iteration = 3;
[Range(0.2f, 3.0f)]
public float blurSpread = 0.6f;
[Range(1, 8)]
public int downSample = 2;
```

在高斯核位数不变的前提下，blurSpread越大，模糊程度越高，但过大的blurSpread会造成虚影；downSample越大，需要处理的像素数越少，也能提高模糊程度，但过大的downSample会使图像像素化。

③OnRenderImage函数

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest) 
{
	if (material != null) 
    {
		int rtW = src.width/downSample;
		int rtH = src.height/downSample;

		RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
		buffer0.filterMode = FilterMode.Bilinear;

		Graphics.Blit(src, buffer0);

		for (int i = 0; i < iterations; i++) 
        {
			material.SetFloat("_BlurSize", 1.0f + i * blurSpread);

			RenderTexture buffer1 = RenderTexture.GetTemporary(rtW, rtH, 0);

			//渲染第一个Pass，x方向的那个Pass
			Graphics.Blit(buffer0, buffer1, material, 0);

			RenderTexture.ReleaseTemporary(buffer0);
			buffer0 = buffer1;
			buffer1 = RenderTexture.GetTemporaray(rtW, rtH, 0);

			//渲染第二个Pass，y方向的那个Pass
			Graphics.Blit(buffer0, buffer1, material, 1);

			RenderTexture.ReleaseTemporary(buffer0);
				buffer0 = buffer1;
		}

		Graphics.Blit(buffer0, dest);
		RenderTexture.ReleaseTemporary(buffer0);
	} 
    else 
    {
		Graphics.Blit(src, dest);
	}
}
```

在迭代开始前，我们首先使用RenderTexture.GetTemporary函数定义一个缓存buffer0，并把src中的图像经缩放后存储在buffer0中；在迭代过程中，我们定义第二个缓存buffer1，在执行第一个Pass时，输入的是buffer0，输出的是buffer1，然后先把buffer0释放了，再把结果值buffer1存储到buffer0中，再重新分配buffer1，迭代完成后，buffer0将存储最终的图像，再利用Graphics.Blit把结果显示到屏幕上，并释放缓存。

（2）shader

①声明需要的属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _BlurSize ("Blur Size", Float) = 1.0
}
```

②我们将在SubShader中使用CGINCLUBE来组织代码

```c#
SubShader {
    CGINCLUBE
        ...
    ENDCG
        ...
}
```

它类似于C++中头文件的功能，使用它的好处就是可以避免我们编写两个完全一样的frag函数，因为在高斯模糊中我们虽然定义了两个Pass，但他们使用的片元着色器代码是完全一样的。

③在Pass的CG代码块中定义与属性对应的变量

```c#
sampler2D _MainTex;  
half4 _MainTex_TexelSize;
float _BlurSize;
```

④竖直方向的顶点着色器代码

```c#
struct v2f 
{
	float4 pos : SV_POSITION;
	half2 uv[5]: TEXCOORD0;
};
		  
v2f vertBlurVertical(appdata_img v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
		
	half2 uv = v.texcoord;
			
	o.uv[0] = uv;
	o.uv[1] = uv + float2(0.0, _MainTex_TexelSize.y * 1.0) * _BlurSize;
	o.uv[2] = uv - float2(0.0, _MainTex_TexelSize.y * 1.0) * _BlurSize;
	o.uv[3] = uv + float2(0.0, _MainTex_TexelSize.y * 2.0) * _BlurSize;
	o.uv[4] = uv - float2(0.0, _MainTex_TexelSize.y * 2.0) * _BlurSize;
					 
	return o;
}
```

水平方向的顶点着色器于此是一样的，所以就不再赘述。

⑤定义两个Pass共用的片元着色器

```c#
fixed4 fragBlur(v2f i) : SV_Target 
{
	float weight[3] = {0.4026, 0.2442, 0.0545};
		
	fixed3 sum = tex2D(_MainTex, i.uv[0]).rgb * weight[0];
			
	for (int it = 1; it < 3; it++) 
    {
		sum += tex2D(_MainTex, i.uv[it*2-1]).rgb * weight[it];
		sum += tex2D(_MainTex, i.uv[it*2]).rgb * weight[it];
	}
			
	return fixed4(sum, 1.0);
}
```

正如我们前面所说，高斯核具有高度的对称性，所以我们就只需要记录三个高斯权重。声明了3个高斯权重后就计算初始的sum值——当前的像素值乘以它的权重值。

每次迭代包含两次纹理采样，要对当前像素的左右像素进行采样，并把像素值核权重相乘后的结果叠加到sum中

⑥使用了CGINCLUBE一定要记得在最后定义两个Pass

```c#
ZTest Always Cull Off ZWrite Off
		
Pass 
{
	NAME "GAUSSIAN_BLUR_VERTICAL"
			
	CGPROGRAM
			  
	#pragma vertex vertBlurVertical  
	#pragma fragment fragBlur
			  
	ENDCG  
}
		
Pass 
{  
	NAME "GAUSSIAN_BLUR_HORIZONTAL"
			
	CGPROGRAM  
			
	#pragma vertex vertBlurHorizontal  
	#pragma fragment fragBlur
			
	ENDCG
}
```

注意，这时候我们要设置渲染状态。同时我们还定义了Pass的名字，是为了在其他Shader在直接通过名字来使用该Pass，毕竟高斯模糊过于常见。

### 18.5 Bloom特效

Bloom特效可以模拟真实摄像机的一种图像效果，它是让画面中较亮的区域扩散到周围的区域中，产生一种朦胧的效果。

它的实现原理也很简单，建立在高斯模糊上：首先根据一个阈值从图像中提取出较亮区域，把它们存储在一张渲染纹理中，再利用高斯模糊处理这张渲染纹理，模拟光线扩散的效果，最后再将这张渲染纹理于原图像进行混合，得到最终的效果。

（1）C#脚本

①创建的脚本依然是基于PostEffectsBase

②声明需要的shader，并据此创建相应的材质

```c#
public Shader bloomShader;
private Material bloomMaterial = null;
public Material material {
    get {
        bloomMaterial = CheckShaderAndCreateMaterial(bloomShader, bloomMaterial);
        return bloomMaterial;
    }
}
```

③一开始我们就说过Bloom效果是建立在高斯模糊上的，所以高斯模糊有的参数，Bloom也需要，此外，还需要新增一个参数luminanceThreshold，来控制提取较亮区域时使用的阈值大小。

```c#
[Range(0, 4)]
public int iteration = 3;
[Range(0.2f, 3.0f)]
public float blurSpread = 0.6f;
[Range(1, 8)]
public int downSample = 2;
[Range(0.0f, 4.0f)]
public float luminanceThreshold = 0.6f;
```

按理来说，图像的亮度是不会超过1的，但一旦我们开启了HDR，硬件就会允许我们把颜色存在一个更高精度范围的缓冲中，此时像素的亮度值就可能会超过1。所以我们将luminanceThreshold的取值范围扩大至4.

④OnRenderImage函数又来了

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest) {
    if (material != null) {
        material.SetFloat("_luminanceThreshold", luminanceThreshold);
        
        int rtW = src.width/downSample;
        int rtH = src.height/downSample;
        
        RenderTexture buffer0 = RenderTexture.GetTemporary(rtW, rtH, 0);
        buffer0.filterMode = FilterMode.Bilinear;
        
        Graphics.Blit(src, buffer0, material, 0);
        
        for (int i = 0; i < iterations; ++i){
            material.SetFloat("_BlurSize", 1.0f + i * blurSpread);
            
            RenderTexture buffer1 = RenderTexture.Gettemporary(rtW, rtH, 0);
            
            Graphics.Blit(buffer0, buffer1, material, 1);
            
            RenderTexture.ReleaseTemporary(buffer0);
            buffer0 = buffer1;
            buffer1 = RenderTexture.Gettemporary(rtW, rtH, 0);
            
            Graphics.Blit(buffer0, buffer1, material, 2);
            
            RenderTexture.ReleaseTemporary(buffer0);
            buffer0 = buffer1;
        }
        material.SetTexture("_Bloom", buffer0);
        Graphics.Blit(src, dest, material, 3);
        
        RenderTexture.ReleaseTemporary(buffer0);
    }else{
        Graphics.Blit(src, dest);
    }
}
```

OnRenderImage函数包括了实现Bloom效果的三个步骤：通过调用Graphics.Blit(src, buffer0, material, 0)来使用shader的第一个Pass来提取图像中的较亮区域，并存储到buffer0中；然后使用第二个和第三个Pass来进行高斯模糊迭代处理，模糊后较亮的区域会存储在buffer0中；最后再把buffer0传递给材质的_Bloom属性，并调用Graphics.Blit(src, dest, material, 3)，使用第四个Pass来进行最后的混合，并将结果存储在目标渲染纹理dest中。

记得释放临时缓存。

（2）shader部分

①声明需要的属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _Bloom ("Bloom (RGB)", 2D) = "black" {}
    _LuminanceThreshold ("Luminance Threshold", Float) = 0.5
    _BlurSize ("Blur Size", Float) = 1.0
}
```

②声明相应的变量

```c#
sampler2D _MainTex;
half4 _MainTex_TexelSize;
sampler2D _Bloom;
float _LuminanceThreshold;
float _BlurSize
```

③先定义提取较亮区域需要使用的顶点着色器和片元着色器

```c#
struct v2f 
{
	float4 pos : SV_POSITION; 
	half2 uv : TEXCOORD0;
};	
		
v2f vertExtractBright(appdata_img v) 
{
	v2f o;
			
	o.pos = UnityObjectToClipPos(v.vertex);
			
	o.uv = v.texcoord;
					 
	return o;
}
		
fixed luminance(fixed4 color) 
{
	return  0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b; 
}
		
fixed4 fragExtractBright(v2f i) : SV_Target 
{
	fixed4 c = tex2D(_MainTex, i.uv);
	fixed val = clamp(luminance(c) - _LuminanceThreshold, 0.0, 1.0);
			
	return c * val;
}
```

顶点着色器的内容与之前的相同，在片元着色器中我们将采样得到的亮度值减去阈值_LuminanceThreshold，并将结果截取到0~1，然后我们把该值和原像素值相乘，得到提取后的亮部区域。

④定义混合亮部图像和原图像时使用的顶点着色器和片元着色器

```c#
struct v2fBloom 
{
	float4 pos : SV_POSITION; 
	half4 uv : TEXCOORD0;
};
		
v2fBloom vertBloom(appdata_img v) 
{
	v2fBloom o;
		
	o.pos = UnityObjectToClipPos (v.vertex);
	o.uv.xy = v.texcoord;		
	o.uv.zw = v.texcoord;
			
	#if UNITY_UV_STARTS_AT_TOP			
	if (_MainTex_TexelSize.y < 0.0)
		o.uv.w = 1.0 - o.uv.w;
	#endif
				        	
	return o; 
}
		
fixed4 fragBloom(v2fBloom i) : SV_Target 
{
	return tex2D(_MainTex, i.uv.xy) + tex2D(_Bloom, i.uv.zw);
} 
```

⑤定义Bloom效果需要的4个Pass

```c#
ZTest Always Cull Off ZWrite Off
		
Pass 
{  
	CGPROGRAM  
	#pragma vertex vertExtractBright  
	#pragma fragment fragExtractBright  
			
	ENDCG  
}
		
UsePass "....../GAUSSIAN_BLUR_VERTICAL"
		
UsePass "....../GAUSSIAN_BLUR_HORIZONTAL"
		
Pass 
{  
	CGPROGRAM  
	#pragma vertex vertBloom  
	#pragma fragment fragBloom  
			
	ENDCG  
}
```

要注意的是Unity内部会将所有Pass的Name转换成大写字母表示，因此在使用UsePass命令时我们必须使用大写形式的名字。

### 18.6 运动模糊

运动模糊是真实世界中的摄像机的一种效果，它是由于相机传感器与物体相对运动按下快门瞬间造成图像产生运动模糊。在游戏世界中，最常见的是出现在一些赛车游戏中。

运动模糊的实现有多种方法，最简单的但也是最耗性能的是利用一块累积缓存来混和多张连续的图像，注意，是连续的图像。当物体快速移动产生多张图像后，我们取它们之间的平均值作为最后的运动模糊图像。非常消耗性能的原因就在于要想获取多帧图像就意味着我们要在同一帧里渲染多次场景。

另一种应用更多的方法是创建和使用速度缓存，这个缓存中存储了各个像素当前的运动速度，然后利用该值来决定模糊的方向和大小。

我们将采用类似于第一种的方法来实现运动模糊——我们选择不在一帧中把场景渲染多次，但需要保存之前的渲染结果，不断把当前的渲染图像叠加到之前的渲染图像中。这样做虽然会提升一定的性能，但模糊效果可能会打折扣。

（1）c#脚本

①依然继承于PostEffectsBase

```c#
public class MotionBlur : PostEffectsBase{
    ...;
}
```

②声明shader和模糊参数

```c#
public Material material 
{  
	get 
    {
		motionBlurMaterial = CheckShaderAndCreateMaterial(motionBlurShader, motionBlurMaterial);
		return motionBlurMaterial;
	}  
}

[Range(0.0f, 0.9f)]
public float blurAmount = 0.5f;
```

为了防止拖尾效果完全替代了当前帧的渲染结果，我们就只把它的值截取在0.0~0.9范围内。

③定义一个RenderTexture类型的变量，用于保存之前图像叠加的结果

```c#
private RenderTexture accumulationTexture;
void OnDisable(){
    DextroyImmediate(accumulationTexture);
}
```

④OnRenderImage函数

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest)
{
	if (material != null) 
    {
		if (accumulationTexture == null || accumulationTexture.width != src.width || accumulationTexture.height != src.height) 
        {
			DestroyImmediate(accumulationTexture);
			accumulationTexture = new RenderTexture(src.width, src.height, 0);
			accumulationTexture.hideFlags = HideFlags.HideAndDontSave;
			Graphics.Blit(src, accumulationTexture);
		}
		accumulationTexture.MarkRestoreExpected();

		material.SetFloat("_BlurAmount", 1.0f - blurAmount);

		Graphics.Blit (src, accumulationTexture, material);
		Graphics.Blit (accumulationTexture, dest);
	} else {
		Graphics.Blit(src, dest);
	}
}
```

在确定材质可用后，先去判断用于混合图像的accumulationTexture是否满足条件——需要判断是否为空、是否与当前屏幕分辨率相同。如果不满足任意一个，那么新创建一个条件都满足的accumulationTexture。由于我们是自己控制accumulationTexture的销毁，所以将其hideFlags设置为HideFlags.HideAndDontSave，就意味着accumulationTexture不会显示在Hierarchy中，也不会保存在场景中。当得到有效的accumulationTexture变量后，调用MarkRestoreExpected函数来表明我们需要恢复渲染纹理。恢复操作发生在渲染到纹理而该纹理又没有被提前清空或销毁的情况下。

（2）shader

①声明需要的变量

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _BlurAmount ("Blur Amount", Float) = 1.0
}
```

②同样我们也要使用CGINCLUBE还组织代码

③顶点着色器的内容与之前的shader一模一样

```c#
struct v2f 
{
	float4 pos : SV_POSITION;
	half2 uv : TEXCOORD0;
};
		
v2f vert(appdata_img v) 
{
	v2f o;
			
	o.pos = UnityObjectToClipPos(v.vertex);
			
	o.uv = v.texcoord;
				 
	return o;
}
```

④定义两个片元着色器，一个用于更新渲染纹理的RGB通道部分，另一个用于更新渲染纹理的A通道部分

```c#
fixed4 fragRGB (v2f i) : SV_Target{
    return fixed4(tex2D(_MainTex, i.uv).rgb, _BlurAmount);
}
fixed4 fragA (v2f i) : SV_Target{
    return tex2D(_MainTex, i.uv);
}
```

⑤定义运动模糊需要的Pass，这里我们需要两个：一个用于更新渲染纹理的RGB通道，一个用于更新A通道。要将RGB与A分开的原因是更新RGB时我们需要设置它的A通道来混和图像，但又不希望A通道的值写入渲染纹理中。

```c#
ZTest Always Cull Off ZWrite Off
		
Pass
{
	Blend SrcAlpha OneMinusSrcAlpha
	ColorMask RGB
			
	CGPROGRAM
			
	#pragma vertex vert  
	#pragma fragment fragRGB  
		
	ENDCG
}
		
Pass 
{   
	Blend One Zero
	ColorMask A
			   	
	CGPROGRAM  
			
	#pragma vertex vert  
	#pragma fragment fragA
		  
	ENDCG
}
```

## 19 使用深度和法线纹理

在18中我们是在屏幕颜色图像上进行各种操作来实现屏幕后处理效果的，这里我们将使用深度和法线纹理来实现特定的屏幕后处理效果。

### 19.1 获取深度和法线纹理

#### 19.1.1 背后的原理

深度纹理其实就是一张渲染纹理，它存储的像素值不再是颜色值，而是一个高精度的深度值。深度纹理中深度值的范围是[0,1]，而且通常是非线性分布的，怎么获取这些深度值呢？来自顶点变换后得到的归一化的设备坐标。很久很久之前的内容了，不知是否还记得，一个模型要想最终被绘制到屏幕上，需要将它的顶点从模型空间变换到齐次裁剪坐标系下，在顶点着色器中通过MVP变换矩阵得到。最后一步，也即那个p，是使用投影矩阵，这一步就能得到NDC的范围在[-1,1]的z分量，当然在DX中是[0,1]。这个z就是我们需要的深度值。

但由于NDC的z分量范围在[-1,1]，所以要将其映射在[0,1]内：
$$
d = 0.5\cdot z_{ndc}+0.5
$$
d对应深度纹理中的像素值，zndc对应NDC坐标中的z分量的值。

在Unity中怎么得到它？

它可以来自于真正的深度缓存，也可以是由一个单独的Pass渲染而得，这些都取决于使用的渲染路径和硬件。

当使用延迟渲染路径时，因为延迟渲染会把这些信息渲染到G-buffer，深度纹理就可以很轻易地访问到；当无法直接获取深度缓存时，深度合法线纹理是通过一个单独的Pass渲染而得的。具体实现是：Unity会使用着色器替换技术选择渲染类型（RenderType）为Opaque的物体，判断它们使用的渲染队列是否小于等于2500（包括Background、Geometry、AlphaTest），如果满足条件，就将其渲染到深度和法线纹理中。总得来说，就是一套在Shader中设置正确的RenderType标签。

深度纹理的精度通常是24位或16位，这取决于使用的深度缓存的精度。如果选择生成一张深度+法线纹理，Unity会创建一张和屏幕分辨率相同的、精度为32位的纹理，其中在观察空间下的法线信息会被编码进纹理R和G通道，而深度信息会被编码进B和A通道。法线信息的获取在延迟渲染中是可以非常容易被找到的，Unity仅仅只需要合并深度和法线缓存即可。在前向渲染中，默认情况下是不会创建发现缓存的，此时Unity底层就是使用了一个单独的Pass把整个场景再次渲染一边来完成。

#### 19.1.2 如何获取

主要是通过depthTextureMode来完成的。

例如我想得到深度纹理，就可以使用：

```c#
camera.depthTextureMode = DepthTextureMode.Depth;
```

这样设置后，就能够在shader中通过声明_CameraDepthTexture变量来访问它。

如果想要获得深度+法线纹理：

```c#
camera.depthTextureMode = DepthTextureMode.DepthNormals;
```

也可以通过组合这些模式，让一个摄像机同时产生一张深度和深度+法线纹理

```c#
camera.depthTextureMode |= DepthTextureMode.Depth;
camera.depthTextureMode |= DepthTextureMode.DepthNormals;
```

当在shader中访问到_CameraDepthTexture或者 _CameraDepthNormalsTexture后，我们就可以使用当前像素的纹理坐标对它进行采样。绝大多数情况我们直接使用tex2D函数采样即可，但对于PS系列，我们就需要使用Unity的内置宏：SAMPLE_DEPTH_TEXTURE来处理由于平台差异造成的问题。使用方法是：

```c#
float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
```

i.uv是一个float2类型的变量，对应当前像素的纹理坐标。

类似的宏还有：SAMPLE_DEPTH_TEXTURE_PORJ和SAMPLE_DEPTH_TEXTURE_LOD。

SAMPLE_DEPTH_TEXTURE_PORJ接受两个参数：深度纹理和一个flaot3或flaot4类型的纹理坐标，其内部采用了tex2Dproj函数进行投影纹理采样，纹理坐标前两个分量首先会除以最后一个分量，再进行纹理采样，如果提供了第四个分量，还会再进行一次比较，通常用于阴影的实现。它的第二个参数通常是由顶点着色器输出插值而得的屏幕坐标，例如：

```c#
float d = SAMPLE_DEPTH_TEXTURE_PORJ(_CameraDepthTexture, UNITY_PROJ_COORD(i.scrPos));
```

i.srcPos是在顶点着色器中通过调用ComputeScreenPos(o.pos)得到的屏幕坐标。

但有个问题，我们这样通过纹理采样得到的深度值，往往是非线性的，这种非线性来自于透视投影使用的裁剪矩阵。但我们计算过程中通常是需要线性的深度值，也就是说我们要把投影后的深度值变换到线性空间下。具体的推导过程我们就不细说了，在Unity中，有两个辅助函数可以帮助我们进行上述的计算过程——LinearEyeDepth和Linear01Depth。

LinearEyeDepth负责把深度纹理的采样结果转换到视角空间下的深度值；Linear01Depth会返回一个范围在[0,1]的线性深度值。这两个函数内部使用了内置的_ZBufferParams变量来得到远近裁剪平面的距离。

如果我们需要获得深度+法线纹理，可以直接使用tex2D函数对_CameraDepthNormalsTexture进行采样，得到里面存储的深度和法线信息。为得到深度值和法线方向，Unity提供了辅助函数来为我们对这个采样结果进行解码——DecodeDepthNormal，它的定义是：

```c#
inline void DecodeDepthNormal(float4 enc, out float depth, out float3 normal){
    depth = DecodeFloatRG(enc.zw);
    normal = DecodeViewNormalStereo(enc);
}
```

### 19.2 使用深度和法线纹理实现运动模糊

在18.6中我们曾说过，实现运动模糊更加广泛的技术是使用速度映射图。在速度映射图中存储了每个像素的速度，然后使用这个速度来决定模糊的方向和大小，速度缓冲的生成有很多种方法，一种方法是将场景中所有物体的速度都渲染到一张纹理中，但这种方法有一个很大的缺点，它需要我们去修改场景中所有物体的shader代码，向其中添加计算速度的代码并输出到一个渲染纹理中。

我们采用另一种方法：利用深度纹理在片元着色器中为每个像素计算其在世界空间下的位置，这是通过使用当前的视角 * 投影矩阵的逆矩阵对NDC下的顶点坐标进行变换得到的。当得到世界空间中的顶点坐标后，我们使用前一帧的视角 * 投影矩阵对其进行变换，得到该位置在前一帧中的NDC坐标。然后我们在计算前一帧与当前帧的位置差，生成该像素的速度。其优点是可以在一个屏幕后处理步骤中完成整个效果的模拟，但缺点是需要在片元着色器中进行两次矩阵乘法操作，会影响性能。

同样依靠一个C#脚本和一个shader来实现：

（1）C#脚本

①依然继承于PostEffectsBase基类

②声明该效果需要的shader及有关变量

```c#
public Shader motionBlurShader;
private Material motionBlurMaterial = null;
public Material material{
    get{
        motionBlurMaterial = CheckShaderAndCreateMaterial(motionBlurShader, motionBlurMaterial);
        return motionBlurMaterial;
    }
}

[Range(0.0f, 1.0f)]
public float blurSize = 0.5f;
```

③由于这种方法需要得到摄像机的视角和投影矩阵，我们需要定义一个Camera类型的变量，以获取摄像机组件

```c#
private Camera myCamera;
public Camera camera{
    get{
        if(myCamera == null){
            myCamera = GetComponent<Camera>();
        }
        return myCamera;
    }
}
```

④定义矩阵变量来保存上一帧摄像机的视角 * 投影矩阵

```c#
private Matrix4x4 previousViewProjectionMatrix; 
```

⑤定义OnEnable函数，并在其中设置摄像机的状态

```c#
void OnEnble(){
    camera.depthTextureMode |= DepthTextureMode.Depth;
}
```

⑥又到了重要的OnRenderImage函数

```c#
void OnRenderImage (RenderTexture src, RenderTexture dest) 
{
	if (material != null) 
    {
		material.SetFloat("_BlurSize", blurSize);

		material.SetMatrix("_PreviousViewProjectionMatrix", previousViewProjectionMatrix);
		Matrix4x4 currentViewProjectionMatrix = camera.projectionMatrix * camera.worldToCameraMatrix;
		Matrix4x4 currentViewProjectionInverseMatrix = currentViewProjectionMatrix.inverse;
		material.SetMatrix("_CurrentViewProjectionInverseMatrix", currentViewProjectionInverseMatrix);
		previousViewProjectionMatrix = currentViewProjectionMatrix;

		Graphics.Blit (src, dest, material);
	} else {
		Graphics.Blit(src, dest);
	}
}
```

通过调用camera.projectionMatrix和camera.worldToCameraMatrix分别得到当前摄像机的投影矩阵和视角矩阵，将它们相乘后求逆，就得到当前帧的视角 * 投影矩阵的逆矩阵，我们再将它传递给材质。我们同时也将取逆前的结果存储在previousViewProjectionMatrix变量中，以便在下一帧时传递给材质的_PreviousViewProjectionMatrix。

（2）shader部分

①声明各个属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white" {}
    _BlurSize ("Blur Size", Float) = 1.0
}
```

你会发现并没有在shader设置两个矩阵的属性，这是因为Unity并没有提供矩阵类型的属性。

②在SubShader中依旧使用CGINCLUDE来组织代码

③声明需要的各个变量

```c#
sampler2D _MainTex;
half4 _MainTex_TexelSize;
sampler2D _CameraDepthTexture;
float4x4 _CurrentViewProjectionInverseMatrix;
float4x4 _PreviousViewProjectionMatrix;
half _BlurSize;
```

_CurrentViewProjectionInverseMatrix和 _PreviousViewProjectionMatrix的值是由脚本传递而来；此外我们声明的 _MainTex_TexelSize，它对应着主纹理的纹素大小，同时也会对深度纹理的采样坐标进行平台差异化处理。

④顶点着色器的代码与之前的没有太大的差别，最主要的是增加了专门用于对深度纹理采样的纹理坐标变量

```c#
struct v2f 
{
	float4 pos : SV_POSITION;
	half2 uv : TEXCOORD0;
	half2 uv_depth : TEXCOORD1;//用于对深度纹理进行采样
};
		
v2f vert(appdata_img v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
			
	o.uv = v.texcoord;
	o.uv_depth = v.texcoord;
	
    //对深度纹理的采样坐标进行平台差异化处理
	#if UNITY_UV_STARTS_AT_TOP//判断是否为DX平台
	if (_MainTex_TexelSize.y < 0)
		o.uv_depth.y = 1 - o.uv_depth.y;//处理因平台差异导致的图像翻转问题
	#endif
					 
	return o;
}
```

⑤片元着色器才是真正的重点和难点

```c#
fixed4 frag(v2f i) : SV_Target 
{
	float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth);
			
	float4 H = float4(i.uv.x * 2 - 1, i.uv.y * 2 - 1, d * 2 - 1, 1);
			
	float4 D = mul(_CurrentViewProjectionInverseMatrix, H);
			
	float4 worldPos = D / D.w;
			
	float4 currentPos = H;
			
	float4 previousPos = mul(_PreviousViewProjectionMatrix, worldPos);
			
	previousPos /= previousPos.w;
			
	float2 velocity = (currentPos.xy - previousPos.xy)/2.0f;
			
	float2 uv = i.uv;
	float4 c = tex2D(_MainTex, uv);
	uv += velocity * _BlurSize;
	for (int it = 1; it < 3; it++, uv += velocity * _BlurSize) {
		float4 currentColor = tex2D(_MainTex, uv);
		c += currentColor;
	}
	c /= 3;
			
	return fixed4(c.rgb, 1.0);
}
```

我们使用内置的SAMPLE_DEPTH_TEXTURE宏和纹理坐标对深度纹理进行采样，得到深度值d。由于深度值是NDC下的坐标映射而来的，所以我们要重新构建像素的NDC坐标，就需要反映射回去，即用映射函数的逆函数：
$$
2*d - 1
$$
NDC的x和y分量同样可以由像素的纹理坐标反映射回去。对得到的NDC下的坐标H，这就是当前帧的坐标，再使用当前帧的视角 * 投影矩阵的逆矩阵对其进行变换，并将结果除以它的w分量来得到世界空间下的坐标表示worldPos。正如前面讲原理时所说，这个坐标是前一帧的世界坐标通过视角 * 投影矩阵变换得到的，所以我们就使用worldPos通过逆矩阵的变换得到前一帧的世界坐标previousPos，与currentPos做差得到速度。再使用该速度对它的左右领域像素进行采样，并将采样值相加后求平均得到一个模糊效果。_BlurSize是用来控制采样距离的。

⑤定义运动模糊所需要的Pass

```c#
Pass
{      
	ZTest Always Cull Off ZWrite Off
			    	
	CGPROGRAM  
			
	#pragma vertex vert  
	#pragma fragment frag  
		  
	ENDCG  
}
```

### 19.3 全局雾效

Unity内置的雾效可以产生基于距离的线性或指数雾效。若想要在自己的着色器中实现这些雾效，我们需要Shader中添加#pragma multi_compile_fog指令，同样还需要使用相关的内置宏，例如UNITY_FOG_GOORDS、UNITY_TRANSFER_FOG、UNITY_APPLY_FOG。这样做虽然可以实现雾化效果，但效果有限并且实现很麻烦。若想实现基于高度的雾效等效果，就要使用基于屏幕后处理的全局雾效的实现。

基于屏幕后处理的全局雾效的关键是：根据深度纹理来重建每个像素在世界空间下的位置。这有点像运动模糊中的方法，但我们在运动模糊中是将这种变换放在片元着色器中的，这对性能的消耗有点大，所以我们将采用另一种快速从深度纹理中重建世界坐标的方法：首先对图像空间下的视锥体射线进行插值，其存储了该像素在世界空间下到摄像机的方向信息；然后我们将该射线和线性化后的视角空间下的深度值相乘，再加上摄像机的世界位置，就可以得到该像素在世界空间下的位置。

#### 19.3.1 重建世界坐标

先得到世界空间下的位置，以及世界空间下该像素相对于摄像机的偏移量，两者相加就可以得到该像素的世界坐标

```c#
float4 worldPos = _WorldSpaceCameraPos + linearDepth * interpolateRay;
```

_WorldSpaceCameraPos是世界空间下摄像机的位置，可以由Unity的内置变量直接访问而得；linearDepth是由深度纹理得到的线性深度值；interpolateRay是由顶点着色器输出并插值后得到的射线，它包括了该像素到摄像机的方向，也包含了距离信息。这里需要我们求解的就是interpolateRay。

interpolateRay来源于对近裁剪平面的四个角的某个特定向量的插值，这四个向量可以利用摄像机的近裁剪平面距离、FOV、横纵比计算而得，包括了摄像机的方向和距离信息。

需要一些辅助向量来进行计算：例如向量toTop和toRight——起点位于近裁剪平面中心、分别指向摄像机正上方和正右方的向量，注意，是摄像机的正上和正右：
$$
\begin{align}
hlafHeight &= Near * tan(\frac{FOV}{2})\\
toTop &= camera.up * hlafHeight\\
aspect &= \frac{halfLength}{halfHeight}\\
toRight &= camera.right * hlafHeight.aspect
\end{align}
$$
通过这两个辅助变量就可以计算4个角相对摄像机的方向了(即摄像机指向四个角的方向，模长为近裁剪平面四个顶点到摄像机的距离)：
$$
\begin{align}
TL &= camera.forward \cdot Near + toTop - toRight\\
TR &= camera.forward \cdot Near + toTop + toRight\\
BL &= camera.forward \cdot Near - toTop + toRight\\
BR &= camera.forward \cdot Near - toTop - toRight\\
\end{align}
$$
由于我们之前得到的线性深度值并不是顶点到摄像机的欧氏距离，所以我们先将其转换成到摄像机的欧氏距离才能与4个角的单位方向的乘积来计算它们到摄像机的偏移量。欧氏距离的计算我们以TL为例：
$$
\begin{align}
\frac{depth}{distance} &= \frac{Near}{|TL|} 相似三角形\\
dist &= \frac{TL}{Near}\\
由于四个的模长相等，所以我们设 scale &= \frac{TL}{Near}\\
就有Ray_{TL} &= \frac{TL}{|TL|}*scale & Ray_{TR} = \frac{TR}{|TR|}*scale\\
Ray_{BL} &= \frac{BL}{|BL|}*scale & Ray_{BR} = \frac{BR}{|BR|}*scale
\end{align}
$$

#### 19.3.2 雾的计算

我们需要一个雾效系数f，作为混合原始颜色和雾的混合系数：

```c#
float3 afterFog = f * fogColor + (1-f) * origColor;
```

由于Unity内置的雾效的实现中支持三种雾的计算方式：线性（Linear）、指数（Exponential）、指数的平方（Exponential Squared）。当给定距离z后，f的计算公式分别如下：

①Linear：
$$
f = \frac{d_{max}-|z|}{d_{max}-d_{min}}\\
d_{min}和d_{max}分别表示受雾影响的最小和最大距离
$$
②Exponential：
$$
f = e^{-d\cdot |z|}\\
d是控制雾的浓度的参数
$$
③Exponential Squared：
$$
f = e^{-(d-|z|)^2}\\
d是控制雾的浓度的参数
$$
我们采用基于线性雾的计算公式——基于高度，其公式是：
$$
f = \frac{H_{end}-y}{H_{end}-H_{start}}\\
H_{start}和H_{end}分别表示受雾影响的起始高度和终止高度
$$

#### 19.3.3 实现

（1）C#脚本

①依然是基于PostEffectsBase基类

②声明shader

```c#
public Shader fogShader;
private Material fogMaterial = null;

public Material material {
    get{
        fogMaterial = CheckShaderAndCreateMaterial(fogShader, fogMaterial);
        return fogMaterial;
    }
}
```

③获取摄像机的Camera组件和Transform组件：

```c#
private Camera myCamera;
public Camera camera {
    get{
        if(myCamera == null){
            myCamera = GetComponent<Camera>();
        }
        return myCamera;
    }
}

private Transform myCameraTransform;
public Transform cameraTransform{
    get{
        if(myCameraTransform == null){
            myCameraTransform = camera.transform;
        }
        return myCameraTransform;
    }
}
```

④定义模拟雾效时使用的各个参数

```c#
[Range(0.0f, 3.0f)]
public float fogDensity = 1.0f;

public Color fogColor = Color.white;

public float fogStart = 0.0f;
public float fogEnd = 2.0f;
```

fogDensity用于控制雾的浓度；fogColor用于控制雾的颜色；fogStart和fogEnd用于控制雾效的起始和终止高度。

⑤OnEnble函数中设置摄像机的相应状态

```c#
void OnEnble(){
    camera.depthTextureMode |= DepthTextureMode.Depth;
}
```

⑥OnRenderImage函数

```c#
void OnRenderImage (RnederTexture src, RenderTexture dest) {
    if(material != null) {
        Matrix4x4 frustumCorners = Matrix4x4.identity;
        
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
        
        material.SetMatrix("_FrustumCornersRay", frustumCorners);

		material.SetFloat("_FogDensity", fogDensity);
		material.SetColor("_FogColor", fogColor);
		material.SetFloat("_FogStart", fogStart);
		material.SetFloat("_FogEnd", fogEnd);
        
        Graphics.Blit(src, dest, material);
    }else{
        Graphics.Blit(src, dest);
    }
}
```

代码很好理解：先计算近裁剪平面四个角对应的向量，并将它们存储在一个矩阵类型的变量中。

（2）shader实现

①声明各个属性

```c#
Properties{
    _MainTex ("Base (RGB)", 2D) = "white"{}
    _FogDensity ("Fog Density", Float) = 1.0
    _FogColor ("Fog Color", Color) = (1,1,1,1)
    _FogStart ("Fog Start", Float) = 0.0
    _FogEnd ("Fog End", Float) = 1.0
}
```

②使用CGINCLUDE来组织代码

③声明需要的变量

```c#
float4x4 _FrustumCornersRay;
		
sampler2D _MainTex;
half4 _MainTex_TexelSize;
sampler2D _CameraDepthTexture;
half _FogDensity;
fixed4 _FogColor;
float _FogStart;
float _FogEnd;
```

④定义顶点着色器

```c#
struct v2f {
    float4 pos : SV_POSTION;
    half2 uv : TEXCOORD0;
    half2 uv_depth : TEXCOORD1;
    float4 interpolatedRay : TEXCOORD2;
};

v2f vert(appdata_img v){
    v2f o;
    o.pos = UnityObjectToClipPos(v.vertex);
    
    o.uv = v.texcoord;
    o.uv_depth = v.texcoord;
    
    #if UNITY_UV_STARTS_AT_TOP
        if(_MainTex_TexelSize.y < 0)
            o.uv_depth.y = 1-o.uv_depth.y;
    #endif
        
    int index = 0;
    if (v.texcoord.x < 0.5 && v.texcoord.y < 0.5){
        index = 0;
    }else if (v.texcoord.x > 0.5 && v.texcoord.y < 0.5){
        index = 1;
    }else if (v.texcoord.x > 0.5 && v.texcoord.y > 0.5){
        index = 2;
    }else{
        index = 3;
    }
    
    #if UNITY_UV_STARTS_AT_TOP
        if(_MainTex_TexelSize.y < 0)
           index = 3- index;
    #endif
	
    o.interpolatedRay = _FrustumCornersRay[index];
    
    return o;
}
```

在v2f结构体中我们额外声明了一个interpolatedRay变量，用于存储插值后的像素向量。在顶点着色器中我们还对深度纹理的采样坐标进行了平台差异化处理。此外我们通过判断它们的纹理值来判断该点的索引这个是于脚本中frustumCorners的赋值顺序是一致的。

⑤定义片元着色器来产生雾效

```c#
fixed4 frag(v2f i) : SV_Target {
    float linearDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth));
    float3 worldPos = _WorldSpaceCameraPos + linearDepth * i.interpolatedRay.xyz;
    
    float fogDensity = (_FogEnd - worldPos.y) / (_FogEnd - _FogStart);
    fogDensity = saturate(fogDensity * _FogDensity);
    
    fixed4 finalColor = tex2D(_MainTex, i.uv);
    finalColor.rgb = lerp(finalColor.rgb, _FogColor.rgb, fogDensity);
    
    return finalColor;
}
```

首先我们要重建该像素在世界空间中的个位置，所以先用SAMPLE_DEPTH_TEXTURE对深度纹理进行采样，再使用LinearEyeDepth对采样结果处理得到视角空间下的线性深度值。之后再与interpolatedRay相乘后与世界空间下的摄像机位置相加即可得到世界空间下的位置。

之后的就是利用之前所说的公式来进行雾效的模拟。

⑥定义雾效渲染所需要的Pass

```C#
Pass{
    ZTest Always Cull Off Zwrite Off
    
    CGPROGRAM
        
    #pragma vertex vert
    #pragma fragment frag
    
    ENDCG
}
```

### 19.4 使用深度和法线纹理实现边缘检测

前面我们是直接利用颜色信息进行边缘检测，使用Sobel算子对屏幕图像进行边缘检测，实现描边效果，但这种方法会产生一些额外的、我们所不期望的边缘线。（虽然这种效果比较好看）

这里我们将通过深度和法线纹理重新实现边缘检测，这种检测是不会受纹理和光照的影响的，检测出的边缘也更加可靠。

这里我们将是使用Roberts算子。

（1）基于PostEffectsBase的EdgeDetectNormalsAndDepth脚本

①声明使用该效果的shader

```c#
public Shader edgeDetectShader;
private Material edgeDetectMaterial = null;
public Material material {
    get{
        edgeDetectMaterial = CheckShaderAndCreateMaterial(edgeDetectShader, edgeDetectMaterial);
        return edgeDetectMaterial;
    }
}
```

②声明用于调整边缘线强度描边颜色以及背景颜色的参数，同时添加用于控制采样距离以及对深度和法线进行边缘检测时的灵敏度参数

```c#
[Range(0.0f, 1.0f)]
public float edgesOnly = 0.0f;
public Color edgeColor = Color.black;
public Color backgroundColor = Color.white;
public float sampleDistance = 1.0f;
public float sensitivityDepth = 1.0f;
public float sensitivityNormals = 1.0f;
```

sampleDistance用于控制对深度+法线纹理采样时使用的采样距离，从视觉上看，值越大，描边越宽。sensitivityDepth和sensitivityNormals将会影响当邻域的深度值或法线值相差多少是会被认为存在边界线，灵敏度越大，那么深度或法线上很小的变化也会被认为存在一条边界线。

③OnEnable函数

```c#
void OnEnable(){
    GetComponent<Camera>().depthTextureMode |= DepthTextureMode.DepthNormals;
}
```

④OnRenderImage函数

```c#
[ImageEffectOpaque]
void OnRenderImage (RenderTexture src, RenderTexture dest){
    if(material != null){
        material.SetFloat("_EdgeOnly", edgesOnly);
		material.SetColor("_EdgeColor", edgeColor);
		material.SetColor("_BackgroundColor", backgroundColor);
		material.SetFloat("_SampleDistance", sampleDistance);
		material.SetVector("_Sensitivity", new Vector4(sensitivityNormals, sensitivityDepth, 0.0f, 0.0f));

		Graphics.Blit(src, dest, material);
    }else{
        Graphics.Blit(src, dest);
    }
}
```

ImageEffectOpaque在之前的内容提到过。在默认情况下，OnRenderImage函数会在所有不透明和透明的Pass执行完毕后被调用，以便对场景所有的物体都产生影响，但有时我们也希望在不透明的Pass（渲染队列小于2500，包括Background、Geometry、AlphaTest）执行完毕后立即调用该函数，而不对透明物体（渲染队列为Transparent的Pass）产生影响，所以我们添加了ImageEffectOpaque属性来实现这样的目的。

（2）EdgeDetectNormalsAndDepth.shader

①声明相关属性

```c#
Properties 
{
	_MainTex ("Base (RGB)", 2D) = "white" {}
	_EdgeOnly ("Edge Only", Float) = 1.0
	_EdgeColor ("Edge Color", Color) = (0, 0, 0, 1)
	_BackgroundColor ("Background Color", Color) = (1, 1, 1, 1)
	_SampleDistance ("Sample Distance", Float) = 1.0
	_Sensitivity ("Sensitivity", Vector) = (1, 1, 1, 1)
}
```

_Sensitivity的xy分量分别对应法线和深度的检测灵敏度，zw分量则没有实际用途。

②依旧使用CGINCLUDE赖组织代码

③声明有关变量

```c#
sampler2D _MainTex;
half4 _MainTex_TexelSize;
fixed _EdgeOnly;
fixed4 _EdgeColor;
fixed4 _BackgroundColor;
float _SampleDistance;
half4 _Sensitivity;
```

④定义顶点着色器

```c#
struct v2f 
{
	float4 pos : SV_POSITION;
	half2 uv[5]: TEXCOORD0;
};
		  
v2f vert(appdata_img v) 
{
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
			
	half2 uv = v.texcoord;
	o.uv[0] = uv;
			
	#if UNITY_UV_STARTS_AT_TOP
		if (_MainTex_TexelSize.y < 0)
			uv.y = 1 - uv.y;
	#endif
			
	o.uv[1] = uv + _MainTex_TexelSize.xy * half2(1,1) * _SampleDistance;
	o.uv[2] = uv + _MainTex_TexelSize.xy * half2(-1,-1) * _SampleDistance;
	o.uv[3] = uv + _MainTex_TexelSize.xy * half2(-1,1) * _SampleDistance;
	o.uv[4] = uv + _MainTex_TexelSize.xy * half2(1,-1) * _SampleDistance;
					 
	return o;
}
```

在v2f结构体中我们定义一个5维的纹理坐标数组，第一个坐标存储了屏幕颜色图像的采样纹理。剩余的4个坐标则存储使用Roberts算子时需要采样的纹理坐标。其中还进行了平台差异化处理。我们在顶点着色器中计算采样纹理坐标是因为可以减少运算并提高性能，而且不影响计算结果（从顶点着色器到片元着色器的插值是线性的）

⑤片元着色器

```c#
fixed4 fragRobertsCrossDepthAndNormal(v2f i) : SV_Target
{
	half4 sample1 = tex2D(_CameraDepthNormalsTexture, i.uv[1]);
	half4 sample2 = tex2D(_CameraDepthNormalsTexture, i.uv[2]);
	half4 sample3 = tex2D(_CameraDepthNormalsTexture, i.uv[3]);
	half4 sample4 = tex2D(_CameraDepthNormalsTexture, i.uv[4]);
			
	half edge = 1.0;
			
	edge *= CheckSame(sample1, sample2);
	edge *= CheckSame(sample3, sample4);
			
	fixed4 withEdgeColor = lerp(_EdgeColor, tex2D(_MainTex, i.uv[0]), edge);
	fixed4 onlyEdgeColor = lerp(_EdgeColor, _BackgroundColor, edge);
			
	return lerp(withEdgeColor, onlyEdgeColor, _EdgeOnly);
}
```

先使用4个纹理坐标对深度+法线纹理进行采样，再调用CheckSame函数来分别计算对角线上两个纹理值的差值。

CheckSame函数的返回值要么是0，要么是1，返回0表明这两个点之间存在一条边界，反之则返回1.

⑥CheckSame函数

```c#
half CheckSame(half4 center, half4 sample) 
{
	half2 centerNormal = center.xy;
	float centerDepth = DecodeFloatRG(center.zw);
	half2 sampleNormal = sample.xy;
	float sampleDepth = DecodeFloatRG(sample.zw);
			
	half2 diffNormal = abs(centerNormal - sampleNormal) * _Sensitivity.x;
	int isSameNormal = (diffNormal.x + diffNormal.y) < 0.1;
	
	float diffDepth = abs(centerDepth - sampleDepth) * _Sensitivity.y;
	
	int isSameDepth = diffDepth < 0.1 * centerDepth;
			
	return isSameNormal * isSameDepth ? 1.0 : 0.0;
}
```

函数先提取两个参数的发现和深度值，再按照代码中所示的步骤进行计算比较。

⑦关闭Fallback之前定义边缘检测所要使用的Pass

```c#
Pass 
{ 
	ZTest Always Cull Off ZWrite Off
			
	CGPROGRAM      
			
	#pragma vertex vert  
	#pragma fragment fragRobertsCrossDepthAndNormal
			
	ENDCG  
}
```

