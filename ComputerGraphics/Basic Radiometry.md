## Radiant Energy

Radiant Energy是辐射所产生的所有能量，用$Q$表示，单位为焦耳J(Joule)。

## Radiant Flux

Radiant Flux又被称为Radiant Power，是单位时间内辐射出来的能量，可以理解为功率。用$\Phi$表示，单位为瓦特W(Watt)，或流明lm(lumen)。

$$\Phi = \frac {dQ}{dt}$$

## Radiant Intensity

要理解Radiant Intensity，首先要明白[solid angle](https://en.wikipedia.org/wiki/Solid_angle)。

### Solid Angle

Solid Angle即立体角，从平面角延伸而来，平面里角的定义是$\theta = \frac l r$，空间角的定义是$\Omega = \frac A {r^2}$，即球面上某个区域的面积除以半径的平方，其单位是steradian。

![Solid Angle](../Images/Solid_angle.png)

立体角可以通过上图中的2个平面偏移角度$\theta$和$\phi$定义，把面积微分想象成一个矩形，矩形的长是一个弧长即$rsin\theta d\phi$，矩形的宽也是一个弧长即$rd\theta$，故而有空间角微分：

$$d\omega = \frac {dA} {r^2} = \frac {(rsin\theta d\phi)(rd\theta)} {r^2} = sin\theta d\theta d\phi$$

另球面为$S^2$，空间角微分对球面做积分的结果应为$4\pi$

$$\Omega = \int _{S^2} d\omega = \int _0^{2\pi} \int _0^{\pi} sin\theta d\theta d\phi = 4\pi$$

立体角通常可以表示一个单位向量。

明白了立体角就可以定义Radiant Intensity: 单位立体角内的Radian Flux:

$$I(\omega) = \frac {d\Phi}{d\omega}$$

单位为瓦特每立体角，或流明每立体角，流明每立体角又称为candela，简称cd。

根据上面的微分，我们也可以得到下面的积分推导：

$$\Phi = \int _{S^2} I d\omega = 4\pi I$$

故而有：

$$I = \frac {\Phi} {4\pi}$$

## Irradiance

Irradiance是单位面积内接收到的Radian Flux:

$$E(x) = \frac {d\Phi(x)}{dA}$$

单位为瓦特每平方名，或流明每平方米，流明每平方米又称为lux。

当然，这里省略了平面法线和光线方向的夹角，假设这个夹角为$\theta$，则有$cos \theta = \vec l \cdot \vec n$，分子应乘以$cos \theta$，与[[Shading]]中描述的一致。

## Radiance

Radiance又称为Luminance亮度，是单位面积、单位立体角的Radian Flux：

![Radiance](../Images/Radiance.png)

$$L(p, \omega) = \frac {d^2 \Phi(p, \omega)}{d\omega dA cos\theta}$$

单位为瓦特每立体角每平方米，或cd每平方米，cd每平方米又称为nit，即尼特(苹果产品的屏幕亮度参数经常看到的单位)。

根据Radiance的定义，又可以将Radiance理解为：

- 单位空间角的Irradiance: $L(p, \omega) = \frac {dE(p)}{d\omega cos\theta}$
- 单位面积内的Radiant Intensity: $L(p, \omega) = \frac {dI(p, \omega)}{dA cos\theta}$

![Irradiance](../Images/Irradiance_by_radiance.png)

根据上面的理解，结合这幅图，假设半球是$H^2$，则可以得到下面的推导：

$$dE(p, \omega) = L_i(p, \omega) cos\theta d\omega$$
$$E(p) = \int _{H^2} L_i(p, \omega) cos\theta d\omega$$

## Bidirectional Reflectance Distribution Function

Bidirectional Reflectance Distribution Function即双向反射分布函数，简称BRDF，其描述的是，从某个角度过来的光的radiance，经过物体表面的某个点的反射之后，会如何分配各个不同的立体角，各个不同的立体角的光的radiance是多少。入射光的立体角为$\omega_i$(incoming direction)，反射光的立体角为$\omega_r$(reflected direction):

![BRDF](../Images/Bidirectional_reflectance_distribution_function.png)

$$f_r(\omega_i -> \omega_r) = \frac {dL_r(\omega_r)}{dE_i(\omega_i)} = \frac {dL_r(\omega_r)}{L_i(\omega_i) cos\theta_i d\omega_i}$$

单位为每立体角，即$sr^{-1}$，描述的只是一个比例，而且是个分布函数，对于不同的反射光立体角$\omega_r$会取不同的值，比如纯粹的镜面就是在反射光方向上是1，其他方向上都是0，比如漫反射就是在所有方向取相同的值。因此不同的BRDF其实描述了不同的材质。

## The Reflection Equation

![The Reflection Equation](../Images/Reflection_Equation.png)

根据上面的所有定义，就可以得到光的反射方程：

$$L_r(p, \omega_r) = \int _{H^2} f_r(p, \omega_i -> \omega_r) L_i(p, \omega_i) cos\theta_i d\omega_i$$

即相机看到的光的radiance，各个方向射向着色点的光的radiance乘以BRDF的积分，而各个方向设想着色点的光的radiance又满足上面的这个方程，是个递归的过程。
