着色分为3种情况：

- Specular highlights
- Diffuse reflection: 漫反射
- Ambient lighting

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
