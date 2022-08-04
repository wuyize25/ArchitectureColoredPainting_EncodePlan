二级编码大体结构及渲染流程如下：

首先分为图元和索引两大块，图元内坐标范围为-1到1，一张图中可能有多个相同的图元，这些图元只需在图元缓存中保存一次，为了记录图元的位置和变换信息，对图元建立一个索引结构，同时为了方便求交，使用BVH层次包围盒结构，BVH每个结点包含leftChild，rightChild和bound即包围盒坐标，BVH的最后一个结点的leftChild为图元索引加上BVH数组长度，rightChild特殊处理为逆时针旋转角度。

图元由轮廓包围的封闭图形和线条构成，轮廓可以由直线、二阶贝塞尔曲线和三阶贝塞尔曲线组成，支持任意数量轮廓围成的图形的渲染，轮廓围成的图形中不能含有空腔，对于一个复杂的含有许多轮廓的封闭图形而言，应当尽可能将其分割成数个由三段轮廓线包围的广义三角形，并对这些广义三角形以及不构成图形的线条建立BVH索引，图元内BVH索引存储在外部BVH数组后。

着色器接收的5个buffer：

```glsl
layout(std430, binding = 1) buffer bvhBuffer
{
	uint bvhLength;
	uvec2 bvhChildren[];
};
layout(std430, binding = 2) buffer bvhBoundBuffer
{
	vec4 bvhBound[];
};
layout(std430, binding = 3) buffer elementOffsetBuffer
{
	/**********************
	** @x elementBvhRoot
	** @y elementBvhLength
	** @z pointsOffset
	** @w linesOffset
	**********************/
	uvec4 elementOffsets[];
};
layout(std430, binding = 4) buffer elementIndexBuffer
{
	uint elementIndex[]; //线和面
};
layout(std430, binding = 5) buffer elementDataBuffer
{
	float elementData[]; //点和Style
};
```

bvhLength为外部索引数组长度，bvhChildren为每个结点的两个儿子，x分量为左儿子，y分量为右儿子。

传入示例：

```c++
		GLuint bvhChildren[] = {7/*rootBVH长度*/,0/*与显存对齐*/, 
			//root
			1,2, 
			3,4, 5,6,
			7,0, 7,30./360* 4294967296 /*右儿子用来表示旋转角度*/, 8,0, 7,0,
			//elememt0
			1,2,
			5+28/*contour索引，由于contour不定长，这里需要给到contour在elementIndex中位置*/,5+12/*style索引，在elementData中位置*/, 3,4,
					   5+36,5+12, 5+32,5+12,
			//elememt1
			1+0/*line索引，element中第几条*/,1 + 25

		};
		QVector4D bvhBound[] = { 
			//root
			QVector4D(-1,-1,1,1),
			QVector4D(-0.9,-0.9,-0.1,0.9),  QVector4D(0.1, -0.9,0.9,0.9), 
			QVector4D(-0.8,-0.8,-0.2,-0.1),  QVector4D(-0.7,0.2,-0.2,0.7), QVector4D(0.2,-0.8,0.8,-0.1), QVector4D(0.2,0.1,0.8,0.8),
			//elememt0
			QVector4D(-1,-1,1,1),
			QVector4D(-1,-0.5,1,1),	QVector4D(-1,-1,1,0.5),
									QVector4D(-1,-1,1,-0.5), QVector4D(-1,-0.5,1,0.5),
			//elememt1
			QVector4D(-1,0,1,1),
		};

		GLuint elementOffset[] = {
			//element0
			7, //elementBvhRoot
			5, //elementBvhLength
			0, //pointsOffset
			0, //linesOffset
			//element1
			12, //elementBvhRoot
			1, //elementBvhLength
			19, //pointsOffset
			40, //linesOffset
		};

		GLuint elementIndex[] = {
			//element0
			//lines, 全部当作三阶贝塞尔, 每条线四个点索引
			0,2,2,4,
			0,0,1,1,
			1,1,4,4,
			1,1,5,5,
			4,4,5,5,
			1,1,3,3,
			3,3,5,5,
			//contours, 第一个元素指明轮廓段数，后面为lines索引
			3, 0,1,2,
			3, 2,3,4,
			3, 3,5,6,

			//element2
			//lines
			0,1,2
		};


		GLfloat elementData[] = {
			//element0
			//points
			-1,0.5, -1,-0.5, 0,1, 0,-1, 1,0.5, 1,-0.5,
			//fillStyle
			//fill
			0, 
			//fillType
			0, //单色
			//fillColorMetallicRoughness
			1,1,0, 0,0.8,

			//element1
			//points
			-1,0.5, 0,1, 1,0.5,
			//strokeStyle
			//stroke
			1,
			//strokeWidth
			0.02,
			//strokeEndType
			0, //圆角
			//strokeFillType
			0, //单色
			//strokeFillColorMetallicRoughness
			0,1,0, 0,0.8
		};
```

