[TOC]



# Unity-shader学习笔记（二）

## 7 法线变换

有法线就必定有切线，都是模型顶点携带的一种信息。既然如此，那么法线的变换能不能直接使用顶点变换的矩阵呢？

我们先来看切线：切线是两个顶点之间的差值计算得到的，那么由于又不考虑平移变换，就可以直接使用3*3的变换顶点的变换矩阵来变换切线，于是就有：
$$
T_{B} = M_{A\rightarrow B}T_{A}
$$
我们知道切线与法线是相互垂直的，那么，法线变换也用顶点矩阵变换会发生什么？

假如变换矩阵相同，设一个三角形的三个顶点分别为：(0,0)(1,0)(0,1)，进行(1,2,1)的非统一缩放后，变为(0,0)(1,0)(0,2)，斜边向量由(-1,1)变换到(-1,2)，法线就由(1,1)变换到(1,2)，变换后的切线与法线的乘积明显不为0。

那怎么求取法线变换矩阵？利用切线与法线的乘积为0的性质，如下：
$$
\begin{align}
设法线为N_{A}，切线为T_{A}，法线变换矩阵为G\\
N_{A}\cdot T_{A} &= 0\\
T_{B} &= M_{A\rightarrow B}T_{A}\\
则：T_{B}\cdot N_{B} &= (M_{A\rightarrow B}T_{A})\cdot (GN_{A}) = 0\\
(M_{A\rightarrow B}T_{A})^{T}\cdot (GN_{A}) &= 0\\
T_{A}^{T}M_{A\rightarrow B}^{T}GN_{A} &= 0\\
T_{A}^{T}(M_{A\rightarrow B}^{T}G)N_{A} &= 0\\
又因为N_{A}\cdot T_{A} &= 0 = N_{A}^{T}\cdot T_{A}\\
所以只有当M_{A\rightarrow B}^{T}G &= I时，成立\\
即G &= (M_{A\rightarrow B}^{T})^{-1} = (M_{A\rightarrow B}^{-1})^{T}
\end{align}
$$
此时就需要分情况讨论了：

①如果变换只包含旋转变换，那这个变换矩阵就是正交矩阵，根据正交矩阵的转置等于正交矩阵的逆，此时：
$$
G =M_{A\rightarrow B}
$$
②如果变换包含了旋转和统一缩放，引入一个统一缩放系数K，来得到变换矩阵，此时：
$$
G = \frac{1}{k}M_{A\rightarrow B}
$$
③如果包含的缩放变换时非统一缩放变换，就只能通过求解逆矩阵来得到变换法线的矩阵。

## 8 Unity Shader的内置变量

### 8.1  内置变换矩阵

|         变量名         |                             描述                             |
| :--------------------: | :----------------------------------------------------------: |
|  **UNITY_MATRIX_MVP**  |       **用于将顶点/方向矢量从模型空间变换到裁剪空间**        |
|  **UNITY_MATRIX_MV**   |       **用于将顶点/方向矢量从模型空间变换到观察空间**        |
|   **UNITY_MATRIX_V**   |       **用于将顶点/方向矢量从世界空间变换到观察空间**        |
|   **UNITY_MATRIX_P**   |       **用于将顶点/方向矢量从观察空间变换到裁剪空间**        |
|  **UNITY_MATRIX_VP**   |       **用于将顶点/方向矢量从世界空间变换到裁剪空间**        |
| **UNITY_MATRIX_T_MV**  |                 **求UNITY_MATRIX_MV的转置**                  |
| **UNITY_MATRIX_IT_MV** | **求UNITY_MATRIX_MV的逆转置，用于将法线从模型空间变换到观察空间，也可以用于求UNITY_MATRIX_MV的逆矩阵** |
|   **_Object2World**    | **当前的模型矩阵，用于将顶点/方向矢量从模型空间变换到世界空间** |
|   **_World2Object**    | **求_Object2World的逆矩阵，用于将顶点/方向矢量从世界空间变换到模型空间** |

对于第六种，当是UNITY_MATRIX_MV正交矩阵时，它的转置等于它的逆，即UNITY_MATRIX_T_MV也可以用于将法线从观察空间变换到模型空间。

如何判断UNITY_MATRIX_MV是不是一个正交矩阵？

①变换只包括旋转，那么一定是正交矩阵；

②变换包括旋转和缩放系数为k的统一缩放，那么勉强算作是正交矩阵，UNITY_MATRIX_MV的逆矩阵是(1/k)UNITY_MATRIX_MV；

③由于平移对方向矢量没有任何的影响，因此可以取UNITY_MATRIX_MV的前三行三列来将方向矢量从观察空间变换到模型空间（前提是又存在旋转或/和统一缩放）

注意：以上矩阵参数的逆矩阵就是反向变换。

