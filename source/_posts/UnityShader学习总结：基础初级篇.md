---
title: UnityShader学习总结：基础初级篇
date: 2023-10-07 19:58:04
tags: [Unity,Shader,图形渲染]
categories: 图形学
copyright_author: 墨墨辰
copyright_author_href: https://the-black-sun.github.io/
copyright_url: https://the-black-sun.github.io/2023/10/07/UnityShader学习总结：基础初级篇/
cover: https://the-black-sun-imgs.pages.dev/Img/1最初的梦想.jpg
---
# 前言

上个月读了《Unity Shader 入门精要》这本书，原本准备将学习笔记整理一下，作为博文发布出来，复习的时候检查了一下。发现很大程度上与书籍内容相同，有点抄书的嫌疑，想了想还是决定整理归纳一下，顺便查缺补漏，看看有没有被遗落的知识点，内容极其主观，特此声明。

---

# 基础部分：渲染流水线与Unity Sahder基础

这本书的基础部分主要是由渲染流水线的介绍和Unity Shader的基础写法构成的。这部分内容主要分布在全书的第2章与第3章。第一章主要介绍了一下全书内容结构，并没有涉及到太多的知识点。第4章主要介绍一些涉及图形学相关的数学知识，关于这部分内容，想要深入学习的同学可以去阅读一下相关书籍《3D游戏数学》。这本书详细介绍了3d游戏中，涉及图形学的数学概念，各个空间变换所需要的矩阵计算方法的底层原理等等。

## 渲染流水线

由于在大学期间学习过龙书（****DirectX12 3D游戏开发实战****），还跟着老师写过论文（混），所以对渲染流水线还是有着基本上的认识的，虽然DirectX12的渲染流水线与Unity的渲染存在一定程度上的差别，但是基础原理与专有名词还是相同的，对理解起来不会存在太大的难度。

在unity的渲染流水线中主要分为以下3个阶段：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled.png)

分别是应用阶段，几何阶段以及光栅化阶段。其中应用阶段主要由CPU进行控制的，而几何阶段以及光栅化阶段则主要由GPU进行控制。

`应用阶段`是CPU的工作阶段，由上图可以看出主要是负责将渲染图元（几何信息，顶点数据）提交给GPU并请求渲染。具体的工作这是：

- 准备好场景数据（摄像机位置、模型、光源等等），进行粗粒度剔除的工作（根据视锥体剔除不可见物体），将不可见物体CPU首先需要将所需要的数据加载进显存当中（大多数显卡缺少对内存的访问权限）；
- 设置好渲染状态（告诉GPU这个渲染物体使用的材质，纹理，shader等等），会得到一个待渲染的几何信息，也就是`渲染图元`，在待渲染的图元列表中；
- 向GPU发送请求渲染命令，指向一个待渲染的图元列表，通知GPU进行渲染。

这一部分的大部分工作能够在Unity的图形化界面上进行操作，包括但不限于：调整场景中物体（顶点，法线）位置，设置纹理（颜色、透明度调整）等等⇒部分纹理属性需要再在shader中定义对应属性。除此之外还有一部分需要再unity的Shader文件中进行设置，例如渲染状态等等。而CPU每次提交渲染图元，向GPU发出的一个请求渲染的命令就是通常开发者们熟悉的`DrawCall`。而通常对DrawCall的优化就是需要减少CPU发出的渲染命令，也就是减少图元列表，这通常也就需要使用到我们常说的`合批`与`图集`了，这里先不进行表述。

`几何阶段`与`光栅化阶段`也就是我们需要重点关注的部分。内部可以细分成多个阶段，如下表所示：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled1.png)

`几何阶段`主要包括以下几个步骤：

顶点着色器：完全可编程。以单个顶点为单位进行处理，主要完成顶点坐标变换，逐顶点光照，计算顶点颜色等功能。

曲面细分着色器：完全可编程，可选。主要用来设置与生成高精度网格（细分图元）。

几何着色器：完全可编程，可选。进行逐图元的着色操作，或者产生更多的图元。

剪裁：可配置。（注意与CPU的粗粒度剔除的区别）将摄像机视锥体范围外的`物体部分`剔除（产生新顶点替代边界）。

屏幕映射：不可配置，不可编程。将图元坐标由齐次剪裁空间转化到屏幕坐标系中。

`光栅化阶段`主要包括以下几个步骤：

三角形设置：不可配置，不可编程。根据几何阶段输出的顶点信息，获取三角形每条边的像素坐标，得到三角形边界的表达方式。

三角形遍历：不可配置，不可编程。检查像素点是否被三角网格覆盖，并生成片元（包括屏幕坐标，深度，发现，纹理等信息）。

![Untitled](../UnityShader学习总结：基础初级篇/Untitled2.png)

片元着色器：完全可编程。进行纹理采样，实现逐片元的着色操作。

逐片元操作：高度可配置。进行颜色修改，可见性测试（深度检测，模板检测），混合等操作。

- 模板检测：将片元模板缓冲区的模板值与设置的参考值比较，比较函数由开发者指定，判断是否通过模板测试。
- 深度检测：将片元深度缓冲区的深度值与片元的深度值比较，比较函数由开发者指定，判断是否通过深度测试。如果通过深度测试，才有覆盖深度缓冲区值的权利，开发者可以选择是否覆盖（开启/关闭深度写入）
- 等等等等

由上面介绍，可以看见有关于于`完全可编程`，`可配置`等等的字样。这些说明了开发者能够控制内部实现又或是能够配置对应阶段的相关属性，并且这些都是需要再Unity Shader文件中通过代码进行实现。在书中，后面章节主要通过使用了顶点着色器与片元着色器以及部分配置功能来实现效果，至于曲面细分着色器与几何着色器的使用，有兴趣的可以参考官网内容进行学习。

UnityShader文件主要通过类CG语言进行编写，由以上介绍可知，我们能够在顶点着色器，曲面细分着色器，几何着色器以及片元着色器中进行具体的效果实现，同时可以配置剪裁或逐片元操作的混合、深度写入等功能。

书中该部分内容还介绍了两大图形接口（OpenGL与DirectX），三大着色语言（HLSL、GLSL、CG）DrawCall，以及固定渲染管线的相关知识，与具体实现Shader相关效果联系不大，这里不做过多的介绍。

## Unity Shader基础

该部分内容主要是书中第3章部分内容，主要介绍了UnityShader的整体结构。需要注意的是Shader是和材质相互绑定的，Unity中创建材质会自动绑定默认Shader，可以通过Unity界面上进行调整。

常见的UnityShader使用流程：

- 创建一个材质
- 创建一个unity shader，并赋给材质
- 将材质赋给要渲染的对象
- 在材质面板调整Unity Shader中的属性

需要注意的是，还存在其他情况的Shader使用情况，例如涉及到屏幕截取以及后处理等功能时（第12章），会通过代码创建材质并将相应Shader进行材质设置，不遵照上述使用流程。

除此之外，创建UnityShader文件还需要注意以下几种Unity的内置模板：

- Standard Surface Shader：一个包含标准光照模型的表面着色器模板。
- Unlit Shader：一个不包含光效，但是包含雾效的基本的顶点/片元着色器
- Image Effect Shader：一个用来处理各种屏幕后处理效果的模板
- Compute Shader（例外）：利用GPU并行性进行一些与流水线无关计算的模板（白嫖计算量）
- Ray Tracing Shader：一个光线追踪shader

