Barycentric Coordinates即重心坐标，重心坐标用于计算三角形的插值。

对于任意三角形ABC，假设A的坐标为A，B的坐标为B，C的坐标为C，那么对于这个三角形所在的平面上的任意一个点(x, y)，都可以表示成：

$$(x, y) = \alpha A + \beta B + \gamma C $$

其中$\alpha + \beta + \gamma = 1$。

那么这个$(\alpha, \beta, \gamma)$就被称为Barycentric Coordinates，即重心坐标。

比如很显然，A点的重心坐标就是$(1, 0, 0)$。

特别的，如果点在ABC内部，那么必然有$\alpha \ge 0$且$\beta \ge 0$且$\gamma \ge 0$，而且这3个值可以通过下面这个求面积的算法算出来：

![Barycentric Coordinates](../Images/Barycentric_coordinates.png)

通过向量叉乘求面积，也可以很容易的得出下面的公式：

$$\alpha = \frac {-(x - x_B)(y_C - y_B) + (y - y_B)(x_C - x_B)}{-(x_A - x_B)(y_C - y_B) + (y_A - y_B)(x_C - x_B)}$$
$$\beta = \frac {-(x - x_C)(y_A - y_C) + (y - y_C)(x_A - x_C)}{-(x_B - x_C)(y_A - y_C) + (y_B - y_C)(x_A - x_C)}$$
$$\gamma = 1 - \alpha - \beta$$

使用重心坐标，当我们知道三角形三个顶点的任意属性之后，就可以对该属性在三角形内部进行插值。

不过有一点需要<font color="red">特别注意</font>，重心坐标在投影到2D屏幕上之后是会变化的，所以当需要插值的属性包含三维信息时，比如对Z进行插值，对法线进行插值，则需要使用投影变换之前的顶点坐标来求重心坐标。如果只有投影之后的坐标，则需要根据投影矩阵的逆矩阵将坐标转换回3D空间上。
