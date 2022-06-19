[[Basic Radiometry#Bidirectional Reflectance Distribution Function]]中提到，不同的BRDF描述了不同的材质。

## Diffuse BRDF

光线射到某个点之后，朝各个方向反射出去的光是均匀分布的，就是Diffuse的情况。

![Diffuse BRDF](../Images/Diffuse_BRDF.png)

所以对于漫反射，BRDF是一个常数，根据[[The Rendering Equation]]我们可以得到以下方程：

$$L_o(\omega_o) = \int_{H^2} f_r L_i(\omega_i) \cos \theta_i \mathrm d \omega_i$$

根据假设光线的radiance是常数，而漫反射的BRDF又是常数，所以可以得到：

$$L_o(\omega_o) = f_rL_i \int _{H^2} \cos \theta_i \mathrm d \omega_i = \pi f_r L_i$$

如果漫反射表面不吸收任何光，那么$L_o$应该与$L_i$完全相等，此时有$f_r = \frac 1 \pi$，但漫反射肯定会吸收一定的光，所以对于漫反射的BRDF，有：

$$f_r = \frac {\rho}{\pi}$$

其中$\rho \in (0, 1)$，被称为albedo，跟这一点的颜色值有关系。

## Glossy Material

光线射到某个点之后，如果朝不同反射方向附近的方向分布反射光，就是Glossy Material的情况。

![Glossy Material](../Images/Glossy_material.png)

## Ideal reflective / refracive material

光线射到某个点之后，一部分光被反射，一部分光被折射，就是水或者玻璃的情况。

![Ideal reflective / refractive material](../Images/Ideal_refractive_material.png)

![reflection](../Images/Perfect_specular_reflection.png)

根据上图可以知道：

$$\vec {\omega_o} + \vec {\omega_i} = 2 \cos \theta \vec n = 2 (\vec {\omega_i} \cdot \vec n)\vec n$$

即可求出反射方向。

![Snell's Law](../Images/Snell's_law.png)

考虑折射时，某一种介质有一个折射率，用$\eta$表示，常见介质的折射率如下：

Medium | $\eta$
--- | ---
Vacuum | 1.0
Air (sea level) | 1.00029
Water (20°C) | 1.333
Glass | 1.5 - 1.6
Diamond | 2.42

那么入射角和折射角满足下面的等式：

$$\eta_i \sin \theta_i = \eta_t \sin \theta_t$$

如果把正弦函数转换成余弦，可以把上面的方程转换为下面的形式：

$$\cos \theta_t = \sqrt {1 - (\frac {\eta_i}{\eta_t})^2(1 - \cos ^2 \theta_i)}$$

余弦值肯定是要有意义的，即根号下的表达式必须大于0，所以当$\eta_i > \eta_t$时，很有可能无法完成折射，此时该物理现象被称为全反射。也就是原本折射的光线也被反射回去。比如从水中往空气看：

![Snell's Window](../Images/Total_internal_reflection.png)

## Fresnel Term

当入射光线的角度取不同的值时，被折射的光的比例时不一样的，Fresnel Term(菲涅尔项)描述的就是这个比例的值。

比如绝缘体的Fresnel Term:

![Dielectric fresnel term](../Images/Dielectric_fresnel_term.png)

导体的Fresnel Term:

![Conductor fresnel term.png](../Images/Conductor_fresnel_term.png)

蓝线和绿线表示光振动方向的2个极。Fresnel Term的计算有明确的公式：

$$R_s = \lvert {\frac {n_1 \cos \theta_i - n_2 \cos \theta_t} {n_1 \cos \theta_i + n_2 \cos \theta_t}}\rvert^2$$
$$R_p = \lvert {\frac {n_1 \cos \theta_t - n_2 \cos \theta_i} {n_1 \cos \theta_t + n_2 \cos \theta_i}}\rvert^2$$
$$R_{eff} = \frac 1 2 (R_s + R_p)$$

但是这么计算太复杂了，通常使用Schlick's approximation来做近似：

$$R(\theta) = R_0 + (1 - R_0)(1 - \cos \theta)^5$$
$$R_0 = (\frac {n_1 - n_2} {n_1 + n_2})^2$$

## Microfacet Material

微表面材质模型是一种常用的材质模型。

![Microfacet Theory](../Images/Microfacet_Theory.png)

微表面模型认为：

- 从远处看，看到的是宏观的表面，即材质
- 从近处看，看到的是几何图形，每一个微小的表面遵从镜面反射，认为微表面是微笑的镜面，有自己的法线

因此微表面模型认为微表面法线的分布决定了宏观表面的材质。

比如下面这种微表面法线分布是glossy:

![Glossy microfacet](../Images/Glossy_microfacet.png)

下面这种微表面法线分布是diffuse:

![Diffuse microfacet](../Images/Diffuse_microfacet.png)

![Microfacet BRDF](../Images/Microfacet_BRDF.png)

微表面模型能够准确计算出BRDF：

- $F(i, h)$: 描述了微表面的Fresnel Term
- $G(i, o, h)$: 描述了有多少光会被微表面本身遮挡，比如当光几乎平行于宏观表面射过来时，由于微表面凹凸不平，前面的微表面很可能挡住后面的微表面，导致后面的微表面接收不到光照
- $D(h)$: 描述了有多少微表面的法线和half vector的方向相同，因为必须在法线相同时才能在微表面上反射这个方向的光，毕竟微表面上都是镜面反射

![Isotropic and anisotropic materials.png](../Images/Isotropic_and_anisotropic_materials.png)

各向同性和各向异性的材质也可以用微表面解释：

- 各向同性的材质：微表面的法线分布在各个方向上的分布是均匀的
- 各向异性的材质：微表面的法相分布在各个方向上的分布是不均匀的

对于各向异性的BRDF，满足下面的关系：

$$f_r(\theta_i, \phi_i ; \theta_r, \phi_r) \neq f_r(\theta_i, \theta_r, \phi_r - \phi_i)$$