以下是2个常用Shader着色器的特点：

- 表面着色器：代码量少，渲染代价大（unity实现的时候需要转换成顶点/片元着色器，相当于上层），unity内部自动处理光照细节
    - 不需要写Pass，最表层接口不关心（只实现什么纹理填充什么颜色，法线，使用什么光照模型等，进行表面渲染）
    - 代码定义在CGPROGRAM与ENDCG之间
    - 使用CG/HLSL语法
    - 与光源打交道可以优先使用
- 顶点/片元着色器：更加复杂，灵活性高
    - 代码定义在CGPROGRAM与ENDCG之间
    - 需要写在Pass语义块中，需要自定义每个Pass使用的shader代码
    - 使用CG/HLSL语法
    - 需要许多自定义的渲染效果
    - 和较少的光源打交道

在之后的学习的过程中主要采用创建使用了Unlit Shader模版（通过编写顶点/片元着色器实现具体特效效果，包括下列代码实例也是）。还有可能使用的Shader面板的相关设置：

- Default Maps：shader使用的默认纹理
- Show generated code：unity为shader模板（我们创建的）生成的顶点/片元着色器代码。
- Compile and show code（下拉列表）：直接点击查看底层汇编指令。下拉列表查看不同图形接口生成编译的shader代码。
- 其余信息：渲染队列（Render queue）情况，批处理（Disable batching）情况、属性列表等

接下来就是Unity Shader代码文件的大概结构了：

``` glsl
//Shader名称，在跨Shader文件使用的时候会用到
Shader "ShaderName"
{
    Properties
    {
		//属性，定义纹理，颜色，透明度等纹理相关属性，
		//会在Unity材质面板上进行显示，根据设置能够调整对应参数
		//名字|显示的名称|类型|赋值
        //_Color ("Color", Color) = (1,1,1,1)
        //_MainTex ("Albedo (RGB)", 2D) = "white" {}
    }

	//显卡A使用的子着色器（顶点/片元），在内部定义
    SubShader
    {
		//可选标签，控制整个SubShader
		//设置渲染对列，渲染类型
		//[Tags]
		//Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		//可选状态，控制整个SubShader
		//设置剔除模式，深度测试函数时，深度写入开关，以及混合模式
		//[RenderSetup]
		//Cull Back

		//结构,Pass可以定义多个
		Pass{
			[Name]
			[Tags]
			[RenderSetup]
			//Other code 
		}
				
		//实际写法
		Pass{
			//名称，ShaderLab可以使用UsePass命令来使用其他shader的pass。
			//跨Shader文件调用时会用到
			Name "MyPassName"

			//UsePass使用方法，使用其他shader的pass。
			UsePass "MyShader/MYPASSNAME"

			//GrabPass使用方法，用来抓取屏幕并存储成一张纹理，用于后续的Pass处理。

			//标签,这里的标签与Subshader不同，是用来设置怎么渲染的
			Tags {"LightMode"="ForwardBase"}
			//定义pass在渲染流水线中的角色

			//状态，除了SubShader的状态之外，还有设置固定管线的着色器命令（3.4.3）
			//关闭深度写入
			//ZWrite Off
			//设置透明混合命令
			//Blend SrcAlpha OneMinusSrcAlpha

//CG代码片段声明						
CGPROGRAM
			//定义顶点着色器与片元着色器
			//#pragma vertex vert
			//#pragma fragment frag
						
			//调用unityShader内置文件，包含变量与函数
			//#include "Lighting.cginc"

			//属性值获取，之前定义的属性信息存储的位置
			//fixed4 _Color;
			//sampler2D _MainTex;
			//float4 _MainTex_ST;
						
			//输入顶点着色器结构体，根据具体需要声明，顶点，法线，纹理等信息
			struct a2v {
				....
			};
			//顶点着色器输出结构体，也是片元着色器输入结构体，神明世界空间顶点位置，法线，纹理
			//甚至切线空间与世界空间的变换矩阵等
			struct v2f {
				....
			};

			v2f vert(a2v v) {
				v2f o;
				...
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				...
				//返回最后的合成颜色
				//return ...;
			}
ENDCG
		}
    }

		//显卡B
	SubShader
    {
		....
    }
    
	//FallBack "Diffuse"
	//回调Shader，当上述Shader不运行时使用定义Shader，可填Off
	FallBack "VertexLit"
}
```

以下是Properties语义块支持的属性类型：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled3.png)

以下是SubShader标签中的标签类型：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled4.png)

以下是SubShader状态的可设置类型：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled5.png)

以下是Pass中的标签可设置类型：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled6.png)

Pass的状态可设置类型与SubShader相同+固定渲染管线命令

---

### 相关资料

- GLSL使用文档：http://docs.unity3d.com/Manual/SL-GLSLShaderPrograms.html
- CG语言编写教程：[CG 教程 - 第 1 章。介绍 (nvidia.cn)](https://developer.download.nvidia.cn/CgTutorial/cg_tutorial_chapter01.html)

---

# 初级部分：基础光照与基础纹理

了解了有关于Unity的渲染流程以及UnityShader的基本结构，那么接下来就是实际操作了，接下来的初级篇分别是第6张介绍了光照的基本原理，第7章介绍了基础的纹理效果实现，以及第8张讲述了透明效果的实现。

## 基础光照

对于了解图形学光照原理的同学，这一部分很好理解的。基础光照本身是由四个部分构成，分别是以下四个部分：

- 自发光：`missive`，一个表面本身该向四周发射多少辐射量。如果没有使用全局光照技术，自发光的表面不会照亮周围物体，只是本身会显得更亮。直接使用材质的自发光颜色数据。
- 高光反射：`specular`，描述光线从光源照射到模型表面，该表面会以完全镜面反射散射多少辐射量。
- 漫反射：`diffuse`，描述光线从光源照射到模型表面，该表面会向每个方向散射多少辐射量。根据`兰伯特定律`：反射光线的强度与表面法线和光源方向之间的夹角的余弦值成正比。
- 环境光：`ambient`，用来描述其他所有的间接光照。使用一个全局变量来进行模拟。

需要注意的，环境光与自发光都能够分别由场景中的平行光数据以及材质的数据直接获取。所以要完成基本光照在于实现漫反射与高光反射。

### 漫反射

首先是漫反射的原理主要根据兰伯特定律进行实现，是根据材质本身的颜色以及光源颜色向四周反射光线，以下是光照的计算公式，也是`兰伯特模型`：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled7.png)

其中n是表面法线；I是指向光源的单位矢量，Mdiffuse是材质的漫反射颜色，Clight是光源颜色。0和点乘结果取最大值，避免物体被从后方来的光源照亮。

除了上述的兰伯特模型之外，还有一个半兰伯特模型也用来实现漫反射效果，这是因为兰伯特模型在处理模型背光的情况下会发现模型全黑的情况（模型细节完全丢失，看过去全是黑的，没有轮廓），这是因为在上述计算中法线与光源的余弦值如果是负数，就会统一变成0来限制背光被照亮的情况。但是也导致了背光没有漫反射效果。

