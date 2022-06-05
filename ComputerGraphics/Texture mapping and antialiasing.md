Texture就是一个物体表面的uv展开，即贴图。

在进行[[Shading]]之前，通常会从Texture上取颜色，并将其颜色值传到shader中进行着色。

通过[[Barycentric Coordinates]]的知识，当我们知道三角形的顶点对应的(u, v)坐标时，我们很容易可以通过重心坐标插值的方式得到三角形内部某个位置的对应的(u, v)坐标，所以这部分代码大概是这么写的：

```
for each rasterized screen sample (x, y):  // usually a pixel's center
    (u, v) = evaluate texture coordinate at (x, y);  // user barycentric cooridinates
    colortexture = texture.sample(u, v);
    set samples's color to textrue color;  // usually the diffuse albedo Kd
```

但因为这本质上是个采样，所以肯定会遇到[[Rasterization and antialiasing and Z-Buffering]]里面遇到的问题，当贴图过小或者贴图过大时都会出现问题。

## Texture magnification

当贴图过小需要将贴图放大时，比较好处理，常用的方法有[Bilinear interpolation](https://en.wikipedia.org/wiki/Bilinear_interpolation) 和 [Bicubic interpolation](https://en.wikipedia.org/wiki/Bicubic_interpolation)。

以Bilinear interpolation双线性插值为例：

![Bilinear interpolation](../Images/Bilinear_interpolation.png)

原理很简单，就是sample一个点的颜色时，取其最近的4个像素点，在水平方向上做2次线性插值，然后根据水平方向上线性插值的结果求竖直方向上的线性插值，根据最终的插值结果作为取样颜色结果。

当然也可以先求竖直方向上的插值再求水平方向上的插值，最终的结果是一样的。

## Texture antialiasing

当贴图过大时，由于屏幕上的像素大小不会发生变化，即采样频率不变，此时就会出现采样频率跟不上原始信号变化频率的问题，出现aliase走样现象。

![texture range query](../Images/Texture_range_query.png)

如上图所示，比如同一张贴图应用在一个超大的物体上，屏幕上同样大小的像素(灰蓝色区域)，在近处只是贴图上的一个很小的区域，但到了远处就变成了贴图上的一个很大的区域，所以出现了走样的现象。

虽然仍然可以通过超采样来解决问题，但是开销比较大。

目前图形学上常采用[mipmap](https://en.wikipedia.org/wiki/Mipmap)来完成一个近似的、方形的区域采样。

mipmap就是在渲染之前，先对所有的texture做预处理，将边长为$\frac {1}{2}$, $\frac {1}{4}$, $\frac {1}{8}$, $\frac {1}{16}$...的一系列图片预先生成，并存在显存中(等比数列求和一下其实只额外增加了$\frac {1}{3}$的显存空间)：

![Mipmap](../Images/Mipmap.png)

其中原始图片为第0层，记作D0，边长为$\frac {1}{2}$的图片为第1层，记作D1，以此类推。

当获取某个屏幕上pixel对应的texel的颜色时，首先计算这个pixel点在texture上对应的区域的边长大小，然后该边长大小求以2为底的对数，求出该区域应当在Mipmap里的第几层。计算过程如下图所示，实际上是一个近似计算，通过相邻像素点映射到texture上的(u, v)坐标，近似算出区域大小：

![Computing mipmap level](../Images/Computing_mipmap_level.png)

此时算出来的D不是整数，不过没关系，可以通过三次插值的方式，等到一个精确的值：

1. 在`math.floor(D)`层上，通过bilinear interpolation，求出该层的颜色插值
2. 在`math.ceil(D)`层上，通过bilinear interpolation，求出该层的颜色插值
3. 根据D的实际值和1、2的结果，再次做线性插值，得到一个精确的值

这个过程如下图所示：

![Trilinear interpolation](../Images/Trilinear_interpolation.png)

不过这种算法仍然会有点问题，比如下面这种情况：

![Irregular pixel footprint in texture](../Images/Irregular_pixel_footprint_in_texture.png)

当屏幕上的像素点对应的是贴图上的矩形区域，而非方形区域时，就会因为我们用了一个过大的区域来插值，而出现了overblur过渡模糊。

此时通过[Anisotropic filtering](https://en.wikipedia.org/wiki/Anisotropic_filtering)各向异性过滤可以缓解这个问题，该方案就是在生成Mipmap的同时，还生成了一些$[1, \frac {1}{2}]$, $[1, \frac {1}{4}]$, $[\frac {1}{2}, \frac {1}{4}]$..等等一系列图(如下图所示)，此时需要的额外显存大小就变成了原来的3倍，而非$\frac {1}{3}$。有了这些新的图之后，就可以对上图这些竖直或水平的矩形做比较好的处理，但是对于倾斜的矩形仍然没有办法处理。

![MipMap_Example_STS101_Anisotropic](../Images/MipMap_Example_STS101_Anisotropic.png)
