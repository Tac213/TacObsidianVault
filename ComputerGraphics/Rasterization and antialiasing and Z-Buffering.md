## Rasterization

Rasterization光栅化就是把[[Viewport transformation]]中已经映射在viewport上的3D对象现实在屏幕上。

3D对象都是由若干个三角面构成的，所以本质上要解决的问题是如何在屏幕上显示一个三角形。

![draw triangle on screen](../Images/Draw_triangle_on_screen_input_and_output.png)

首先屏幕是由若干个三角形组成的，所以最简单的做法就是做一个采样：对于每个像素点的中心，判断这个点是否在三角形内部，如果在内部的话则给这个像素点着色，否则不处理该像素点。

如何判断点在三角形内部？比如三角形ABC，判断P在不在ABC内部，则分别用AC叉乘AP，CB叉乘CP，BA叉乘BP，如果三次叉乘结果同向，或者符号相同，则P在三角形内部，否则不在。

遍历像素点时，没必要每次从0到width这样遍历，可以根据输入的3个坐标点的最小xy和最大xy的值进行遍历。

按照这样采样之后，会得到这样的结果：

![sampling cause jaggies](../Images/Sampling_cause_jaggies.png)

很显然，渲染结果产生了Jaggies锯齿，即aliase走样。

## Antialiasing

为了完成反走样，首先需要明白走样是如何产生的。

这部分内容较为复杂，实际上属于**数字信号处理**这门课的知识。

数字信号处理常用[傅立叶变换](https://zh.wikipedia.org/wiki/%E5%82%85%E9%87%8C%E5%8F%B6%E5%8F%98%E6%8D%A2)对数字信号进行时域和频域之间的变换。

![fourier transform](../Images/Fourier_transform.png)

比如下面是对一个图片在时域和频域之间的变换：

![image spatial and frequency domain](../Images/Image_spatial_and_frequency_domain.png)

右边的频域图，中心是低频段，周围是高频段。

通常对于一张图片来说，边界的部分属于信号的高频段，可以理解为“信号剧烈变化”为边界，比如衣袖和背景的边界、人脸和头发的边界。

所以对于频域应用高通滤波High-pass filter，再对频域做傅立叶逆变换回到时域，会得到这样的结果：

![image apply high-pass filter](../Images/Image_apply_high_pass_filter.png)

而如果应用低通滤波Low-pass filter，会得到这样的结果，图片变模糊了：

![image apply low-pass filter](../Images/Image_apply_low_pass_filter.png)

根据[卷积定理](https://zh.wikipedia.org/wiki/%E5%8D%B7%E7%A7%AF%E5%AE%9A%E7%90%86)，函数卷积的傅立叶变换是函数傅立叶变换的乘积。

![convolution theorem](../Images/Convolution_theorem.png)

而卷积的本质是在求一个平均值。所以根据卷积定理，给图片做卷积，等效于给图片应用低通滤波，也就是使图片变模糊。

而从频域上看，采样的本质是重复频域上的内容：

![sampling = repeating sampling content](https://www.researchgate.net/profile/Haijun-He-2/publication/301556095/figure/fig5/AS:667779598389262@1536222471472/The-evolution-of-sampling-theorem-a-The-time-domain-of-the-band-limited-signal-and-b.ppm)

所以当采样频率不足时，就会产生走样(采样频率越高，频域内容的间隔越宽)：

![aliasing](../Images/aliasing.png)

而如果事先对信号饮用低通滤波，再按相同的频率采样，就可以完成反走样：

![antialiasing](../Images/antialiasing.png)

所以结论是：

> 走样产生的原因是采样频率不足。采样前如果事先做卷积(求平均)，再做采样，即可完成反走样。

假设如果对于每个像素，都能求出当前三角形在像素内所占的面积，再根据面积占比在着色时乘以该占比值，就能完成反走样。

![antialiased sampling](../Images/antialiased_sampling.png)

但要完成**求三角形占每个像素点的比例**这件事情不容易，目前比较常见的做法是MSAA(multi-sampling antialiase)，即在每个像素点中多采几次样，该像素点最终的值等于这几个采样点的平均值，通过这种方法来近似求出三角形占每个像素点的比例。

![MSAA supersampling](../Images/MSAA_supersampling.png)

除了MSAA，常见的还有FXAA(Fast Approximate Antialiase)和TAA(Temporal Antialiase)。

## Z-Buffering

Z-Buffering要解决的问题是：当多个三角形重叠时，应该如何绘制三角形？

画家算法可能可以解决这个问题：先画最远处的物体，最后画最近的物体。

但是画家算法无法解决多个三角形交织在一起，比如A叠在B上边，B叠在C上边，而C又叠在A上边。

为了解决这个问题，采用Z-Buffering算法，就是针对每个像素，只画这个像素的Z最近的三角形的像素，其余不画。

算法很简单：

```
for (each triangle T)
    for (each sample (x, y, z) in T)
        if (z < zbuffer[x, y]).      // closest sample so far
            framebufer[x, y] = rgb;  // update color
            zbuffer[x, y] = z.       // update depth
        else
            continue;                // do nothing, this sample is occluded
```
