![Whitted-Style Ray Tracing](../Images/Whitted-Style_ray_tracing.png)

从摄像机点，穿过当前要绘制的像素点，连出一条光线primary ray，如果光线和物体表面有交点（计算方法参见[[Ray-Surface Intersection]][光线和曲面的交点](Ray-Surface%20Intersection.md)），则从该交点到光源方向连一条shadow ray，如果shadow ray没有碰到其他物体，说明该点会被光源照亮，否则在阴影里。如果该点可以被光源照亮，则计算该点的着色，画在像素上。

primary ray和物体的交点，可能存在反射或者折射的光线secondary ray，对于secondary ray，递归地做primary ray的计算，如果通过secondary ray也能计算出新的着色，则用secondary ray的着色加上primary ray的着色。

像素的最终着色等于primary ray的着色加上所有secondary ray的着色。

需要注意，在Whitted-Style Ray Tracing中，当光线打到Specular点上才会反射或折射，一旦达到漫反射点上，光线就会停止。
