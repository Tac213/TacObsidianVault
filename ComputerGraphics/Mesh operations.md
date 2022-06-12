## Mesh subdivision

通过Mesh subdivision算法提高mesh的面数，常用的算法有Loop Subdiviaion和Catmull-Clark Subdivision。

Mesh subdivision通常包括2个步骤：

- 细分原本的面
- 更新顶点位置

### Loop Subdivision

![Loop Subdivision](../Images/Loop_subdivision.png)

Loop Subdivision只能处理面全部都是三角形的Mesh。原理就是在一个三角形的每条边取中点，将原本的1个三角形细分成4个三角形。

细分后，原本的三角形顶点称为old vertices，新的三角形顶点被称为new vertices，对于这2类顶点，采用不同的顶点位置更新方式。

首先是new vertices:

![Loop Subdivision](../Images/Loop_subdivision_new_vertices.png)

取原本与这个顶点共线的old vertices: $A$和$B$，再取跟其再同一个三角形内但不共线的顶点$C$和$D$，最终的顶点位置为这4个的加权：

$$\frac 3 8 (A + B) + \frac 1 8 (C + D)$$

然后是old vertices:

![Loop Subdivision](../Images/Loop_subdivision_old_vertices.png)

首先定义2个量：

- $n$: 顶点的度，也就是说该顶点和多少个其他的顶点相连，比如上图是6
- $u$: 如果$n$为3则为$\frac 3 {16}$，否则为$\frac 3 {8n}$

则最终顶点位置为自身位置和相邻顶点位置的加权：

$$(1 - nu)Position_{origin} + u\sum {}{}Position_{neighbor}$$

### Catmull-Clark Subdivision

Catmull-Clark Subdivision可以用于更通用的mesh，mesh的每个面不一定是要给三角形。

首先要定义2个概念：

- Non-quad face: 非4边形面，通常为三角形面
- Extraodinary vertex: 奇异点，也就是度不为4的点

![Catmull-Clark Subdivision](../Images/Catmull-Clard_subdivision.png)

细分方法：

- 在每一条线上找中点作为新的顶点
- 在面内部找一个点作为新的顶点

比如上图细分后的结果如下图：

![Catmull-Clark Subdivision after](../Images/Catmull-Clard_subdivision_after.png)

细分后结果：

- 增加Non-quad face面数的Extraodinary vertex
- 所有的面都变成了quad face

细分后，Mesh原本的顶点被称为Vertex Point，面内的新点被称为Face point，边缘上的新点位Edge point。对于这3类点，分别采用不同的位置更新策略：

![Catmull-Clark Vertex Update Rules](../Images/Catmull-Clard_vertex_update_rules.png)

## Mesh Simplification

Mesh Simplification可以减少mesh的面数，常用的方法是edge collapsing。

![Error Quadirc](../Images/Mesh_simplification_edge_collapsing.png)

edge collapsing的难点在于如何更新collapse之后的顶点的位置，让简化后的mesh和原本的mesh长得差不多。常用的方法是Quadric Error Metrics二次度量误差。

![Error Quadirc](../Images/Mesh_simplification_error_quadric.png)

原理：新顶点应满足，到原三角形面的距离的平方和最小。

比如上图，把原本的4个面简化位2个面，那么新顶点应该是到原来4个面的距离的平方和最小的那个点。

整个edge collapsing的过程：

1. 计算所有的新顶点到原来的4个面的最小平方和，放到一个**最小二叉堆**里
2. 处理堆顶顶点，对其进行edge collapsing
3. 根据更新后的面的位置重复计算1所计算的值