然后你就会发现，我们对UNITY_MATRIX_IT_MV求转置，就会得到UNITY_MATRIX_MV的逆矩阵。所以如果我们要将顶点或方向矢量从观察空间变换到模型空间就有两种方法：求UNITY_MATRIX_MV的逆矩阵或者求UNITY_MATRIX_IT_MV的转置。

### 8.2 摄像机和屏幕参数

|               变量名               |     类型     |                             描述                             |
| :--------------------------------: | :----------: | :----------------------------------------------------------: |
|      **_WorldSpaceCameraPos**      |  **float3**  |               **获取摄像机在世界空间中的位置**               |
|       **_ProjectionParams**        |  **float4**  | **x = 1.0/-1.0,y = Near,z = Far,w = 1.0+1.0/Far;其中Near和Far分别是近远裁剪平面和摄像机的距离** |
|         **_ScreenParams**          |  **float4**  | **x = width,y = height,z = 1.0+1.0/width,w = 1.0+1.0/height;其中width和height分别是该摄像机的渲染目标的像素宽高** |
|         **_ZBufferParams**         |  **float4**  | **x = 1-Far/Near,y = Far/Near,z = x/Far,w = y/Far;用于线性化Z缓存中的深度值** |
|       **unity_OrthoParams**        |  **float4**  | **x = width,y = height,w = 1.0(该摄像机是正交摄像机)/0.0(该摄像机是透视摄像机)'其中width和height是正交投影摄像机的宽高** |
|     **unity_CameraProjection**     | **float4*4** |                    **该摄像机的投影矩阵**                    |
|   **unity_CameraInvProjection**    | **float4*4** |                  **该摄像机的投影矩阵的逆**                  |
| **unity_CameraWorldClipPlanes[6]** |  **float4**  | **该摄像机的6个裁剪平面在世界空间下的等式，按左右下上近远的顺序** |

### 8.3 Unity中的屏幕坐标

在顶点着色器或片元着色器中，有两种方式来获得片元的屏幕坐标。

①在片元着色器的输入中声明VPOS或WPOS语义，就不需要自己定义输入输出的数据结构：

```c#
fixed4 frag(float4 sp : VPOS) : SV_Target{
    //用屏幕坐标除以屏幕分辨率_ScreenParams.xy，得到视口空间中的坐标
    return fixed(sp.xy/_ScreenParams.xy,0.0,1.0);
}
```

VPOS/WPOS语义定义的输入是一个float4类型的变量。

shader实例：

```c#
SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
			struct appdata
			{
				float4 pos : POSITION;
			};

			struct v2f
			{
				float4 pos : SV_POSITION;
			};

			v2f vert(appdata v)
			{
				v2f o;
				o.pos = UnityObjectToClipPos(v.pos);
				return o;
			}

			fixed4 frag(v2f i : WPOS) : SV_Target
			{
				fixed2 viewPos = i.pos.xy / _ScreenParams.xy;
				return fixed4(viewPos, 0, 1);
			}
			ENDCG
        }
    }
```

②通过ComputeScreenPos函数：通常需要两个步骤，首先在顶点着色器中将ComputeScreenPos的结果输出在结构体中，然后在片元着色器中进行一个齐次除法运算后得到视口空间下的坐标：

```c#
struct vertOut{
    float4 pos : SV_POSITION;
    float4 scrPos : TEXCOORD0;
};

vertOut vert(appdata_base v){
    vertOut o;
    o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
    o.scrPos = ComputeScreenPos(o.pos);
    return o;
}

fixed4 frag(vertOut i) : SV_Target{
    float2 wcoord = (i.scrPos.xy/i.scrPos.w);
    return fixed4(wcoord, 0.0, 1.0);
}
```

注意：ComputeScreenPos函数是定义在UnityCG.cginc里的，所以一定要带有这个头文件。

ComputeScreenPos输入的参数pos是通过MVP变换后在裁剪空间中的顶点坐标，通过查看ComputeScreenPos源码，推导出来发现实际输出的是：
$$
\begin{align}
Output_{x} &= \frac{clip_{x}}{2}+\frac{clip_{w}}{2}\\
Output_{y} &= \frac{clip_{y}}{2}+\frac{clip_{w}}{2}\\
Output_{z} &= clip_{z}\\
Output_{z} &= clip_{z}
\end{align}
$$
就是说输出的坐标是包含了w分量的，所以说在片元着色器中还要除以裁剪坐标的w分量。你可能就会有疑问，为什么不直接在顶点着色器中进行齐次除法。

回想顶点着色器和片元着色器，从前者到后者，中间有一个插值的过程，如果我们直接在顶点着色器中进行这个除法，插值对象就是x/w和y/w，想对于原本的对象x和y，插值结果就会不正确。（这里也得出一个结论：不能够在投影空间中进行插值，因为插值是线性的，而投影空间并不是一个线性空间）