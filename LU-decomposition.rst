LU分解求逆矩阵
===============================

:id: LU-decomposition
:translation_id: LU-decomposition
:lang: zhs
:date: 2020-08-09 21:33
:tags: note, Math

最近在写一个小玩意，需要进行大量的4*4矩阵运算(乘法、求逆)，且对性能有
比较高的要求，所以特地研究一下。

在矩阵阶数比较小且算法不会过于复杂的前提下，改变矩阵乘法算法带来的效率提升可以小到忽略不计；
要得到明显的效率提升一般都是从底层入手，可以调整代码的执行顺序以针对访问内存进行优化或是使用
指令集，这是后话。乘法算法我所知的有Definition和Strassen。两种方法差距如下

Strassen： :math:`O(n^{2.81})` <----->Definition： :math:`O(n^3)` 

据查到的实验数据显示，只有当矩阵阶数在300以上时Strassen才会明显优于定义；
因我需要计算的只是4阶的方阵，所以直接用定义计算还方便一些。

虽然乘法方面不会对效率产生多少影响；但对于矩阵求逆来说，算法上的优化是有必要的。
如果使用定义法，先算出伴随矩阵后再除以行列式来求逆，其时间複雜度为O(n^4)(要算出每个元素的代数余子式)；
在大量运算的基础下势必会对运行效率造成影响。

除伴随矩阵法之外，我所知的矩阵求逆方法还有高斯消元法和LU分解法

高斯消元法(Gauss elimination)是考试常用方法，求A的逆矩阵时b为单位阵(Identify Matrix)，
过程是利用初等行变换令原矩阵和单位阵同时变换，当原矩阵转换单位阵时元单位阵即为逆矩阵。
编程实现时一般使用其变种Gauss-Jordan elimination，
`求解过程 <https://baike.baidu.com/item/%E9%AB%98%E6%96%AF-%E8%8B%A5%E5%B0%94%E5%BD%93%E6%B6%88%E5%85%83%E6%B3%95/19969775?fr=aladdin>`_

算法比较简单，按照规则逐行约化就行。
Gauss elimination的结果是上三角矩阵，要得到最终解还需要一个代入数值求解的过程；
而Gauss-Jordan elimination的结果是单位阵，可以直接得到线性方程组的解。

LU分解(LU decomposition)是计算机中常用的求线性方程组的算法，现有的大部分数学库的
求线性方程组算法都基于LU分解设计；既然是用于求线性方程组，那自然也能用于求逆。

它的想法很容易理解。数值计算中有一个经验定律：能不直接算 :math:`A^{-1}` 就不算。
所以将一个方阵分解成一个上三角阵和一个下三角阵，它们的乘积等于元矩阵就是LU分解
的基本思想。

:math:`A = LU`

根据基本定理有： :math:`A^{-1} = (LU)^{-1} = U^{-1}L^{-1}`

L、U分别为下、上三角矩阵，对它们分别求逆显然比直接对原矩阵求逆要来得简单；
另一个求逆思路是解线性方程组。

LU分解用于求解线性系统的思路如下(解线性方程组)

.. math::

	Ax = b

	A = LU

	LUx = b

	令 y = Ux,则Ly = b

	Ux = y

当b为单位阵时x为逆矩阵。

伪代码( `来源 <http://iacs-courses.seas.harvard.edu/courses/am205/slides/am205_lec07.pdf>`_ )

.. image:: {static}/images/LU.PNG
    :alt: LU
    :align: center

但这样的算法在计算机上实现时会有运算问题，当Ujj为0时分母为0；为了避免这一点，需要使用LU分解
的变种LUP分解。



其原理和LU分解相似，但逐行消元时要选择该列最大的元素来规避除0(还有保持数值稳定，
令Lij<=1)。实现的方法是引入一个置换矩阵P，在消元时将该列绝对值最大的项置换到当前行，如果列绝对值
最大值为0，则这个矩阵是奇异的，不存在逆。

:math:`L_{n-2}P_{n-2}...L_1P_1L_0P_0A = U`

:math:`PA = LU`

:math:`P = \prod_{i=0}^{n-2}P_i`

伪代码( `来源 <http://iacs-courses.seas.harvard.edu/courses/am205/slides/am205_lec07.pdf>`_ )

.. image:: {static}/images/LUP.PNG
    :alt: LUP
    :align: center


我个人的LUP求逆实现。Matrix4x4是我封装的4x4矩阵类，可视作2维数组

