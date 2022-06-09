常见的透光物有3种：

- Directional Light: 平行光源
- Point Light: 点光源
- Spotlight: 聚光

## Directional Light

![Directional Light](https://learnopengl.com/img/lighting/light_casters_directional.png)

平行光源是最简单的光源，类似太阳光一样，光源的位置都不需要定义，只需要定义光的方向即可，也不需要考虑光强衰减的问题。

## Point Light

![Point Light](https://learnopengl.com/img/lighting/light_casters_point.png)

点光源是一个是一个带有位置的光源，而且光照强度会随着距离的增加而衰减。

衰减值通常是这么定义的：

$$F_{att} = \frac {1.0}{K_c + K_l * d + K_q * d^2}$$

其中$d$表示shading point距离光源的距离，还有3个可以配置的值：常数项$K_c$、一次项$K_l$和二次项$K_q$。

衰减图像如下：

![light attenuation](https://learnopengl.com/img/lighting/attenuation.png)

如何给这3个配置项取值就决定了不同的点光源性质。

[Ogre3D's Wiki](https://wiki.ogre3d.org/tiki-index.php?page=-Point+Light+Attenuation)给出了下面这个表，Distance可以理解为点光源可以照到的最远距离，对于这个性能的点光源，各参数的取值如下表：

Distance | Constant | Linear | Quadratic
--- | --- | --- | ---
7 | 1.0 | 0.7 | 1.8
13 | 1.0 | 0.35 | 0.44
20 | 1.0 | 0.22 | 0.20
32 | 1.0 | 0.14 | 0.07
50 | 1.0 | 0.09 | 0.032
65 | 1.0 | 0.07 | 0.017
100 | 1.0 | 0.045 | 0.0075
160 | 1.0 | 0.027 | 0.0028
200 | 1.0 | 0.022 | 0.0019
325 | 1.0 | 0.014 | 0.0007
600 | 1.0 | 0.007 | 0.0002
3250 | 1.0 | 0.0014 | 0.000007

## Spotlight

![Spotlight](https://learnopengl.com/img/lighting/light_casters_spotlight_angles.png)

Spotlight的例子比如路灯、手电筒。上面这个图给出了Spotlight几个重要的值的定义：

- `LightDir`: 从shading point指向光源的向量
- `SpotDir`: Spotlight的方向
- $\phi$: 指定了聚光半径的切光角，落在这个角度之外的物体都不会被这个聚光所照亮
- $\theta$: `LightDir`和`SpotDir`之间的夹角

需要注意，为了计算方便，通常代码里都是用$cos(\phi)$来代替$\phi$，用$cos(\theta)$代替$\theta$。

判断方法也很简单，只要$cos(\theta) \gt cos(\phi)$就认为shading point可以被照亮。

但是这个算法会导致下面这个效果，光照有个硬边，看起来很假：

![try spotlight](https://learnopengl-cn.github.io/img/02/05/light_casters_spotlight_hard.png)

为了让边缘更平滑，可以增加一个一个光照必定为0的角度$\gamma$，最终计算的光照强度在$\phi$和$\gamma$之间做个插值即可，公式很简单：

$$I = \frac {cos(\theta) - cos(\gamma)}{cos(\phi) - cos(\gamma)}$$

计算时为了保证稳妥可以把$I$ clamp在$[0, 1]$这个区间上。
