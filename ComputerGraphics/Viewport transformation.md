[[Orthographic and Perspective Projection]]完成之后的下一步就是Viewport transformation，将3D对象映射到视口。

![viewport transformation definition](../Images/Viewport_transformation_definition.png)

如上图所示，为了完成viewport transformation，首先要定义2个变量：

1. viewport的宽高比aspect
2. Field of View, FOV

图中的FOV使用的是fovY，有时也会用fovX，不过只要知道了宽高比，通过其中一个方向的fov必然能算出来另一个方向的fov。

有了fovY和aspect，很容易算出[[Orthographic and Perspective Projection]]中的l, r, b, t(l和r互为相反数，b和t互为相反数):

$$tan \frac {fovY}{2} = \frac {t}{|n|}$$

$$aspect = \frac {r}{t}$$

viewport transformation就是把$[-1, 1]^3$这个立方体映射到viewport空间，如果丢弃z的话，很容易得到viewport transformation的矩阵：

$$M_{viewport} = \begin{pmatrix} \frac {width}{2} & 0 & 0 & \frac {width}{2} \\\ 0 & \frac {height}{2} & 0 & \frac {height}{2} \\\ 0 & 0 & 1 & 0 \\\ 0 & 0 & 0 & 1 \end{pmatrix}$$