为了解决这个问题，就采用了`半兰伯特模型`来进行改进。对于法线与入射光线的余弦值的max处理修正为进行一个a倍的缩放以及一个b大小的偏移。通常情况下是0.5。这样映射范围就在[0,1]之间。

![Untitled](../UnityShader学习总结：基础初级篇/Untitled8.png)

![Untitled](../UnityShader学习总结：基础初级篇/Untitled9.png)

接下来就主要是根据UnityShader的实际结构实现漫反射光照了，这里加上一个完整的Shader文件进行展示，之后如果有同样的代码内容只展示原理实现部分：

- 漫反射效果=入射光线*漫反射系数*法线与入射光线的余弦值（具体公式上查）
- Max函数的写法⇒函数saturate(x)，参数x是用于操作的标量或矢量，是float、float2、float3类型。能够将x截取在[0,1]范围内，如果x是矢量，就对每一个分量都进行一次该操作。

以下是基于顶点着色器实现的兰伯特漫反射光照：

``` glsl
Shader "Unity Shaders Book/Chapter 6/Diffuse Vertex-Level"
{
    Properties{
        //漫反射颜色
        _Diffuse("Diffuse",color) = (1,1,1,1)
    }

    SubShader{
       Pass{
            Tags{"LightMode" = "ForwardBase"}

            CGPROGRAM
						
						//声明顶点着色器与片元着色器
            #pragma vertex vert
            #pragma fragment frag

            #include"Lighting.cginc"

            fixed4 _Diffuse;

            //定义输出输入结构体
            struct a2v {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
            };

            struct v2f {
                float4 pos:SV_POSITION;
                fixed3 color : COLOR;
            };
            
            v2f vert(a2v v) {
                v2f o;
                //顶点从模型空间转换到剪裁空间中；UNITY_MATRIX_MVP=>模型*世界*投影矩阵
                o.pos = UnityObjectToClipPos(v.vertex);
                
                //获取环境光，UNITY_LIGHTMODEL_AMBIENT
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

                //在世界空间计算，mul，点乘，将模型顶点的法线与世界矩阵相乘，获取世界空间的顶点法线；
                //_World2Object 模型空间到世界空间的变换矩阵的逆矩阵，调换在mul中的位置，得到与转置矩阵相同的矩阵乘法
                //normalize 归一化
                fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));

                //获得环境光的光源方向
                fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);

                //漫反射=环境光*表面漫反射颜色*世界法线与世界入射光夹角的余弦值（小于0，则取0）
                //saturate 截取[0,1]
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));

                //光照效果=环境光+漫反射
                o.color = ambient + diffuse;

                return o;
            }

            fixed4 frag(v2f i) :SV_Target{
                return fixed4(i.color,1.0);
            }

            ENDCG
        }
    }
    FallBack "Diffuse"
}
```

由上述代码可以看出，兰伯特光照的计算填写在顶点着色器中，也就是说这里实现的是逐顶点光照。但如果使用片元着色器的话能够得到更好的效果。原因在于模型的细分程度，如果是顶点数量高的高细分模型，逐渐顶点关照能够表现出良好的效果，如果是低细分模型可能会存在一些视觉问题（锯齿等）。

以下是逐像素兰伯特漫反射的实现：

``` glsl
//顶点着色器
//可以使用unity的内置函数
//负责将过顶点位置转换到投影空间
//将顶点法线从模型空间转换到世界空间

//片元着色器（漫反射原理实现）
fixed4 frag(v2f i) : SV_Target {
	//进行漫反射数据计算
				
	//获取环境光数据
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
	//获取世界空间的顶点法线，并归一化
	fixed3 worldNormal = normalize(i.worldNormal);

	//获取世界空间的光源归一化矢量
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				
	//进行漫反射模型计算
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
				
	//合并环境光与漫反射光线
	fixed3 color = ambient + diffuse;
				
	return fixed4(color, 1.0);
}
```

除了兰伯特模型之外，这里也能够进行半兰伯特效果的实现：

``` glsl
fixed4 frag(v2f i) : SV_Target {
	//获取环境光数据
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;

	//获取世界空间的顶点法线，并归一化
	fixed3 worldNormal = normalize(i.worldNormal);

	//获取世界空间的光源归一化矢量
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

	//进行漫反射模型计算
	fixed halfLambert = dot(worldNormal, worldLightDir) * 0.5 + 0.5;
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * halfLambert;

	//合并环境光与漫反射光线
	fixed3 color = ambient + diffuse;

	return fixed4(color, 1.0);
				
}
```

### 高光反射

高光反射主要根据两个高光反射模型进行实现的，分别是Phone高光反射模型与Bline高光反射模型。Phone模型主要根据光源的`反射光线与视角方向之间的夹角`来确定颜色显示。而Bline高光反射模型主要是引入新的矢量h作为入射光线与视角方向的一个划分平均值，并通过他`与法线方向之间的夹角`确定颜色。如下：

- **Phone高光反射模型：**
    
    ![Untitled](../UnityShader学习总结：基础初级篇/Untitled10.png)
    
    - 反射方向计算方法
        
        ![Untitled](../UnityShader学习总结：基础初级篇/Untitled11.png)
        
    - 高光反射计算方法
        
        ![Untitled](../UnityShader学习总结：基础初级篇/Untitled12.png)
        
        - Mglass是表面的光泽度，Mspecular是表面的高光反射颜色，控制高光反射的强度与颜色，Clight是光源的颜色与强度
- **Bline高光反射模型**
    
    引入新的矢量h，作为入射光线l与出射光线v的一个平均值（使用时需要进行归一化变成单位矢量）
    
    ![Untitled](../UnityShader学习总结：基础初级篇/Untitled13.png)
    
    - 计算中间平均矢量的单位矢量
        
        ![Untitled](../UnityShader学习总结：基础初级篇/Untitled14.png)
        
    - 高光反射计算方法
        
        ![Untitled](../UnityShader学习总结：基础初级篇/Untitled15.png)
        
        - Mglass是表面的光泽度，Mspecular是表面的高光反射颜色，控制高光反射的强度与颜色，Clight是光源的颜色与强度

总而言之，**高光反射效果**=**入射光线*****高光反射系数***（**视角方向**与**反射方向**的**余弦值**）或（**视角方向**与平分矢量的**余弦值**）的**物体表面光泽度的指数方**（具体公式上查）

以下是Phone高光反射的逐顶点光照实现：

``` glsl
v2f vert(a2v v) {
	v2f o;
	//顶点：模型空间=>投影空间
	o.pos = UnityObjectToClipPos(v.vertex);
				
	//环境光
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
	//法线：模型空间=>世界空间
	fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
	//入射方向，归一化
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				
	//漫反射
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
				
	//反射光线，归一化
	fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
	//视角方向，归一化
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
				
	//获取高光反射效果
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
				
	//最终效果=环境光+漫反射+高光反射
	o.color = ambient + diffuse + specular;
							 	
	return o;
}
```

以下是Phone高光反射的逐像素光照实现：

``` glsl
fixed4 frag(v2f i) : SV_Target {
	//环境光
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
	//世界法线
	fixed3 worldNormal = normalize(i.worldNormal);
	//世界，入射光线（反向的）
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				
	//漫反射效果
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLightDir));
				
	//反射光线归一化
	fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
	//世界空间，观察视角（摄像机-顶点位置）
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
	//高光反射
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
	

	return fixed4(ambient + diffuse + specular, 1.0);
}
```

