![Projection](../Images/Orthographic_Perspective_Projection.png)

这个变换过程总共包括MVP，3步：

- M: Model transformation
- V: View Transformation
- P: Projection Transformation

其中MV是一起的，又统称为ModelView Transformation。

## Camera / View Transformation

在做投影之前，需要先把camera和model变换到标准位置上，方便后续计算。

首先做camera transformation(view transformation): 把相机变换到坐标系原点，相机y轴为坐标系y轴方向，z轴为坐标系z轴方向(左手坐标系下，如果是右手坐标系下，相机z轴为坐标系z轴负方向)。下面的计算基于右手坐标系。

假设相机在$\vec {e}$位置，朝向为$\vec {g}$，上方向为$\vec {t}$，则显然左方向为$\vec g \times \vec t$。显然应该先把相机平移到原点，平移变换为$T_{view}$，再把相机方向旋转过去，旋转变换为$R_{view}$。

对于平移，显然有：

$$T_{veiw} = \begin{pmatrix}1 & 0 & 0 & -x_{e} \\\ 0 & 1 & 0 & -y_{e} \\\ 0 & 0 & 1 & -z_{e} \\\ 0 & 0 & 0 & 1\end{pmatrix}$$

旋转则相对复杂，从正向去求不太好求，可以求反向：把标准方向旋转到相机当前的方向，求出这个变换矩阵之后，这个变换矩阵的逆矩阵就是$R_{view}$。逆矩阵怎么求？因为旋转矩阵是**正交矩阵**，所以**旋转矩阵的转置矩阵就是旋转矩阵的逆矩阵**。

把这个逆变换的矩阵用$R^{-1}_{view}$表示，那么有：

$$R^{-1}_{view} = \begin{pmatrix}x_{\vec g \times \vec t} & x_{\vec t} & x_{-\vec g} & 0 \\\ y_{\vec g \times \vec t} & y_{\vec t} & y_{-\vec g} & 0 \\\ z_{\vec g \times \vec t} & z_{\vec t} & z_{-\vec g} & 0 \\\ 0 & 0 & 0 & 1\end{pmatrix}$$

求转置矩阵得到：

$$R_{view} = \begin{pmatrix}x_{\vec g \times \vec t} & y_{\vec g \times \vec t} & z_{\vec g \times \vec t} & 0 \\\ x_{\vec t} & y_{\vec t} & z_{\vec t} & 0 \\\ x_{-\vec g} & y_{-\vec g} & z_{-\vec g} & 0 \\\ 0 & 0 & 0 & 1\end{pmatrix}$$

然后对场景内所有的model，都做一次这个view transformation，即可将场景中所有的物体都变换到这个标准位置上。最终变换矩阵为$M_{view} = R_{view} \times T_{view}$

所以该过程成为ModelView Transformation。

## Orthographic Projection

正交投影的概念比较简单，就是想象成相机无限远，近平面和原平面大小相同。

当ModelView Transformation完成后，其实把Model的z轴坐标丢掉，投影变换就完成了。

不过为了方便，一般会把Model投影到一个$[-1, 1]^2$的矩形上。

一般是这样操作的：先定义一个立方体：有左面left / 右面 right / 下面 bottom / 上面 top / 远面 far / 近面 near，注意因为相机朝着z轴负方向，所以有$near > far$。这个立方体简写为$[l, r] \times [b, t] \times [f, n]$。

然后尝试把这个立方体变换到canonical cube$[-1, 1]^3$。所以就是先把立方体的中心平移到原点，然后xyz轴都缩放到$[-1, 1]$的范围，就完成了。

所以有：

$$M_{ortho} = \begin{pmatrix}\frac {2}{r - l} & 0 & 0 & 0 \\\ 0 & \frac {2}{t - b} & 0 & 0 \\\ 0 & 0 & \frac {2}{n - f} & 0 \\\ 0 & 0 & 0 & 1\end{pmatrix}\begin{pmatrix}1 & 0 & 0 & -\frac {r + l}{2} \\\ 0 & 1 & 0 & -\frac {t + b}{2} \\\ 0 & 0 & 1 & -\frac {n + f}{2} \\\ 0 & 0 & 0 & 1\end{pmatrix}$$

## Perspective Projection

透视投影会达到近大远小的效果，有近平面和远平面，如本文第一幅图所示呈一个四棱锥状。

![how to do perspective projectsion](../Images/How_to_do_perspective_projection.png)

上面这个图讲的很清楚了，只需要把近平面和远平面中间的Frustum变换为Cuboid，然后对Cuboid做正交投影即可。

