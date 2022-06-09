[推导文章](http://www.lighthouse3d.com/tutorials/glsl-12-tutorial/the-normal-matrix/)

如果要把法向量从模型的局部空间转换到世界空间，如果模型存在non-uniform scale，会出现以下问题：

![non-uniform scale](https://learnopengl.com/img/lighting/basic_lighting_normal_transformation.png)

原本垂直的法向量，会因为non-uniform scale而变得不垂直。

uniform scale是没有这个问题的，但是uniform scale会让法向量的长度变得不再是1，此时通过normalize向量即可解决这个问题。

把法向量从模型局部空间转换到世界空间的矩阵称为Normal Matrix。

假设现在要求Normal Matrix，将其设为$G$，Model Matrix为$M$，则有：

$$G = (M^{-1})^T$$

推导过程：

设$\vec T$为原本的平面上的任意一个标准向量，$\vec N$为原本的法向量。$\vec {T'}$(读作T prime)为$\vec T$转换后的结果，$\vec {N'}$(读作N prime)为$N$转换后的结果，这2个结果是正确计算后的结果。则以下等式成立：

$$\begin{cases} \vec T \cdot \vec N = 0 \\ \vec {T'} \cdot \vec {N'} = 0 \\ M\vec T = \vec {T'} \\ G\vec N = \vec {N'}\end{cases}$$

根据上面的等式，可以得到下面这条等式：

$$(G\vec N) \cdot (M\vec T) = 0$$

向量的点成也可以写成前一个向量的转置矩阵乘以后一个向量的矩阵，得到：

$$(G\vec N) \cdot (M\vec T) = (G\vec N)^T(M\vec T)$$

根据矩阵转置的性质，可以得到：

$$(G\vec N)^T(M\vec T) = \vec N ^ T G^T M \vec T$$

展开后就是：

$$\vec {N'} \cdot \vec {T'} = (G\vec N) \cdot (M\vec T) = \vec N ^ T G^T M \vec T = 0$$

而又有：

$$\vec {N'} \cdot \vec {T'} = \vec N \cdot \vec T = \vec N ^ T \vec T = 0$$

则必然有($I$为标准矩阵)：

$$G^T M = I$$

故而得到：

$$G = (M^{-1})^T$$