以下是blinn-Phone模型高光反射的逐像素光照实现：

``` glsl
fixed4 frag(v2f i) : SV_Target {
	//环境光
	fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
	//世界法线
	fixed3 worldNormal = normalize(i.worldNormal);
	//世界，入射光线（反向的）
	fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
				
	//漫反射效果
	fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
				
	//世界空间，观察视角
	fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
	//反射夹角中线，归一化
	fixed3 halfDir = normalize(worldLightDir + viewDir);
	//高光反射效果
	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
	return fixed4(ambient + diffuse + specular, 1.0);
}
```

以下是光照效果图：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled16.png)

### Unity常用内置函数

![Untitled](../UnityShader学习总结：基础初级篇/Untitled17.png)

---

## 基础纹理

Unity Shader的使用离不开纹理，所以第7章内容介绍了在unityShader中的纹理使用，分别介绍了单张纹理的使用，凹凸映射（法线）纹理的使用，渐变纹理的使用以及遮罩纹理的使用。

### 单张纹理的使用

首先是单张基础纹理的使用，首先需要定义Shader中纹理的属性，之后在Pass中声明属性存储之前定义的属性信息，设置好进出着色器的存储信息的结构体，之后再在顶点着色器中进行纹理采样即可。

重点在于顶点着色器中对纹理的偏移以及在片元着色器中对纹理的采样，可以使用unity内置的偏移函数**TRANSFORM_TEX（纹理，偏移量）**以及采样函数**tex2D(纹理,偏移UV)**。

``` glsl
Shader "Unity Shaders Book/Chapter 7/Single Texture"{
    Properties
    {
				//定义纹理和会用到的纹理属性
        //颜色
        _Color ("Color Tint", Color) = (1,1,1,1)
        //声明一个2D纹理，内置纹理“white”，全白
        _MainTex ("Main Tex", 2D) = "white" {}
        //镜面反射（高光反射）
        _Specular("Specular",Color)=(1,1,1,1)
        //光泽（材质高光反射光泽度，计算后面的那个指数）
        _Gloss ("Gloss", Range(8.0,256)) =20
    }
    SubShader
    {
       Pass{
            //光照模式
            Tags{"LightMode"="ForwardBase"}

            CGPROGRAM
						
            #pragma vertex vert
            #pragma frafment frag
            #include "Lighting.cginc"

            //Properties中的属性描述
            fixed4 _Color;
            sampler2D _MainTex;
            fixed4 _Specular;
            float _Gloss;

            //纹理属性描述，st是缩放和平移的缩写，.xy存放的是缩放值，.zw存放的是偏移值
            float4 _MainTex_ST;

            //输出输入结构体
            struct a2v {
                float4 vertex:POSITION;
                float3 normal:NORMAL;
                //模型第一组纹理坐标
                float4 texcoord:TEXCOORD0;
            };

            struct v2f {
                float4 pos:SV_POSITION;
                float3 worldNormal:TEXCOORD0;
                float3 worldPos:TEXCOORD1;
                //存储纹理采样数据，对纹理坐标进行采样
                float2 uv:TEXCOORD2;
            };
       
            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);

                o.worldNormal = UnityObjectToWorldNormal(v.normal);

                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

                //_MainTex_ST.xy 与纹理坐标相乘，进行缩放，加上_MainTex_ST.zw进行纹理偏移
                o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                //以上过程可以直接调用TRANSFORM_TEX函数
                // TRANSFORM_TEX(tex,name)
                // tex=>顶点纹理坐标，name=>纹理名称
                //o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);

                //输出
                return o;
            }

            fixed4 frag(v2f i) : SV_Target{
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));

                //使用纹理对漫反射颜色进行采样
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;

                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;

                //漫反射
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(worldNormal, worldLightDir));

                //Bline模型
                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);

                return fixed4(ambient + diffuse + specular, 1.0);
            }

            ENDCG
        }

    }
    FallBack "Diffuse"
}
```

除了上述在UnityShader脚本中编写的代码之外，材质需要绑定Shader，根据Shader中的属性设置能够绑定纹理，但同时还需要注意一下纹理的界面设置：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled18.png)

- Texture Type：纹理类型
- Alpha Source：透明度选项来源（from Grayscale  来自像素灰度值生成，等等）
- Wrap Mode：贴图模式
    - Repeat：纹理坐标超过1，整数部分舍弃，直接使用小数部分进行采样（纹理将会不断重复）
    - Clamp：纹理坐标大于1，将会截取到1，如果小于0，将会截取到0（超出范围部分取纹理边界颜色）
        
        ![Untitled](../UnityShader学习总结：基础初级篇/Untitled19.png)
        
- Filter Mode：滤波模式，一共有三种：Point，Bilinear，Trilinear。得到的图片显示效果依次提升。以下是三种效果：
    - Point：采样像素点数目只有一个
    - Bilinear：采用了线性滤波，每个像素目标会找周围4个邻近像素，通过线性插值混合之后最终像素。
    - Trilinear：大致与Bilinear相同，只是多了**多级渐远纹理**之间的相互混合。

![Untitled](../UnityShader学习总结：基础初级篇/Untitled20.png)

### 多级渐远纹理（mipmapping）技术（有点像lod技术，⇒Levels of Detail）

将原纹理提前使用滤波处理得到更小的图像，形成图像金字塔，每一层级都是上一层级的降采样效果。（需要多余空降进行降采样纹理存储，大致多耗费33%左右的空间）是一种空间换时间的方法。

Unity中使用方法：Texture Type选择Advanced，勾选Generate Mip Maps 即可。以下是渲染效果：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled21.png)

![Untitled](../UnityShader学习总结：基础初级篇/Untitled22.png)

- 材质面版的选项是由shader中的属性条目进行设置的
- MainTex：纹理的缩放与偏移量
    - Tiling（对应代码中的MainTex.xy）:纹理缩放值
    - Offset（对应代码中的MainTex.zw）:纹理偏移值
- Render Queue：渲染队列，可以看一下渲染队列顺序那一篇博文（关于Unity渲染顺序）
- Enable GPU Instancing：高效渲染大量相似物体或粒子的技术。通过合并相同材质和属性的物体或粒子，并以单个绘制调用的方式发送给GPU，从而减少了CPU与GPU之间的数据传输和渲染开销。（降低draw call的手段之一）
- Gouble Sided Global Illumination：双面全局照明，指定光照贴图是否在计算全局光照时考虑几何体的两面。设置为 true 时，如果使用渐进光照贴图，则背面将使用与正面相同的发射和反照率来反射光。

### 凹凸映射（高度图，法线纹理）

实现凹凸映射的两种方法，但两者一般一起使用（丰富表面凹凸额外信息——光照）：

- **高度映射**：使用一张**高度纹理**来模拟表面位移，然后得到一个修改后的法线值
- **法线映射**：使用一张**法线纹理**来存储表面法线

**`法线映射：`**

- 法线方向的分量范围是[-1,1]，像素的分量范围是[0,1]（以像素形式存储）。所以需要进行映射。公式如下：
    
    $pixel=（normal+1）/2$
    
