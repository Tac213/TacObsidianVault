解[[The Rendering Equation]][渲染方程](The%20Rendering%20Equation.md)的过程即Path Tracing。

使用[蒙塔卡洛积分](https://zhuanlan.zhihu.com/p/146144853)可以将渲染方程简化为：

$$L_o(p, \omega_o) \approx \frac 1 N \sum _{i = i}^n \frac {L_i(p, \omega_i) f_r(p, \omega_i, \omega_o) (n \cdot \omega_i)} {pdf(\omega_i)}$$

根据这个等式实际上就可以写出下面的伪代码：

```
shade(p, wo)
    Randomly choose N directions wi~pdf
    Lo = 0.0
    For each wi
        Trace a ray r(p, wi)
        If ray r hit the light
            Lo += (1 / N) * L_i * f_r * cosine / pdf(wi)
        Else If ray r hit an object at q
            Lo += (1 / N) * shade(q, -wi) * f_r * cosine / pdf(wi)
    Return Lo
```

实际上就是一个递归的运算过程，但是这个算法有2个问题：

1. 递归次数过多，假设每次选100个随机数，则光线每反弹一次就乘一个100，指数级增长
2. 不存在递归的结束条件

第一个问题比较好解决，每次只取1个随机数即可，尽管这样有较大的误差，但是本身在像素点上时就会做随机取样，因此此处的取样可以产生一些误差也没关系。

第二个问题如果要解决的话是跟物理学违背的，因为光本身就会在场景中弹射无限次，不可能停止。不过可以通过改变采样策略来让递归停止。

此处可以采用[俄罗斯轮盘切割(Russian Roulette and Splitting)](https://www.pbr-book.org/3ed-2018/Monte_Carlo_Integration/Russian_Roulette_and_Splitting)的方法，在做计算之前先取一个概率$p$，然后做一次随机试验，如果随机试验的结果不满足我们定义的概率就直接return，如果满足结果则继续做刚才的计算，只不过返回的结果要除以$p$，这样能保证做多次随机试验的结果的值还是原本的那个值。

优化后的伪代码如下：

```
shade(p, wo)
    Manually specify a probability P_RR
    Randomly select ksi in a uniform dist. in [0, 1]
    If (ksi > P_RR) return 0.0;
    Randomly choose ONE direction wi~pdf(w)
    Trace a ray r(p, wi)
    If ray r hit the light
        Return L_i * f_r * cosine / pdf(wi) / P_RR
    Else If ray r hit an object at q
        Return shade(q, -wi) * f_r * cosine / pdf(wi) / P_RR
```

这个算法的pdf选取非常重要，如果均匀抽样，其实就是在碰运气，每次弹射的光的方向不一定是最重要的，就会导致渲染出来的结果不一定合理，比如下面的图，左边是均匀采样的结果，右边是重要性采样的结果。

![uniform sampling vs importance sampling](https://pic4.zhimg.com/80/v2-5fe1cff059c3dd83c0824fa9526bc193_720w.jpg)

一种可能的方法是，让每次采样的方向尽量靠近光源方向

![relationship between dw and dA](../Images/Path_tracing_dw_dA.png)

根据这个图，把$dA$映射到球面上，再取球半径的平方，可以得到$d\omega$和$dA$的关系：

$$\mathrm d\omega = \frac {cos \theta ^ \prime \mathrm{d}A} {\lVert \vec {x^\prime} - \vec x \rVert ^ 2}$$

故而渲染方程改写为：

$$L_o(p, \omega_o) = L_e(p, \omega_o) + \int _{A} L_i(p, \omega_i) f_r(p, \omega_i, \omega_o) \frac {(n \cdot \omega_i) (n^\prime \cdot -\omega_i)} {\lVert p^\prime - p \rVert^2} \mathrm d A$$


根据蒙特卡洛积分，上式简化为：

$$L_o(p, \omega_o) \approx \frac 1 N \sum _{i = i}^n \frac {L_i(p, \omega_i) f_r(p, \omega_i, \omega_o) (n \cdot \omega_i) (n^\prime \cdot -\omega_i)} {\lVert p^\prime - p \rVert ^2 pdf(A)}$$

故而得到算法伪代码：

```
shade(p, wo)
    # Contribution from the light source.
    Uniformly sample the light at x’ (pdf_light = 1 / A)
    Shoot a ray from p to x’
    If the ray is not blocked in the middle
        L_dir = L_i * f_r * cos θ * cos θ’ / |x’ - p|^2 / pdf_light

    # Contribution from other reflectors.
    L_indir = 0.0
    Test Russian Roulette with probability P_RR
    Uniformly sample the hemisphere toward wi (pdf_hemi = 1 / 2pi)
    Trace a ray r(p, wi)
    If ray r hit a non-emitting object at q
        L_indir = shade(q, -wi) * f_r * cos θ / pdf_hemi / P_RR

    Return L_dir + L_indir
```
