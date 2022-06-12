## Shadow Mapping

Shadow Mapping是比较常用的一种技术，他是根据这样一个原理来画阴影的：不在阴影里面的点，一定能同时被相机和光源看到。

所以Shadow Mapping其实就是做2次Rasterization，第一次从光源处做，但不渲染结果，只拿[[Rasterization and antialiasing and Z-Buffering#Z-Buffering]]中的深度图数据，第二次再从相机处做，用这一次的深度图数据和上一次的深度图数据做比较，如果深度相等，则说明都能看到，不是阴影，否则就是阴影。

但是要比较浮点数是否相等是很困难的事情，浮点数基本上很难完全相等，经常要在做浮点数比较时做scale / bias / tolerance等处理，但实际上这些方法很难从根本上解决问题。

而且Shadow Mapping从原理上讲只能做[[Light casters#Point Light]]类型的光源的阴影。

进一步的，Shadow Mapping产生的阴影很硬，因为计算的结果非0则1，所以阴影存在很明显的边界。为了解决这个问题，必须给光源增加大小的概念，从光源的2个地方做Shadow Mapping，然后对结果做插值，得到软阴影。
