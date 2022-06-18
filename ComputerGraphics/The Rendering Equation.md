根据[[Basic Radiometry#The Reflection Equation]]，我们知道某个点反射的光跟设想这个点的各个方向的光有关系，那么如果加上这个点本身的自发光Radiance，就可以得到渲染方程：

$$L_o(p, \omega_o) = L_e(p, \omega_o) + \int _{\Omega^+} L_i(p, \omega_i) f_r(p, \omega_i, \omega_o) (n \cdot \omega_i) d\omega_i$$

这个复杂的式子可以得到下面这个式子：

$$l(u) = e(u) + \int l(v) K(u, v)dv$$

即u点的radiance等于u点的自发光加上所有v点的radiance与$K(u, v)dv$的乘积的积分，$K(u, v)dv$又称为Kernal of equation。

进一步地，可以将其简化为一个线性地矩阵方程：

$$L = E + KL$$

该矩阵方程可以推导出：

$$L = (I - K)^{-1} E $$

根据[Binomial Theorem](https://en.wikipedia.org/wiki/Binomial_theorem)二项式定理：

$$L = (I + K + K^2 + K^3 + \cdots)E$$

即：

$$L = E + KE + K^2E + K^3E + \cdots$$

![Ray Tracing](../Images/Ray_tracing.png)

该等式的前两项是在光栅化着色中可以计算的部分，后面的部分则为光反射了若干次之后的着色，比如$K^2E$为反射了1次光的着色，$K^3E$为反射了2次光的着色，这个式子会逐渐收敛到一个值。