- 在shader中对法线纹理采样之后还需要进行一次反映射，公式如下：
    
    $normal=pixel*2-1$
    

Unity内置函数能够提供这部分计算。

**`采用的法线纹理：`**

法线是矢量，所以法线的保存通常是需要找到是哪一个空间的矢量，才有意义：

- 模型空间的法线纹理：（以模型为中心）模型顶点自带的法线，是定义在模型空间中的，所以将修改后模型空间的表面法线存储在一张纹理中，称为模型空间的法线纹理。
- 切线空间的法线纹理（**使用**）：（以顶点为中心）z轴为法线方向，x轴为顶点切线方向，y轴是x轴与z轴叉乘得到的，称为副切线或副法线。如图。

![Untitled](../UnityShader学习总结：基础初级篇/Untitled23.png)

模型空降存储法线优点：

- 实现简单
- 纹理坐标的缝合处和尖锐的边角部分，突变（缝隙）减少，可以提供跟平滑的边界。

切线空间存储法线优点：

- 自由度高，模型空间下的法线信息是**绝对法线信息**，只能够用于创建他的模型。切线空间上的是**相对法线信息**，用在不用网格上能得到较为合理的结果。
- 可以进行UV动画，可以移动纹理的UV坐标实现凹凸移动的效果（水面），模型空间的法线纹理会完全错误，原因同上。
- 可以重用法线纹理，一个砖块，使用一张法线纹理就能够用到所有的6个面
- 可压缩，切线空间的法线纹理的法线的z方向总是正方向，因此能够仅存储xy，推导得到z。

以下是切线空间的凹凸渲染

``` glsl
Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		//纹理
		_MainTex ("Main Tex", 2D) = "white" {}
		//法线纹理“bump”是内置法线纹理
		_BumpMap ("Normal Map", 2D) = "bump" {}
		//用来控制凸凹程度（法线），为0时，法线不对光照造成任何影响
		_BumpScale ("Bump Scale", Float) = 1.0
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}

		Pass { 
			......
			CGPROGRAM
			......

			//属性描述
			fixed4 _Color;

			//主纹理与主纹理缩放偏移
			sampler2D _MainTex;
			float4 _MainTex_ST;
			//法线纹理与法线纹理缩放偏移
			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			//用来控制凸凹程度（法线），为0时，法线不对光照造成任何影响
			float _BumpScale;

			fixed4 _Specular;
			float _Gloss;
			
			//顶点空间是顶点法线（已知）与切线构建出的空间，这里需要获取切线方向
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				//切线方向
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};
			
			//需要在顶点着色器中计算切线空间的光照与视角
			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
				float3 lightDir: TEXCOORD1;
				float3 viewDir : TEXCOORD2;
			};

			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				//uv的xy分量存储第一张纹理_MainTex的纹理坐标
				o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				//uv的zw分量存储法线纹理_BumpMap的纹理坐标
				o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;

				//副切线
				float3 binormal = cross( normalize(v.normal), normalize(v.tangent.xyz) ) * v.tangent.w;
				//将模型空间的切线方向、副切线方向与法线方向安行排列,得到模型空间=>切线空间的变换矩阵
				float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
				//unity内置方法在下方，直接计算模型空间=>切线空间的变换矩阵,存储到rotation
				//TANGENT_SPACE_ROTATION;
				
				//获取切线空间的光源
				o.lightDir = mul(rotation, normalize(ObjSpaceLightDir(v.vertex))).xyz;
				//获取切线空间的视角
				o.viewDir = mul(rotation, normalize(ObjSpaceViewDir(v.vertex))).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				//切线空间的光源方向与视角方向矢量归一化
				fixed3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentViewDir = normalize(i.viewDir);
				
				//进行法线纹理采样
				fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
				fixed3 tangentNormal;

				//法线纹理存储的是法线经过映射之后的像素值。需要将其反映射回来
				// 如果unity中没有设置法线纹理为Normal map类型，就需要在代码中进行反映射
				//tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale;
				//tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
				//unity内置方法
				tangentNormal = UnpackNormal(packedNormal);
				tangentNormal.xy *= _BumpScale;
				//切线空间的法线z分量，由xy分量获得
				//saturate函数限制数值范围在[0,1]中
				tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
				
				//使用纹理对漫反射颜色进行采样
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				//环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				//漫反射
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));

				//高光反射，blinn模型
				fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}

```

以下是世界空间的凹凸渲染

