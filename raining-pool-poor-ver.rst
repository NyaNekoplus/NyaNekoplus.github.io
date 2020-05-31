丐版池塘夜雨
===============================

:id: raining-pool-poor-ver
:translation_id: raining-pool-poor-ver
:lang: zhs
:date: 2020-03-30 17:00
:tags: toy, c++, cg

写在前面
----------------------
因为对图形学比较感兴趣，前段时间陆续看了fundamentals of computer graphics、raytracing系列，了解了一些
图形学的基础知识，也手撸了一遍光追(丐中丐版)。出来的图的效果 :del:`看着还行` ：img，img。

如前面所写的，我之前所了解的都是图形学中离线渲染的部分。众所周知，虽然ray tracing所呈现的效果相对真实，但
如果不加一些黑魔法优化的话，效率简直不能看。上面的结果每像素1000samples，用时5min..效率感人。离线渲染坑太
多，不是一时半会儿可以搞定的；图形学中还有另外一个分支实时渲染，就像名字中所描述的，效率大都能做到实时，也是
游戏开发的主力，但相对于基本遵循物理定律的光追，真实度方面肯定有所差距。

之前看到过文刀秋二大大的 `一篇文章 <https://www.zhihu.com/question/29504480/answer/44764493>`_ ，正好这
学期了解了数据结构，与文中的背景基本相似。虽然大作业的题目并不是文中的“池塘夜雨”，但由于学校出的题目过于无聊，
索性就尝试一下，顺便学习一些实时渲染方面的知识。

先放上结果：

.. image:: {static}/images/rainingpool.PNG
    :alt: 池塘夜雨

完成了：天空盒、基本的水面效果(折射反射)、波纹、pbr光照地形、雨点(指雪花)

环境需求：OpenGL3.3-32bit，相对应的GLFW，glm0.9

下面记录一下实现时卡住的地方
------------------------------
- 雨

记得我是从雨点开始着手这个程序的；一开始想用Geometry Shader给整成粒子特效，写出来之后发现跟想象的效果的差距
有点大，粒子完全就是个像素点，一点都不像雨；思来想去没有头绪，干脆直接用三角形画个球代替一下，于是就有了现在
这样圆形的雨(其实连飘落方式也给写成雪的形式了，或许叫池塘夜雪更合适- -)。思路比较简单，实时渲染中的物体实际是由无
数个三角形面片组成的，所以只需想象一下把N个三角形近似成一个球就明白了。想要OpenGL画一个模型，需要提供模型的顶点
数据，所以问题就变成给定球心和半径，求球面的任意一点的坐标。这时可以将球像地球仪一样按经线和纬线分割；经过切割
后球面可被近似看为由N个矩形组成，一个矩形又由两个三角形组成；计算实际就是球坐标转笛卡尔坐标。灵魂画图：

.. image:: {static}/images/sample.PNG
    :alt: 样例

需要注意的是OpenGL中渲染三角形时按照顶点的顺序来区分正反面，默认逆时针为正面，顺时针为反面；在下面的反射会
用到。球坐标中的θ∈[0, π]， φ∈[0,2π]，代码如下。

.. code-block:: console

    void GetPoint(std::vector<float>& p,float u, float v, float r) {
        constexpr float pi = glm::pi<float>();
        float z = r * std::cos(pi * u);
        float x = r * std::sin(pi * u) * std::cos(2 * pi * v);
        float y = r * std::sin(pi * u) * std::sin(2 * pi * v);
        p.push_back(x);
        p.push_back(y);
        p.push_back(z);
        p.push_back(u);
        p.push_back(v);
    }
    (std::vector<float>& vertex, size_t longitude, size_t latitude, float r) {
        float longitude_step = 1.0f / longitude;
        float latitude_step = 1.0f / latitude;
        size_t offset = 0;
        for (size_t lo = 0; lo < longitude; lo++) {
            for (size_t lat = 0; lat < latitude; lat++) {
                std::vector<float> point1, point2, point3, point4;
                GetPoint(point1,lo * longitude_step, lat * latitude_step, r);
                GetPoint(point2,(lo + 1) * longitude_step, lat * latitude_step, r);
                GetPoint(point3,(lo + 1) * longitude_step, (lat + 1) * latitude_step, r);
                GetPoint(point4,lo * longitude_step, (lat + 1) * latitude_step, r);
            }
        }
    }

