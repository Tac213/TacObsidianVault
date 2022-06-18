进行光线追踪时，很重要的一步是要做[[Ray-Surface Intersection]][计算光线和三角形的交点](Ray-Surface%20Intersection.md)，对于每一条光线，不可能每次都遍历场景内的所有三角形来判断光线是否和三角形相交，此时需要在场景的数据结构上进行处理，对场景做划分，达到加速的目的。

## Uniform spatial partitions

如下图所示，可以把空间划分成均等的格子：

![Uniform spatial partition](../Images/Uniform_spatial_partition.png)

划分完成后记录哪些格子是有跟物体相交的，光线进入场景时，根据光线的方向得到candidate gird，使用[[Ray-Surface Intersection#Ray intersection with Axis-Aligned Box]]中描述的方法判断光线是否与格子相交，如果相交了再判断格子是否有跟物体相交，最终得到光线和物体的交点。

使用这种方法，格子的数量不能太多也不能太少，根据人们使用的经验，通常格子的数量是场景中物体数量的27倍时，效果是比较好的。

而且这种方法只对场景中物体分布比较均匀的场景试用，如果分布不均匀时则不适用。

## Oct-Tree

Oct-Tree即[八叉树](https://en.wikipedia.org/wiki/Octree)，将一个3D的场景，在x轴方向中点切一次，y轴方向中点切一次，z轴方向中点切一次得到8个子节点，每8个子节点重复这个过程，就可以不断地将场景划分。

![Octree](../Images/Octree.png)

这种方法划分的场景由于每个节点的子节点个数过多，也不是特别常用。

## KD-Tree

KD-Tree即[k-dimensional tree](https://en.wikipedia.org/wiki/K-d_tree)，是一种空间二叉树，可以解决八叉树每个节点的子节点个数较多的问题。

![KD-Tree](../Images/KD-Tree.png)

如上图所示，KD-Tree就是给空间做一个二叉的划分，而且每次划分时，为了尽可能让空间均匀地划分(不出现长条地划分结果)，每次现在x方向上划分，再在y方向上划分，再在z方向上划分。进在叶子节点存储空间的物体信息，中间节点只存子节点的指针，当然所有节点都要存其所占据的空间信息。

当光线穿过场景时，就可以根据这个二叉树的结构，如果光线跟节点相交，则说明光线可能根该节点的某个叶子节点所在的空间的某个物体相交，因此可以对子节点递归地调用这个相交地过程，直到找到相交的物体和其交点为止。

## BVH

BVH即[Bounding Volume Hierarchy](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy)，为了解决KD-Tree所存在的问题：

1. 物体可能会被切割，本分散到2个不同的节点里
2. KD-Tree的建立不好写，triangle和box的求交很容易写错，比如一个triangle可能3个点都在box外，但这3个点正好围着这个box

BVH实际上时一种object partiions，是基于物体对场景做划分：

![BVH](../Images/Bounding_volume_hierarchy.png)

每次的分割，是将当前空间内的所有物体，切分成2个不同的子集，然后重新计算每个子集所在的空间(很好算，遍历所有物体求xyz的最小值和最大值即可)。

分割时要注意2点：

1. 为了均等划分空间，每次都选最长的轴向进行分割
2. 在这个轴向上，根据重心坐标的轴向坐标，比如选x轴划分就取重心坐标的x分量，使用[快速选择](https://zhuanlan.zhihu.com/p/64627590)算法，将物体集合从中间切开，保证树的平衡，避免树太深

通常节点中的物体个数足够小，比如5个时，就可以停止分割了。

使用BVH求光线和物体交点的伪代码：

```
Intersect(Ray ray, BVH node) {
    if (ray misses node.bbox) return;
    if (node is a leaf node) {
        test intersection with all objs;
        return closest intersection;
    }
    hit1 = Intersect(ray, node.child1);
    hit2 = Intersect(ray, node.child2);
    return the closer of hit1, hit2;
}
```