问题就是如何把Frustum变换为Cuboid。

![similar triangle](../Images/Perspective_projection_similar_triangle.png)

根据上面的这个相似三角形，我们可以知道，我们需要对向量做这种变换：

$$\begin{pmatrix}x \\\ y \\\ z \\\ 1\end{pmatrix} => \begin{pmatrix}\frac {nx}{z} \\\ \frac {ny}{z} \\\ ? \\\ 1\end{pmatrix}$$

而根据其次坐标的定义，我们知道：

$$\begin{pmatrix}x \\\ y \\\ z \\\ 1\end{pmatrix} => \begin{pmatrix}\frac {nx}{z} \\\ \frac {ny}{z} \\\ ? \\\ 1\end{pmatrix} = \begin{pmatrix}nx \\\ ny \\\ ? \\\ z\end{pmatrix}$$

所以我们是要求一个这样的矩阵：


$$\begin{pmatrix}? & ? & ? & ? \\\ ? & ? & ? & ? \\\ ? & ? & ? & ? \\\ ? & ? & ? & ?\end{pmatrix} \begin{pmatrix}x \\\ y \\\ z \\\ 1\end{pmatrix} = \begin{pmatrix}nx \\\ ny \\\ ? \\\ z\end{pmatrix}$$

显然有3行已经清楚了：

$$\begin{pmatrix}n & 0 & 0 & 0 \\\ 0 & n & 0 & 0 \\\ ? & ? & ? & ? \\\ 0 & 0 & 0 & z\end{pmatrix} \begin{pmatrix}x \\\ y \\\ z \\\ 1\end{pmatrix} = \begin{pmatrix}nx \\\ ny \\\ ? \\\ z\end{pmatrix}$$

再根据2条规律：

1. 近平面上的点变换后坐标不变；
2. 远平面上的中心点，变换后坐标不变；

根据这2条规律，得到下面2个变换结果(注意需要和上面一样，其次左边全部乘z，n为近平面的z轴坐标，f为远平面的z轴坐标)：

$$\begin{pmatrix}x \\\ y \\\ n \\\ 1\end{pmatrix} => \begin{pmatrix}nx \\\ ny \\\ n^2 \\\ n\end{pmatrix}$$

$$\begin{pmatrix}x \\\ y \\\ f \\\ 1\end{pmatrix} => \begin{pmatrix}fx \\\ fy \\\ f^2 \\\ f\end{pmatrix}$$

最终得到：

$$M_{persp->ortho} = \begin{pmatrix}n & 0 & 0 & 0 \\\ 0 & n & 0 & 0 \\\ 0 & 0 & n+f & -nf \\\ 0 & 0 & 0 & z\end{pmatrix}$$

结合正交投影，有：

$$M_{persp} = M_{ortho} \times M_{persp->ortho}$$

### Frustum变换为Cuboid时z如何变化？

根据$M_{persp->ortho}$，很容易得到变换关系：

$$\begin{pmatrix}x \\\ y \\\ z \\\ 1\end{pmatrix} => \begin{pmatrix}\frac {nx}{z} \\\ \frac {ny}{z} \\\ n + f - \frac {nf}{z} \\\ 1\end{pmatrix}$$

设原本的z(original z)为$o(z)$，变换后的z(perspective z)为$p(z)$，显然有：

$$o(z) = z$$

$$p(z) = n + f - \frac {nf}{z}$$

这2个函数的图像很容易画出来，$o$是个一次函数，$p$是个反比例函数在y方向平移了$n + f$(注意这俩都小于0所以是向下平移)，而且有$o(n) = p(n)$和$o(f) = p(f)$，所以显然对于区间$(f, n)$($n < 0$)，有$o(z) > p(z)$。

也可以用简单的微分证明这一点，令：

$$q(z) = o(z) - p(z) = z + \frac {nf}{z} - n - f$$

求$q$的导数，有：

$$q'(z) = 1 - \frac {nf}{z^2}$$

- $q'(f) = 1 - \frac {n}{f} > 0$
- $q'(-\sqrt {nf}) = 1 - 1 = 0$
- $q'(n) = 1 - \frac {f}{n} < 0$

所以在$[f, -\sqrt {nf}]$内$q$单调递增，在$[-\sqrt {nf}, n]$内$q$单调递减。

又因为$q(f) = 0$且$q(n) = 0$，所以在区间$(f, n)$内$q(z) > 0$，故而有$o(z) - p(z) > 0$即$o(z) > p(z)$。

综上，最终的结论为: Frustum变换为Cuboid的过程中，Frustum内的点的z轴坐标是会变得更加靠近近平面的。
