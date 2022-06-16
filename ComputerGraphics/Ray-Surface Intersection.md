## Ray Equation

$$r(t) = \vec o + t \vec d$$

其中$\vec o$是光线起点，$\vec d$是光线方向的单位向量，$0 \le t \lt +\infty$

## Ray intersection with sphere

球的方程：

$$(\vec p - \vec c)^2 - R^2 = 0$$

其中$\vec c$是球心，$R$是半径。

把光线方程代入球的方程：

$$(\vec o + t \vec d - \vec c)^2 - R^2 = 0$$

上面的方程只是一个关于$t$的一元二次方程，很容易求解。方程的解的数量代表了交点的数量，但需要注意$t$必须是非负数。

## Ray intersection with triangle

当计算光线于mesh的相交时，实际上需要计算的是光线和三角形的交点。

计算的思路很简单，可以先根据三角形的法线求出三角形所在平面的方程，求平面和光线的交点，再判断交点是否在三角形内部即可。

平面方程很容易计算，假设三角形的法向量为$(a, b, c)$，那么平面方程必然为:

$$ax + by + cx + d = 0$$

将三角形的任意一个点代入求出$d$的值即可。

接着把光线的方程代入上面的平面方程即可求出交点的坐标。

判断交点是否在三角形内部也非常简单，假设三个顶点分别为ABC，交点为P，那么：

- $\vec a = \vec {AB} \times \vec {AP}$
- $\vec b = \vec {BC} \times \vec {BP}$
- $\vec c = \vec {CA} \times \vec {CP}$

只要$\vec a$ $\vec b$ $\vec c$这3个向量同向，不存在方向相反的情况，那么点P必然在三角形内部，可以用叉乘的右手定则验证这一点。

上面的这个方法经过了2步，根据[[Barycentric Coordinates]][重心坐标](Barycentric%20Coordinates.md)的定义，实际上也可以得出下面的这条式子：

$$\vec o + t \vec d = (1 - b_1 - b_2)\vec {P_0} + b_1\vec {P_1} + b_2\vec {P_2}$$

根据线性代数，可以知道：

$$\begin {bmatrix} t \\ b_1 \\ b_2 \end {bmatrix} = \frac 1 {\vec {S_1} \cdot \vec {E_1}}\begin {bmatrix} \vec {S_2} \cdot \vec {E_2} \\ \vec {S_1} \cdot \vec S \\ \vec {S_2} \cdot \vec D \end {bmatrix}$$

其中：

$$\begin {cases} \vec {E_1} = \vec {P_1} - \vec {P_0} \\ \vec {E_2} = \vec {P_2} - \vec {P_0} \\ \vec {S} = \vec {o} - \vec {P_0} \\ \vec {S_1} = \vec {d} \times \vec {E_2} \\ \vec {S_2} = \vec {S} \times \vec {E_1} \end {cases}$$

## Ray intersection with Axis-Aligned Box

把Axis-Aligned Box看成3对无穷大的面的交集。

那么对于每对面，代入光线方程，都能算出一个$t_{min}$和$t_{max}$。

对于3个面，可以算出：

$$\begin {cases} t_{enter} = max(t_{x-min}, t_{y-min}, t_{z-min}) \\ t_{exit} = min(t_{x-max}, t_{y-max}, t_{z-max}) \end {cases}$$

判断光线与Axis-Aligned Box是否相交，只需要满足下列条件：

$$\begin {cases} t_{enter} \lt t_{exit} \\ t_{exit} \gt 0 \end {cases}$$

如果光线与Axis-Aligned Box相交且$t_{enter} \lt 0$则显然光线起点在Axis-Aligned Box内部。