``` glsl
//凹凸，世界空间——法线纹理
Shader "Unity Shaders Book/Chapter 7/Normal Map In World Space" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		
		//贴图与法线贴图以及法线凹凸系数
		_MainTex ("Main Tex", 2D) = "white" {}
		_BumpMap ("Normal Map", 2D) = "bump" {}
		_BumpScale ("Bump Scale", Float) = 1.0

		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Color;

			sampler2D _MainTex;
			float4 _MainTex_ST;

			sampler2D _BumpMap;
			float4 _BumpMap_ST;
			float _BumpScale;

			fixed4 _Specular;
			float _Gloss;
			
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				float4 pos : SV_POSITION;
				float4 uv : TEXCOORD0;
				//注意这里计算的是世界空间的凹凸纹理
				//这里是存储了切线空间到世界空间的转换矩阵
				float4 TtoW0 : TEXCOORD1;  
				float4 TtoW1 : TEXCOORD2;  
				float4 TtoW2 : TEXCOORD3;*
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				//对纹理坐标进行缩放与平移
				o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				o.uv.zw = v.texcoord.xy * _BumpMap_ST.xy + _BumpMap_ST.zw;
				
				//世界空间顶点
				float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
				//世界空间法线
				fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
				//世界空间切线
				fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);
				//世界空间副切线
				fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w;
				
				// 摆放世界空间法线，世界空间切线，世界空间副切线得到切线空间到设计空间的变换矩阵
				// 保存世界空间顶点xyz（利用存储空间，保存世界空间坐标）
				o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
				o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
				o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);**
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				// 获取世界空间顶点坐标	
				float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
				// 获取世界空间光源与视角
				fixed3 lightDir = normalize(UnityWorldSpaceLightDir(worldPos));
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
				
				// 获取切线空间的法线，使用UnpackNormal进行法线贴图采样与解码（将法线纹理标格式识成Normal map）
				fixed3 bump = UnpackNormal(tex2D(_BumpMap, i.uv.zw));
				//通过_BumpScale对法线进行缩放
				bump.xy *= _BumpScale;
				bump.z = sqrt(1.0 - saturate(dot(bump.xy, bump.xy)));

				//将法线从切线空间转换到世界空间
				bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
				
				//纹理材质采样
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				//环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				//漫反射
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(bump, lightDir));

				fixed3 halfDir = normalize(lightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(bump, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

- 注意，如果使用UnpackNormal函数来进行法线贴图的法线获取。需要将法线贴图设置成Normal map类型。这是因为设置成Normal map类型后，unity会根据不同平台对纹理进行压缩（如DZT5nm格式会省略z轴，通过xy轴求）。之后在通过UnpackNormal解压缩并获取正确的法线。

### 渐变纹理

- 一种冷到暖色调的着色技术（插画风格的渲染效果）

以下是三种渐变纹理的区别：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled24.png)

``` glsl
//渐变纹理
Shader "Unity Shaders Book/Chapter 7/Ramp Texture" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		**//渐变纹理
		_RampTex ("Ramp Tex", 2D) = "white" {}**
		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag

			#include "Lighting.cginc"
			
			fixed4 _Color;

			//定义渐变纹理与渐变纹理缩放平移
			sampler2D _RampTex;
			float4 _RampTex_ST;

			fixed4 _Specular;
			float _Gloss;
			
			//顶点着色器数据输入
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				//第一组纹理（输入的纹理）
				float4 texcoord : TEXCOORD0;
			};
			
			struct v2f {
				//投影空间顶点
				float4 pos : SV_POSITION;
				float3 worldNormal : TEXCOORD0;
				float3 worldPos : TEXCOORD1;
				float2 uv : TEXCOORD2;
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				
				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
				
				//TRANSFORM_TEX,计算平铺与平移之后的纹理坐标
				o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 worldNormal = normalize(i.worldNormal);
				//获得世界空间光源
				fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				
				//环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
				
				// 半兰伯特模型（控制显示范围）
				fixed halfLambert  = 0.5 * dot(worldNormal, worldLightDir) + 0.5;
				//获取漫反射颜色（通过纹理采样获取）
				fixed3 diffuseColor = tex2D(_RampTex, fixed2(halfLambert, halfLambert)).rgb * _Color.rgb;
				
				//_LightColor0顶点光源强度与漫反射强度得到漫反射效果
				fixed3 diffuse = _LightColor0.rgb * diffuseColor;
				
				//高光反射，blinn模型
				fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
				fixed3 halfDir = normalize(worldLightDir + viewDir);
				fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
				
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

- 在片元着色器中使用半兰伯特模型对法线方向与光照方向的点积进行一次半兰伯特计算，映射了显示范围。
    - 个人推测，避免兰伯特光照造成的背光模型无变化（不符合渐变纹理显示效果），所以使用半兰伯特模型
- 由_RampTex纹理图片可以看出，这是一个横轴颜色逐渐变化，重轴颜色相同的纹理。所以使用半兰伯特部分构建一维纹理坐标，对_RampTex纹理进行采样。（fixed2(halfLambert, halfLambert)）
- 需要注意：进行纹理采样的时候，需要将渐变纹理的Wrap Mode模式设置成Clamp模式，防止纹理采样由于浮点数精度造成的问题。

### 遮罩纹理

- 用来控制部分区域不同于其他区域的显示效果
- 遮罩纹理流程：通过采样获取遮罩纹理的值，使用其中某个（或某几个）通道的值与某种表面属性相乘（改通道值为0时，表面不受该属性影响）

``` glsl
// Upgrade NOTE: replaced 'mul(UNITY_MATRIX_MVP,*)' with 'UnityObjectToClipPos(*)'
//遮罩纹理
Shader "Unity Shaders Book/Chapter 7/Mask Texture" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		//主纹理
		_MainTex ("Main Tex", 2D) = "white" {}
		//法线纹理与法线凹凸缩放
		_BumpMap ("Normal Map", 2D) = "bump" {}
		_BumpScale("Bump Scale", Float) = 1.0
		//遮罩纹理与遮罩缩放
		_SpecularMask ("Specular Mask", 2D) = "white" {}
		_SpecularScale ("Specular Scale", Float) = 1.0

		_Specular ("Specular", Color) = (1, 1, 1, 1)
		_Gloss ("Gloss", Range(8.0, 256)) = 20
	}
	SubShader {
		Pass { 
			Tags { "LightMode"="ForwardBase" }
		
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#include "Lighting.cginc"
			
			fixed4 _Color;
			//主纹理与纹理缩放
			sampler2D _MainTex;
			float4 _MainTex_ST;
			//法线纹理与纹理缩放
			sampler2D _BumpMap;
			float _BumpScale;

			//遮罩纹理与缩放（是用来控制高光反射效果的）
			sampler2D _SpecularMask;
			float _SpecularScale;**

			fixed4 _Specular;
			float _Gloss;
			
			//顶点着色器输入：模型空间顶点、法线、切线、第一张纹理
			struct a2v {
				float4 vertex : POSITION;
				float3 normal : NORMAL;
				float4 tangent : TANGENT;
				float4 texcoord : TEXCOORD0;
			};
			
			//顶点输出与片元着色器输入：顶点位置、uv纹理、光源、视角
			struct v2f {
				//剪裁空间
				float4 pos : SV_POSITION;
				float2 uv : TEXCOORD0;
				//切线空间
				float3 lightDir: TEXCOORD1;
				float3 viewDir : TEXCOORD2;
			};
			
			//顶点着色器
			v2f vert(a2v v) {
				v2f o;
				//模型空间顶点=>剪裁空间顶点
				o.pos = UnityObjectToClipPos(v.vertex);
				//对第一张纹理进行采样（缩放平移）
				o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				
				//unity内置方法，直接计算模型空间=>切线空间的变换矩阵,存储到rotation
				TANGENT_SPACE_ROTATION;
				//获得切线空间的光源与视角
				o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
				o.viewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				//切线空间光源视角归一化
			 	fixed3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentViewDir = normalize(i.viewDir);

				//获得切线空间法线，纹理采样与解码
				fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uv));
				tangentNormal.xy *= _BumpScale;
				tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

				//主纹理材质采样
				fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
				
				//环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				//漫反射
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));
				
				//blinn模型，视角与入射光源的平分线
			 	fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
			 	
				//遮罩纹理采样（本纹理，三个通道数值相同步，rgb三个通道只使用了r通道）
			 	fixed specularMask = tex2D(_SpecularMask, i.uv).r * _SpecularScale;
			 	
				//高光反射——高光反射结果乘以遮罩纹理采样数据
			 	fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss) * specularMask;
			
				return fixed4(ambient + diffuse + specular, 1.0);
			}
			
			ENDCG
		}
	} 
	FallBack "Specular"
}
```

以下是效果对比：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled25.png)

---

## 透明测试、混合与面剔除

透明效果是游戏中最常见的效果之一。在unity中，实现透明效果有2种方式，分别是完全透明的透明度测试以及会混合前后颜色，实现半透明效果的透明度混合。透明度测试相对简单，这也是因为透明度测试实现的是完全透明效果，不用关闭深度写入，效果极端；而透明度混合则相对复杂，主要原因还是因为关闭了深度写入，导致的一些由于渲染顺序或物体相互之间的层级引起的一些问题。所以首先我们要先来了解一下深度写入与渲染顺序的概念。

### 深度写入与渲染顺序

深度写入：用于判断物体遮挡情况下需要渲染的颜色。将深度数值写入深度缓冲区，在深度测试的时候，在颜色缓冲区将深度小的像素的颜色替换深度大的物体的深度颜色。这样就能够渲染出离摄像机更近的物体颜色。

- 关闭**深度写入**：需要注意的是，透明度混合是由`透明物体的颜色`以及`透明物体之后的物体颜色`组合成的。其中透明物体之后的物体颜色存储在存储在颜色缓冲区。如果不关闭深度写入，在进行深度测试的时候，之后的物体由于深度更大，会因为深度写入将原本存储在颜色缓冲区的物体表面颜色数值剔除掉。就无法进行透明度混合了。（之后的物体颜色丢失）
    - 简单来说，两个物体如果都有深度值，就会进行深度检测，导致颜色缓冲区的颜色被替代。
    - 关闭深度写入，颜色缓冲区就有之后物体的颜色，与当前半透明物体的颜色混合，就有了混合效果的颜色。
