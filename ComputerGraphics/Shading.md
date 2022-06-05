着色分为3种情况：

- Specular highlights: 高光
- Diffuse reflection: 漫反射
- Ambient lighting: 环境光

以下讨论基于Blinn-Phong Reflectance model。

## Diffuse reflection

漫反射将分析对象视为一个shading point：

![shading point](../Images/Shading_point.png)

- viewer direction: $\vec v$观测者方向
- surface normal: $\vec n$法线
- light direction: $\vec l$光照方向

这3个向量都是标准向量。

对于漫反射的shading point，无论viewer direction是什么，观测到的结果都是一致的，因为假设光射到漫反射shading point之后会朝着半球状的各个方向散发出去。

假设点光源强度为I，光照方向和法线的夹角为$\theta$，点光源到shading point的距离为r。

首先光强度和距离的关系应该为$\frac {I}{r^2}$，因为光以球面的形式从点光源逐渐向外扩散。

shading point接收到的光强于$\theta$的关系应为$cos \theta$，即$\vec l \cdot \vec n$，因为夹角越大所能接受到的光的能量就越低。

所以最终得到：

$$L_{d} = k_{d} \frac {I}{r^2}max(0, \vec l \cdot \vec n)$$

其中$L_{d}$为diffusily reflected light，即漫反射反射的光强，$k_d$为diffuse coefficent，即漫反射系数，比如颜色值。

## Specular term

当观测者方向和光照的反射方向非常接近时，就会看到高光。

计算光照的反射方向需要根据法线进行计算，虽然也不是特别复杂，但是有更简单的办法：可以计算光照方向$\vec l$和观测者方向$\vec v$的半程向量(角平分线方向)，根据平行四边形法则，半程向量非常容易计算：

$$\vec h = bisector(\vec v, \vec l) = \frac {\vec v + \vec l}{||\vec v + \vec l||}$$

计算出半程向量之后，只需要计算半程向量和法线是否靠近即可，这个用点乘就行了。所以有：

$$L_s = k_s\frac {I}{r^2}max(0, \vec n \cdot \vec h)^p$$

其中$L_s$为specularly reflected light，即高光反射的光强，$k_s$为specular coefficent，即高光反射系数。

p这个指数的引入是为了提升高光的灵敏程度，如下图所示，p约大，可以shade为specular term的角度范围越窄，通常Blinn-Phong模型的p会取到100 - 200之间。

![cosine power plot](../Images/Cosine_power_plots.png)

## Ambient term

环境光非常复杂，其光照来自四面八方反射过来的光的结果，在Blinn-Phong模型中认为任何一个被当作Ambient term的shading point，即没有光直接照射的地方，光亮程度是一个常数，所以有：

$$L_a = k_a I_a$$

其中$L_a$为reflected ambient light，即环境光反射的光强，$k_a$为ambient coefficent，即环境光系数，$I_a$为上面所提及的常数光亮程度。

## shading frequency

如何选取shading point就是shading frequency要做的事情，通常来说有3种策略：

![shading frequency](../Images/Shading_frequency.png)

从左到右分别为：

- Flat shading: 以每个三角形的面作为shading point
- Gouraud shading: 以每个三角形的顶点作为shading point，三角形内部的颜色通过插值完成，在计算三角形顶点的发现时，通常以该顶点相邻的面的法线全部加起来算个加权平均
- Phong shading: 以每个像素作为shading point，每个像素的法线时根据顶点法线插值得到的
