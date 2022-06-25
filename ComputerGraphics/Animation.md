# Dot Notation for Derivatives

在动画领域通常用点来表示导数，比如$x$表示举例，则：

$$\dot x = v$$
$$\ddot x = a$$

## Mass Spring System

![Mass Spring](../Images/Mass_spring.png)

根据胡克定律，可以定义上面这个弹簧，a点受到的朝向b点的力：

$$\vec {f_{a \rightarrow b}} = k_s \frac {\vec b - \vec a}{\lVert \vec b - \vec a \rVert} (\lVert \vec b - \vec a \rVert - l)$$

其中$l$为弹簧原始的长度，$\frac {\vec b - \vec a}{\lVert \vec b - \vec a \rVert}$这个表达式定义了力的方向的normalized向量。

但是如果这么模拟，因为能量守恒的原因，质点永远都停不下来，为了让质点停下来，可以通过a和b的相对速度定义一个反向的“摩擦力”。当a和b之间没有相对运动时，这个力不存在，只有当两者之间有相对运动时，这个力才会存在：

$$\vec f_b = -k_d(\frac {\vec b - \vec a}{\lVert \vec b - \vec a \rVert} \cdot (\dot b - \dot a))\frac {\vec b - \vec a}{\lVert \vec b - \vec a \rVert}$$

其中$\frac {\vec b - \vec a}{\lVert \vec b - \vec a \rVert} \cdot (\dot b - \dot a)$这一项是一个点乘，实际上算的是相对速度在力方向上的投影，如果相对速度方向和力的方向垂直，则同样不存在这个力。

可以把弹簧质点系统推广到平面和空间中：

![Mass spring sheet](../Images/Mass_spring_sheet.png)
![Mass spring block](../Images/Mass_spring_block.png)

比如讲弹簧质点系统推广到平面，就可以模拟布料的效果：

![Mass spring plane](../Images/Mass_spring_plane.png)

- 蓝线：模拟以对角线方向拉扯布料时，布料内部的作用力防止布料发生形变的效果
- 红线：每隔一个质点在水平和竖直方向再连弹簧，模拟布料对折时布料内部所产生的排斥力，避免布料像纸一样对折后重合
- 通常红线的作用力要比蓝线弱很多

## Particle Simulation

在做粒子模拟时通常都知道在某个位置和时间时的速度：

$$v(x, t)$$

因此粒子模拟的本质是在求解一个Ordinary Differential Equation常微分方程：

$$\frac {\mathrm d x}{\mathrm d t} = \dot x = v(x, t)$$

通常可以使用欧拉法来求解这个方程：

$$x^{t + \Delta t} = x^t + \Delta t \dot x^t$$
$$\dot x^{t + \Delta t} = \dot x^t + \Delta t \ddot x^t$$

但是欧拉法非常不稳定：

![Euler's Method errors](../Images/Euler's_method_errors.png)

数值上误差通常用$O(h)$这种形式来表示，$O(h)$表示误差是一阶的，$h$指的是步长，在这里就是$\Delta t$，显然$\Delta t$越小误差越小，$\Delta t$减少一半误差也会减少一半。$O(h^2)$表示误差是二阶的，$\Delta t$减少一半误差会变成原来的$\frac 1 4$，所以阶数越高越好。

通常有以下几种方法可以减少误差。

### Midpoint Method / Modified Euler

![Modified Euler](../Images/Modified_euler.png)

这个方法的思路很简单，首先在$\Delta t$上求$x^{t + \Delta t}$，然后求$x$和$x^{t + \Delta t}$的中点$x_{mid}$，以$x_{mid}$的速度作为初始点的速度重新做欧拉算法。实际上就是用一个更能表示$\Delta t$时间内的平均速度来求位移。

$$x_{mid} = x(t) + \frac {\Delta t} 2 v(x(t), t)$$
$$x(t + \Delta t) = x(t) + \Delta t v(x_{mid}, t)$$

将这个式子展开后：

$$x^{t + \Delta t} = x^t + \frac {\Delta t} 2 (\dot x^t + \dot x^{t + \Delta t})$$
$$\dot x^{t + \Delta t} = \dot x^t + \Delta t \ddot x^t$$
$$x^{t + \Delta t} = x^t + \Delta t \dot x + \frac {(\Delta t)^2} 2 \ddot x^t$$

可以发现，Modified Euler实际上就是在Euler method的基础上加了一个二次项。

### Adaptive Step Size

![Adaptive Step Size](../Images/Adaptive_step_size.png)

这个方法的本质是自适应一个合适的$\Delta t$来进行模拟，思路是这样：

- 根据欧拉法算出$x_T$
- 将$\Delta t$平均分成2段，在这2段都用欧拉法，算出$\Delta t$时的位置$x_{\frac T 2}$
- 计算$\lVert x_T - x_{\frac T 2} \rVert$，和预设的一个阈值threshold比较，如果小于阈值，则说明小的和大的$\Delta t$计算的结果没有太大的区别，则用$x_T$作为结果；如果大于阈值，说明误差很大，则继续将$\frac {\Delta t} 2$进行二分，重复欧拉法、对比误差，知道误差小于阈值为止

上面这个算法实际上就是在不同的点应用不同的$\Delta t$，达到一种自适应的效果。

## Implicit Euler Method

隐式欧拉法实际上是在用$\Delta t$时刻的速度和加速度计算位移：

$$x^{t + \Delta t} = x^t + \Delta t \dot x^{t + \Delta t}$$
$$\dot x^{t + \Delta t} = \dot x^t + \Delta t \ddot x^{t + \Delta t}$$

通常需要用牛顿法之类的方法求函数的零点。

上面这种隐式欧拉法的误差是$O(h)$，相对来说误差比较大，通常来说可以用Runge-Kutta Families的方法(龙格-库塔法)比如RK4解这种常微分方程，将误差变成$O(h^4)$。

初始定义：

$$\frac {\mathrm d y}{\mathrm d t} = f(y, t)$$
$$y(t_0) = y_0$$

RK4的解：

$$y_{n + 1} = y_n + \frac 1 6 h (k_1 + 2k_2 + 2k_3 + k_4)$$
$$t_{n + 1} = t_n + h$$

其中：

$$k_1 = f(y_n, t_n)$$
$$k_2 = f(y_n + h \frac {k_1} 2, t_n + \frac h 2)$$
$$k_3 = f(y_n + h \frac {k_2} 2, t_n + \frac h 2)$$
$$k_4 = f(y_n + hk_3, t_n + h)$$