.. code-block:: console

    inline bool LUPdecomposition(Matrix4x4& A, Matrix4x4& L, Matrix4x4& U, int* P) {
		const int n = 4;
		int row = 0;
		for (int i = 0; i < n; ++i)
			P[i] = i;
		for (int i = 0; i < n-1; ++i) {
			Float max = 0.0;
			for (int j = i; j < n; ++j) { // 最大元素所在行
				Float cur = std::abs(A[j][i]);
				if (cur > max) {
					max = cur;
					row = j;
				}
			}
			if (max == 0) { std::cout << "Singular matrix" << "A:\n" << A << "L:\n" << L << "U:\n" << U; return false; }
			
			if (i != row) { std::swap(P[i], P[row]); } //更新置换矩阵

			for (int j = 0; j < n; ++j) // 应用置换矩阵后的A
				std::swap(A[i][j], A[row][j]);
			
            // LU分解，更新A
			Float u = A[i][i], l = 0.0f;
			for (int j = i + 1; j < n; ++j) {
				l = A[j][i] / u;
				A[j][i] = l;
				for (int k = i + 1; k < n; ++k) {
					A[j][k] = A[j][k] - l * A[i][k];
				}
			}
		}
		// 从A中作出L、U
		for (int i = 0; i < n; ++i) {
			for (int j = 0; j <= i; ++j) { //下三角
				if (j == i)
					L[i][j] = 1.0;
				else
					L[i][j] = A[i][j];
			}
			for (int k = i; k < n; ++k) { // 上三角
				U[i][k] = A[i][k];
			}
		}
		return true;
	}

	inline void LUPsolve(const Matrix4x4& L, const Matrix4x4& U,const int* P,const Float* b,Float* result) {
		const int n = 4;
		Float y[n];
		for (int i = 0; i < n; ++i) {           // Ly = b 解线性方程组
			y[i] = b[P[i]];
			for (int j = 0; j < i; ++j) {
				y[i] = y[i] - L[i][j] * y[j];
			}
		}
		for (int i = n - 1; i >= 0; i--) {      // Ux = y
			result[i] = y[i];
			for (int j = n - 1; j > i; j--) {
				result[i] = result[i] - U[i][j] * result[j];
			}
			result[i] /= U[i][i]; // x系数可能不为1
		}
	}
	inline Matrix4x4 Inverse(const Matrix4x4& m) {
		const int n = 4;
		Matrix4x4 result;
		Matrix4x4 IdentityM;
		Matrix4x4 A, L, U;
		int P[n]{ 0 };
		memcpy(A.mat, m.mat, 16 * sizeof(Float));
		LUPdecomposition(A, L, U, P);
		for (int i = 0; i < n; ++i) {   // 每次传入单位矩阵的一列，最后的结果拼接起来再转置为逆矩阵
			LUPsolve(L, U, P, IdentityM[i], result[i]);
		}
		return (Transpose(result));
	}

时间複雜度方面，LU(P)分解和高斯消元都约为 :math:`O(n^3)` ，看起来没有必要去使用较复杂的LU(P)分解；网上有很多总结都说
LU(P)分解优于高斯消元，但这种说法是片面的，它们的效率高低取决于不同的条件。
对两种方法的时间複雜度细化分析。LU分解：分解成LU两个矩阵约为 :math:`O(n^3)` ，解Ly = b和Ux = y都约为 :math:`O(n^2)` ，所以总
时间複雜度约为 :math:`O(n^3)+O(n^2)` 。高斯消元： :math:`O(n^3)` 。

以上分析都是针对求解单个线性方程组。但可以发现的是，LU分解第一步的结果L、U矩阵可以存储下来，即 :math:`O(n^3)` 的操作可以
只做一次；设要求的线性方程组数量为m，则LU分解的複雜度为 :math:`O(n^3)+mO(n^2)` ，而高斯消元的複雜度为 :math:`mO(n^3)` ；
所以对于求解多个线性方程组(多个b)的情况，使用LU分解更优于高斯消元，且求解的线性方程组数量越多则越明显；反之亦然，如果线性方程
的数量仅有一个，那么高斯消元效率比较高。对于求逆矩阵，只需要解一个b；按上面所说，用高斯消元会比较快。下面是测试代码

.. code-block:: console

	int main(int argc, char* argv[]) {
		vmath::Matrix4x4 origin;
		for (int i = 0; i < 4; i++) 
			for (int j = 0; j < 4; j++) 
				origin[i][j] = (int)(drand48() * 10.0); // [0,10)
			
		std::cout << "origin:\n" << origin;
		std::chrono::steady_clock::time_point LUPstart = std::chrono::steady_clock::now();

		vmath::Matrix4x4 LUPinv = vmath::Inverse(origin);
		std::chrono::steady_clock::time_point LUPend = std::chrono::steady_clock::now();
		std::chrono::duration<Float> LUP_UseTime = std::chrono::duration_cast<std::chrono::duration<Float>>(LUPend - LUPstart);

		std::chrono::steady_clock::time_point GJstart = std::chrono::steady_clock::now();
		vmath::Matrix4x4 Gaussinv = vmath::GaussJordan(origin);
		std::chrono::steady_clock::time_point GJend = std::chrono::steady_clock::now();
		std::chrono::duration<Float> GJ_UseTime = std::chrono::duration_cast<std::chrono::duration<Float>>(GJend - GJstart);

		std::cout << "Gauss Jordan Inverse:\n" << Gaussinv;
		std::cout << "LUP Inverse:\n" << LUPinv;

		std::cout << "Multiply:\n" << Gaussinv * origin;
		std::cout << "LUP Use time: " << LUP_UseTime.count() << "s\n";
		std::cout << "GJ Use time: " << GJ_UseTime.count() << "s\n";
	}

上面代码的测试结果(O2)：

.. image:: .. image:: {static}/images/LUPGUASStimetest.PNG
    :alt: Test

可以看到结果与元矩阵相乘后的确是一个单位阵。LUP和GJ分别是0.0009ms、0.0007ms，虽然差距很小，但的确是GJ更快一些。

放着一堆带优化的开源数学库不用自己去造轮子，我也是有够无聊(

.. math::

	\begin{matrix}
	1 & 0 & 0 \\
	0 & 1 & 0 \\
	0 & 0 & 1
	\end{matrix} 
	