- 第二个解决的是天空盒

`这里 <https://learnopengl.com/PBR/IBL/Diffuse-irradiance>`_ 的介绍非常详尽。需要注意的是
从Equirectangular Map转换到Cube Map时每次循环都要glclear清一下深度和颜色缓冲...之前就是因为漏写了这一句调试了一下午..
说多了都是泪。

解决了雨点和天空盒后，能看到的效果是一个飘雨的广袤世界，就像这样：(天空盒资源取自 `sIBL <http://www.hdrlabs.com/sibl/archive.html>`_ )

.. image:: {static}/images/ame.PNG
    :alt: 天空盒和雨


- 池塘夜雨，总不能少了池塘。

最初用的是视差贴图和法线贴图结合，效果也很不错；但视差贴图完全是视觉上的trick，并不是
真实的高度，而之后的水面的实现是在另一个平面上，与地形基本处于同一高度，两个平面的深度冲突会导致只能显示一个平面。
于是转为用displacement mapping实现；displacement mapping实际上就是根据高度图调整顶点位置，这样生成的地形的高度
是真实的，但缺点是如果面数不够生成的地形就会棱角分明，很难看。解决方法是增加面(三角形)的数量，但这样开销也会增大；
生成顶点的代码如下：

.. code-block:: console

    (size_t rowLen, size_t colLen) {
	std::vector<glm::vec3> tv;
	std::vector<glm::vec2> te;
	size_t triangleNum = (rowLen - 1) * (colLen - 1) * 2;
	double d = 2.0 / rowLen;
	for (double z = -1.0; z < 1.0; z += d) {
		for (double x = -1.0; x < 1.0; x += d) {
			tv.push_back(glm::vec3(x, 0.0, z));
			te.push_back(glm::vec2((x + 1.0) * 0.5, (z + 1.0) * 0.5)); // [-1.0,1.0]->[0.0,1.0]
		}
	}

题图为1023*1023*2个三角形，再加上法线贴图的效果。光照用了pbr+IBL， `这个教程 <https://learnopengl.com/PBR/Theory>`_
写的太好了，真正的 :del:`简单` 易懂。

- 最后是水面，也是最复杂的了。

使用了平面反射来实现水面的反射效果，相比于Screen Space Reflection(SSR)和对cube map采样在
效率和效果上都有优势。SSR的原理是获取当前视空间的深度、法线等信息，然后在后处理阶段通过深度信息得出场景中物体的位置,再
通过光线步进的方式获取反射颜色。方式是从当前点出发，沿着反射方向步进直到碰到物体为止；反射方向可以由法线得到，物体碰撞
则用深度来判断；缺点也很明显，既然只有当前视空间的深度、法线信息，那视空间之外的物体肯定是没有反射了。因此在室外的场景
很容易出现反射缺失的情况。另一种反射方案是将当前场景渲染6次到一个cube map上，再对cube map采样；只是看描述就知道这玩意
开销有多大了，每帧6次..简单的场景还好说，复杂一点直接gg。

而平面反射的原理是将正常camera变换到对称与平面的位置，再用对称相机渲染场景一次作为reflect texture，然后对这个texture
采样就得到反射效果。要将正常camera转换到对称位置，需要一个反射矩阵，下面是公式推导和代码实现。

.. panel-default::
    :title: Planar Reflection
        
        .. image:: {static}/images/planar.PNG
            :alt: Planar Reflection

.. code-block:: console

    glm::mat4 result = glm::mat4(1.0);
	float d = -glm::dot(normal, p);
	result[0][0] = 1.0 - 2 * normal.x * normal.x;
	result[0][1] = -2 * normal.x * normal.y;
	result[0][2] = -2 * normal.x * normal.z;
	result[0][3] = -2 * normal.x * d;

	result[1][0] = -2 * normal.x * normal.y;
	result[1][1] =  1.0 - 2 * normal.y * normal.y;
	result[1][2] = -2 * normal.y * normal.z;
	result[1][3] = -2 * normal.y * d;

	result[2][0] = -2 * normal.x * normal.z;
	result[2][1] = -2 * normal.z * normal.y;
	result[2][2] = 1 - 2 * normal.z * normal.z;
	result[2][3] = -2 * normal.z * d;

	result[3][0] = 0;
	result[3][1] = 0;
	result[3][2] = 0;
	result[3][3] = 1;

将获得的反射矩阵在观察空间中变换即可(先乘view再乘reflect)；但这并不是最终结果。当物体在平面下时，反射贴图本应不显示物体
的反射虚像，但由于camera的frustum是下图的形式，所以依然会显示虚像，视觉上的效果是物体翻了个面。解决方法是将frustum的近
平面与反射平面平齐。如下，参考了 `这个 <http://www.terathon.com/lengyel/Lengyel-Oblique.pdf>`_

.. panel-default::
    :title: FrustumClipping
        
        .. image:: FrustumClipping.png
            :alt: FrustumClipping

.. code-block:: console
    
    glm::vec4 viewSpacePlane = glm::transpose(glm::inverse(reflectView)) * plane;
	glm::vec4 ViewSpaceFraPlanePoint = glm::transpose(glm::inverse(projection)) * glm::vec4(sign(viewSpacePlane.x), sign(viewSpacePlane.y), 1, 1);
	glm::vec4 M4 = glm::vec4(projection[3][0], projection[3][1], projection[3][2], projection[3][3]);
	auto u = 2.0f * (glm::dot(M4,ViewSpaceFraPlanePoint) / glm::dot(ViewSpaceFraPlanePoint, viewSpacePlane));
	auto newViewSpaceNearPlane = u * viewSpacePlane;
	auto M3 = newViewSpaceNearPlane - M4;
	//glm::vec4 M3 = 2.0f * (glm::dot(M4, ViewSpaceFraPlanePoint) / glm::dot(plane, ViewSpaceFraPlanePoint) * C) - M4;
	projection[0][2] = M3.x;
	projection[1][2] = M3.y;
	projection[2][2] = M3.z;
	projection[3][2] = M3.w;

- 数据结构的内容

写了个循环队列来控制雨点和波纹的逻辑；每一次渲染循环将雨点出队，修改位置后在入队；如果已经飘落到湖面上则重置雨点位置，然后
往波纹队列中增加一个波纹(在雨点飘落的位置)。同时将波纹出队，判断生命周期是否结束后入队。

最后
------------
为什么是丐版？

因为真的就只是把文刀秋二大佬文章中的features做了出来而已..全部代码大概1000行多点， :del:`完全就是屎山` ;最终效果一般而且
还有一些bug

一共用了一个月左右，不得不提OpenGL的api简直就是反直觉，用起来总感觉膈应，而且也非常底层，什么都要自己写的感觉实在是酸爽..
最后就是写shader的时候没有任何语法联想，出错了只能看那短短几行的错误输出来定位..还好这次的shader并不复杂。

要说有什么实质性的收获的话，:del:`学会了如何调节单机游戏里的图形选项在一般的硬件上以最好画质运行` 代码水平提升了！

仓促间写的总结，应该会有很多纰漏，还有一些点也没来得及写出来。学校的作业还没写完，然而我又想玩游戏了；；总之有时间再改吧