- 渲染顺序：由于关闭了深度写入，所以无法判断物体里摄像机远近，所以渲染时需要注意物体的渲染顺序，先渲染不透明物体，在渲染半透明物体。以下是个渲染效果说明：
    - 存在半透明物体A，在前；不透明物体B，在后。（都为半透明也同理）
    - 正确渲染效果：先渲染物体B，再渲染物体A。不透明物体B开启了深度测试与深度写入，此时深度缓冲区没有数值，B将自己的深度写入深度缓冲区，颜色写入颜色缓冲区。再渲染半透明物体A，由于物体A关闭深度写入，只有深度检测。深度检测物体A里摄像机更近，读取颜色缓冲区B的颜色，与物体A颜色混合，达成半透明效果。
    - 错误渲染效果：先渲染物体A，再渲染物体B。半透明物体A只开启了深度测试，没有深度写入，此时深度缓冲区没有数值也不写入，A将自己的颜色写入颜色缓冲区。再渲染不透明物体B，由于B开启了深度检测与深度写入。同时发现深度缓冲区内没有深度值，B会直接修改颜色缓冲区中的颜色（为B的颜色），A的颜色被覆盖丢失，最后屏幕上渲染出B的颜色（看起来B在A之前，也就是没有混合，只有B的颜色）。

至于Unity的渲染顺序，有需要的可以参考[Unity渲染顺序探究 | 墨墨辰的旋转小屋 (the-black-sun.github.io)](https://the-black-sun.github.io/2023/09/05/Unity%E6%B8%B2%E6%9F%93%E9%A1%BA%E5%BA%8F%E6%8E%A2%E7%A9%B6/)的简要说明。需要注意的是：

- 先渲染所有不透明物体，并开启深度测试与深度写入
- 将半透明物体安距离摄像机远近排序，然后从后往前渲染，并开启深度测试，关闭深度写入

### 存在的问题

虽然有深度测试与渲染顺序的调整存在，但是仍然存在一个问题：单个重叠物体或多个相互重叠物体会导致混合效果错误。存在物体相互堆叠的情况，存在物体的部分区域的渲染前后排序顺序不一致的情况，但是调整渲染顺序无法解决这一个问题。因为根据深度进行渲染顺序调整是以物体为单位进行的，而深度测试与深度写入是以像素点为单位进行的。面对这种情况，UnityShader通常使用两个Pass进行解决（一个进行混合操作，一个进行深度写入，注意关闭深度写入的相关操作也在Pass中进行）。

![Untitled](../UnityShader学习总结：基础初级篇/Untitled26.png)

### UnityShader的渲染队列

上述由于物体堆叠导致的渲染顺序问题，Unity通过渲染队列来进行解决。SubShader的`Queue`标签可以决定模型改归于哪一个渲染队列，索引号越小表示越早被渲染。渲染队列有以下几种：

| 名称 | 队列索引号(RenderQueue) | 描述 |
| --- | --- | --- |
| Background | 1000 | 该队列在其他所有队列之前被渲染，通常用来渲染需要被绘制在背景上的物体 |
| Geometry | 2000 | 默认渲染队列，大多数物体使用的队列，不透明物体使用的队列 |
| AlphaTest | 2450（小于2500为完全不透明） | 需要进行透明度测试的物体使用的队列。 |
| Transparent | 3000 | 该队列中的物体会在前两个队列（Geometry与AlphaTest）渲染之后，再根据从后往前的顺序进行渲染，所有使用了透明度混合的物体使用该队列。 |
| Overlay | 4000 | 该队列用来实现叠加效果，需要在最后渲染的物体使用这个队列 |

以下是UnityShader中的渲染队列设置：

- 透明度测试
    
    ``` glsl
    SubShader{
    	Tags{"Queue"="AlphaTest"}
    	Pass{
    		......
    	}
    }
    ```
    
- 透明度混合
    
    ``` glsl
    SubShader{
    	Tags{"Queue"="Transparent"}
    	Pass{
    		ZWrite Off
    		......
    	}
    }
    ```
    

接下来来看一下透明度测试与透明度混合的具体效果实现。

### 透明度测试

实现的是完全透明或完全不透明效果。需要用到一个由CG提供的透明度测试函数clip，如下所示：

- 函数：void clip(float**X**  x)⇒`X代表数字无~4`
- 参数：剪裁时使用的标量或矢量条件
- 描述：如果给定参数的任何一个分量是负数，就会舍弃当前像素的输出颜色

``` glsl
void clip(float4 x){
	if(any(x<0))
		discard;
}
```

以下是透明度测试的部分代码：没有通过透明度测试的像素直接舍弃掉（discard）。

``` glsl
Shader "Unity Shaders Book/Chapter 8/Alpha Test" {
	Properties {
		//颜色与纹理等属性
		....
		
		//像素透明度参数
		_Cutoff ("Alpha Cutoff", Range(0, 1)) = 0.5
	}
	SubShader {
		//Queue设置为透明度测试；
		//IgnoreProjector设置为true，不收到投影器影响；
		//RenderType类型，该shader加入Transparent组
		Tags {"Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}
		
		Pass {
			....
			//透明度参数
			fixed _Cutoff;
			....
			
			fixed4 frag(v2f i) : SV_Target {
				....

				//纹理采样与解码
				fixed4 texColor = tex2D(_MainTex, i.uv);
				
				// Alpha test  开启透明度测试
				clip (texColor.a - _Cutoff);
				// Equal to 
				// if ((texColor.a - _Cutoff) < 0.0) {
				//discard;
				//}
				
				....
			}

			....
		}
	}
	FallBack "Transparent/Cutout/VertexLit"
}

```

### 透明度混合

透明度混合能够实现真正的半透明效果。透明度混合需要通过Blend命令进行逐片元透明混合设置，主要的Blend命令如下：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled27.png)

由上图可以看出第2种和第3种命令存在着混合因子这个概念，这个混合因子是为了控制两种颜色的混合比例而产生的。其中第2种混合命令，提供两个混合因子用来控制两个混合颜色的混合比例，RGB与A通道的混合因子相同；第3种提供了四个混合因子，多出来的两个分别用来里控制两种颜色的透明度通道，RGB与A通道的混合因子不同。

同时以下是SubShader中支持的混合因子：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled28.png)

除此之外还有SubShader支持的混合操作（前后两种颜色混合方式）以及常见的混合类型，通过`BlendOp命令`进行修改：

| 操作 | 描述 |
| --- | --- |
| Add | 将混合后的源颜色与混合后的目的颜色相加。是默认的混合操作，等式如下：O=SrcFactor*S+DstFactor*D |
| Sub | 将混合后的源颜色减去混合后的目的颜色。O=SrcFactor*S-DstFactor*D |
| RevSub | 将混合后的目的颜色减去混合后的源颜色。O=DstFactor*D-SrcFactor*S |
| Min | 使用源颜色和目的颜色中较小的值，每个分量都进行比较。 |
| Max | 使用源颜色和目的颜色中较大的值，每个分量都进行比较。 |
| 其他逻辑操作 | 仅仅DX11支持 |

