一个简单的BMP编码函数 TOYBMP()
===============================

:id: simple-c-function-bmp
:translation_id: simple-c-function-bmp
:lang: zhs
:date: 2019-08-04 21:22
:tags: C, toy

写在前面
------------------

计算机图形学(CG)，是一种使用数学算法将二维或三维图形转化为计算机
显示器的栅格形式的科学。其主要的研究内容就是研究如何在计算机中表
示图形，以及利用计算机进行图形的计算、处理和显示。

我对这个领域也有相当的兴趣。为了更好地学习图形学，我尝试写了
`TOYBMP() <https://github.com/NyaNekoplus/toybmp>`_ 。
以MIT协议发布。这个函数由C语言实现，可以将数据写入24-bit RGB 
或 32-bit RGBA 无压缩的BMP。代码长度为25行。

代码实现如下(`toybmp.h <https://github.com/NyaNekoplus/toybmp/blob/master/toybmp.h>`_)：

.. image:: {static}/images/toybmp.PNG

用法
------------------

.. code-block:: c

    #include <stdio.h>
    #include "toybmp.h"
    unsigned char rgb[256 * 256 * 3];
    int main(void) {
        unsigned char* p = rgb;
        for (int y = 0; y < 256; ++y) {
            for (int x = 0; x < 256; ++x) {
                *p++ = (unsigned char)x;
                *p++ = (unsigned char)y;
                *p++ = (unsigned char)128;
            }
        }
        toybmp(fopen("rgb.bmp", "wb"), 256, 256, p, 0);
    }

上面的代码输出此文件:

.. image:: {static}/images/rgb.bmp

函数声明:

.. code-block:: c

    /*!
	\brief Save a RGB/RGBA image in BMP format.
	\param TOYBMP_OUTPUT Output stream (by default using file descriptor).
	\param w Width of the image.
	\param h Height of the image.
	\param img Image pixel data in 24-bit RGB or 32-bit RGBA format.
	\param alpha Whether the image contains alpha channel.
    */
    void toybmp(TOYBMP_OUTPUT, unsigned w, unsigned h, const unsigned char* img, int alpha)


实现
-----------------

简单介绍下实现要点。不同于PNG与JPG格式，BMP的实现并不复杂。其基本结构如下。

- BitmapFileHeader 共14bit 描述bmp格式，显示文件大小
- BitmapInfoHeader 共40bit 描述位图的维度，色深，压缩方式
- BitmapData 位图像素数据

在位图文件头中值得注意的是，Windows中的数据是倒着显示的。即如果
一段数据显示为36 00 04 00，则实际数据为00 04 00 36。

在位图数据区中，由于位图信息头中的图像高度是正数，所以位图数据在文件中的排列
顺序是从左下角到右上角，以行为主序排列的。可以理解为标准的二维平面坐标系。
此外，24-bit RGB在BMP中按照BGR的顺序储存各通道的值。32-bit RGBA则按照BGRA
顺序储存。

最后是对齐规则。在Windows中，默认扫描的最小单位是4byte，如果数据按此规则对齐
则获取数据的速度会大大加快。也因此，BMP要求每行数据的长度必须是4的倍数，如果
不够则需要在每行末尾补零(比特填充)。

填充后每行的字节数可以表示为:
:code:`((宽度*位深+7)/8+3)/4`

两张调试过程中的错误图片:

.. image:: {static}/images/err0.bmp

.. image:: {static}/images/err1.bmp