``` glsl
//注意指令格式,SrcFactor、DstFactor以及SrcFactorA和DstFactorA都能用以上混合因子表中的值代替，如下
Blend SrcFactor DstFactor
Blend SrcFactor DstFactor,SrcFactorA DstFactorA

//正常（Normal），透明度混合
Blend SrcAlpha OneMinusSrcAlpha

//柔和相加
Blend OneMinusDstColor One

//正片叠底，即相乘
Blend DstColor Zero

//两倍相乘
Blend DstColor SrcColor

//变暗，注意这里的混合因子无用，max也相同
BlendOp Min
Blend One One

//变亮
BlendOp Max
Blend One One

//滤色
Blend OneMinusDstColor One

//等同于
Blend One OneMinusSrcColor

//线性减淡 
Blend One One
```

下面是最常见的混合计算公式（一般而言是按透明度进行混合的，但是可以根据混合命令进行设置）：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled29.png)

- DstColor：颜色缓冲区的颜色
- SrcAlpha：原颜色的混合因子：SrcFactor。可能是原颜色的透明度。
- SrcColor：原颜色
- DstColor：目标颜色；原本颜色缓冲区颜色
- （1-SrcAlpha）：DstFactor，目标颜色的混合因子
- 注意：修改公式内容可以得到其他类型的混合公式与混合效果

以下是透明度混合的部分部分UnityShader代码设置：

``` glsl
//由于透明度混合是（逐片元）可配置的，并非完全可编程的，所以只需要在Shader中设置好混合命令即可。
Shader "Unity Shaders Book/Chapter 8/Alpha Blend" {
	Properties {
		....
		
		//像素半透明参数
		_AlphaScale ("Alpha Scale", Range(0, 1)) = 1
	}
	SubShader {
		//Queue设置为半透明Transparent
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		Pass {
			....
			//关闭深度写入
			ZWrite Off
			//设置透明混合命令
			Blend SrcAlpha OneMinusSrcAlpha
			
			....
			fixed _AlphaScale;
			....
			
			fixed4 frag(v2f i) : SV_Target {
				....
				//返回最后的合成颜色，并设置透明度通道=纹理像素的透明通道*材质参数_AlphaScale
				return fixed4(ambient + diffuse, texColor.a * _AlphaScale);
			}
			
			ENDCG
		}
	} 
	FallBack "Transparent/VertexLit"
}
```

## 开启深度写入的半透明效果

还记得之前在`存在的问题`小节中提到的问题吗？不同物体部分区域前后位置不一致导致的混合效果错误现象，是由于深度写入关闭的原因，解决方法就是开启深度写入了（- -“）。之前说了，之所以关闭深度写入是因为避免深度写入之后，将颜色缓冲区的旧颜色（目标颜色，后面物体的颜色）替换成深度较浅的前面物体的颜色。所以这里说的开启深度写入指的是将深度写入深度缓冲区，但是不进行包括颜色替换的其他任何操作。在实际实现中表现为单独使用一个Pass进行深度写入操作。部分代码如下：

``` glsl
Shader "Unity Shaders Book/Chapter 8/Alpha Blend" {
	Properties {
		....

		//像素半透明参数
		_AlphaScale ("Alpha Scale", Range(0, 1)) = 1
	}
	SubShader {
		//Queue设置为半透明Transparent
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		// 只进行深度写入
		Pass {
			//打开深度写入
			ZWrite On

			//渲染命令ColorMask，用于设置颜色通道的写掩码（write mask）
			//语义:ColorMask RGB|A|0|其他RGBA的组合
			//以下表示，该Pass不写入任何颜色通道，不会输出任何颜色
			ColorMask 0
		}

		Pass {
			Tags { "LightMode"="ForwardBase" }
			//关闭深度写入
			ZWrite Off
			//设置透明混合命令
			Blend SrcAlpha OneMinusSrcAlpha
			
			//其余部分操作与透明度混合相同
			....
		} 
	} 
	FallBack "Transparent/VertexLit"
}
```

PS：这种深度写入能够完成不同顺序的物体之间的混合效果，但是在同一物体的遮挡混合中不能够实现混合，具体效果可以查看下方效果展示中的`透明度混合，深度写入`，除此之外，在透明效果中还需要注意透明应该需要能够看见当前透明物体的内部结构，这就需要使用到剔除的功能了，同样经过测试，开启了深度写入的同物体也不能展示双面不被剔除的显示效果（也就是关闭剔除之后，开启深度写入，没办法展示出`双面`的效果，反而与开启剔除效果相同），具体原因待探究。

### 双面渲染的透明效果

如果物体有透明效果。那么不但应该能够透过它看见其他物体的样子，还应该能够看见物体的内部结构。但是以上的渲染效果并不会实现这些功能，导致渲染的透明或半透明物体看上去只是半个物体（背面的半个消失了，如下图）

![Untitled](../UnityShader学习总结：基础初级篇/Untitled30.png)

这是因为物体渲染的时候开启了剔除（cull）功能，并且默认设置成了Back。让背对摄像机，摄像机看不见的部分不进行渲染。所以造成了透明效果的穿帮。

- Cull Back：开启背面剔除
- Cull Front：开启正面剔除
- Cull Off：关闭剔除

以下是透明度测试的双面剔除的部分代码,效果展示可以再本节最下方查看：

``` glsl
Pass {
	Tags { "LightMode"="ForwardBase" }
			
	// 关闭剔除功能
	Cull Off
	......

}
```

透明度测试的剔除很简单，但是透明度混合不能够向透明度测试一样操作。这是因为透明度混合关闭了深度写入，为了保证同一个物体的渲染顺序，需要分成两个Pass分别进行正面剔除的渲染与背面剔除的渲染。部分代码如下：

``` glsl
Shader "Unity Shaders Book/Chapter 8/Alpha Blend With Both Side" {
	Properties {
		....
		_AlphaScale ("Alpha Scale", Range(0, 1)) = 1
	}
	SubShader {
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		Pass {
			Tags { "LightMode"="ForwardBase" }
			
			// First pass renders only back faces 
			Cull Front
			
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha
			......
		}
		
		Pass {
			Tags { "LightMode"="ForwardBase" }
			
			// Second pass renders only front faces 
			Cull Back
			
			ZWrite Off
			Blend SrcAlpha OneMinusSrcAlpha
			......
		}
	} 
	FallBack "Transparent/VertexLit"
}
```

本章实现的透明效果如下：

![Untitled](../UnityShader学习总结：基础初级篇/Untitled31.png)

---

## 问题探究

等unityShader总结全部整理好之后，会归纳一下可能不太熟悉的知识点或问题，看看能不能出个问题总结。这里先记录一下这部分内容学习遇上的问题，各位大佬有了解的也可以尝试在评论区进行评论。

1. 透明混合效果的阴影实现
2. 开启深入写入的本物体无法进行透明混合，原因与是否有解决方法
3. uv坐标与tex2d（）的采样原理
4. UnityShader中定义的属性存储语义（这里指的是结构体中的POSITION等等）以及属性（这里指的是属性）相关的声明类型。