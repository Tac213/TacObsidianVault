- VBO: 顶点缓冲对象Vertex Buffer Object
- EBO: 元素缓冲对象Element Buffer Object 或 索引缓冲对象 Index Buffer Object, IBO
- VAO: 顶点数组对象Vertex Array Object

## Buffer Object

首先要明白数据为什么要缓存。有很多历史原因，主要就是因为**从CPU把数据发送到显卡相对较慢**，所以只要可能我们都要尝试尽量一次性发送尽可能多的数据。当数据发送至显卡的内存中后，顶点着色器几乎能立即访问顶点，这是个非常快的过程。

Buffer Object的意义就是一个内存管理对象，在渲染过程中一次性发送大量顶点数据到显卡，避免每个顶点发送一次顶点数据。

## VBO

通过图形学的知识我们知道，每个三角形的顶点可以有多个不同的顶点数据(vertices)，比如顶点的位置、顶点的颜色等等。在OpenGL中可以自行定义每个三角形顶点的顶点数据，并通过VBO来管理这些顶点数据(会被储存到显存中)。

VBO通过如下方法创建：

```cpp
unsigned int vbo;
glGenBuffers(1, &vbo);  // 因为可以传数组来创建buffer, 所以第一个参数传的就是数组的大小
glBindBuffer(GL_ARRAY_BUFFER, vbo);
```

VBO通过如下方法删除(render loop结束之后要删除)：

```cpp
glDeleteBuffers(1, &vbo);
```

当有了顶点数据之后，就可以将顶点数据放到vbo中储存：

```cpp
std::array<float, 12> vertices = {                      // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  0.5f, 0.5f, 0.0f,     // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  0.5f, -0.5f, 0.0f,    // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  -0.5f, 0.5f, 0.0f,    // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  -0.5f, -0.5f, 0.0f};  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), &vertices, GL_STATIC_DRAW);
```

`glBufferData`是一个专门用来把用户定义的数据复制到当前绑定缓冲的函数。它的第一个参数是目标缓冲的类型：顶点缓冲对象当前绑定到`GL_ARRAY_BUFFER`目标上。第二个参数指定传输数据的大小(以字节为单位)；用一个简单的sizeof计算出顶点数据大小就行。第三个参数是我们希望发送的实际数据。

第四个参数指定了我们希望显卡如何管理给定的数据。它有三种形式：

- `GL_STATIC_DRAW` ：数据不会或几乎不会改变。
- `GL_DYNAMIC_DRAW`：数据会被改变很多。
- `GL_STREAM_DRAW` ：数据每次绘制时都会改变。

如果三角形的位置数据不会改变，每次渲染调用时都保持原样，那么它的使用类型最好是`GL_STATIC_DRAW`。如果，比如说一个缓冲中的数据将频繁被改变，那么使用的类型就是`GL_DYNAMIC_DRAW`或`GL_STREAM_DRAW`，这样就能确保显卡把数据放在能够高速写入的内存部分。

### 通过VBO链接顶点属性

![vertex buffer data](https://learnopengl.com/img/getting-started/vertex_attribute_pointer.png)

一个只有顶点坐标的顶点数据是上图所示的样式。

在[glBindBuffer](https://docs.gl/gl4/glBindBuffer)之后，通过下面的函数可以链接顶点数据到vbo：

```cpp
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0); 
```

-   第一个参数指定我们要配置的顶点属性。在顶点着色器vertex shadaer中使用`layout(location = 0)`定义了position顶点属性的Location)和这个参数一一对应，它可以把顶点属性的位置值设置为`0`。因为我们希望把数据传递到这一个顶点属性中，所以这里我们传入`0`。
-   第二个参数指定顶点属性的大小。顶点属性是一个`vec3`，它由3个值组成，所以大小是3。
-   第三个参数指定数据的类型，这里是GL_FLOAT(GLSL中`vec*`都是由浮点数值组成的)。
-   第四个参数定义我们是否希望数据被标准化(Normalize)。如果我们设置为GL_TRUE，所有数据都会被映射到0（对于有符号型signed数据是-1）到1之间。我们把它设置为GL_FALSE。
-   第五个参数叫做步长(Stride)，它告诉我们在连续的顶点属性组之间的间隔。由于下个组位置数据在3个`float`之后，我们把步长设置为`3 * sizeof(float)`。要注意的是由于我们知道这个数组是紧密排列的（在两个顶点属性之间没有空隙）我们也可以设置为0来让OpenGL决定具体步长是多少（只有当数值是紧密排列时才可用）。一旦我们有更多的顶点属性，我们就必须更小心地定义每个顶点属性之间的间隔（这个参数的意思简单说就是从这个属性第二次出现的地方到整个数组0位置之间有多少字节）。
-   最后一个参数的类型是`void*`，所以需要我们进行这个奇怪的强制类型转换。它表示位置数据在缓冲中起始位置的偏移量(Offset)。由于位置数据在数组的开头，所以这里是0。如果有offset可以写成(`reinterpret_cast<void*>(3 * sizeof(float))`)。

## VAO

有了VBO之后实际上就可以绘制物体了，但是代码很麻烦，流程如下面所示：

```cpp
// 0. 复制顶点数组到缓冲中供OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 1. 设置顶点属性指针
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// 2. 当我们渲染一个物体时要使用着色器程序
glUseProgram(shaderProgram);
// 3. 绘制物体
someOpenGLFunctionThatDrawsOurTriangle();
```

上面的代码在每个render frame都要被执行，这意味着每绘制一个物体都要把该物体的VBO配置一遍，非常麻烦，而且每次都通过buffer把数据从cpu发送到显卡，效率很低。

VAO就是为了解决这个问题而出现的，而且在OpenGL的核心模式中必须使用VAO。VAO放在显存里，通过绑定VAO可以很快地从现存中拿到VBO地配置。

一个VBO可以跟多个VAO绑定，但是一个VAO只能绑定一个VBO。

VAO和VBO的关系如下图：

![vao vbo](https://learnopengl.com/img/getting-started/vertex_array_objects.png)

VAO通过如下方法创建：

```cpp
unsigned int vao;
glGenVertexArrays(1, &vao);
```

VAO通过如下方法删除(render loop结束之后要删除)：

```cpp
glDeleteVertexArrays(1, &vao);
```

一个顶点数组对象会储存以下这些内容：

- glEnableVertexAttribArray和glDisableVertexAttribArray的调用。
- 通过glVertexAttribPointer设置的顶点属性配置。
- 通过glVertexAttribPointer调用与顶点属性关联的顶点缓冲对象。

也就是说每次调用上述接口时，都会在当前绑定的vao里存储数据。

有了vao，就可以像下面这样使用vbo:

```cpp
unsigned int vbo, vao;
glGenVertexArrays(1, &vao);
glGenBuffers(1, &vbo);
// bind the Vertex Array Object first, then bind and set vertex buffer(s), and then configure vertex attributes(s).
glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), &vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), nullptr);
glEnableVertexAttribArray(0);
// note that this is allowed, the call to glVertexAttribPointer registered VBO as the vertex attribute's bound vertex buffer object so afterwards we can safely unbind
glBindBuffer(GL_ARRAY_BUFFER, 0);
// You can unbind the VAO afterwards so other VAO calls won't accidentally modify this VAO, but this rarely happens. Modifying other
// VAOs requires a call to glBindVertexArray anyways so we generally don't unbind VAOs (nor VBOs) when it's not directly necessary.
glBindVertexArray(0);

while (!glfwWindowShouldClose(window)) {
    // draw our first triangle
    glUseProgram(shaderProgram);
    glBindVertexArray(vao);  // seeing as we only have a single VAO there's no need to bind it every time, but we'll do so to keep things a bit more organized
    glDrawArrays(GL_TRIANGLES, 0, 3);
    glBindVertexArray(0);
}
```

## EBO

一个物体通常由多个三角形构成，很多顶点都是被复用的，比如绘制一个矩形只需要4个顶点，但是如果只有VBO的话，VBO里画一个矩形就必须存储6个顶点数据(1个矩形由2个三角形构成，1个三角形有3个顶点数据)，其中有重复的顶点数据，这些重复的顶点数据是没有必要的。

EBO就是用来解决这个问题的，比如在定义矩形的4个顶点之后，额外定义各三角形所使用的顶点索引值信息，就可以不定义重复的顶点但绘制多个三角形：

```cpp
std::array<float, 12> vertices = {                      // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  0.5f, 0.5f, 0.0f,     // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  0.5f, -0.5f, 0.0f,    // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  -0.5f, 0.5f, 0.0f,    // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                  -0.5f, -0.5f, 0.0f};  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
std::array<unsigned int, 6> indices = {                 // NOLINT(cppcoreguidelines-avoid-magic-numbers)
                                       0, 1, 3,
                                       0, 2, 3};
```

由于都是Buffer Object，EBO的使用和VBO非常相似，只需要把`GL_ARRAY_BUFFER`替换为`GL_ELEMENT_ARRAY_BUFFER`即可：

```cpp
unsigned int ebo;
glGenBuffers(1, &ebo);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, ebo);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), &indices, GL_STATIC_DRAW);
```

但是要注意，在unbind VAO之前不要unbind EBO，如下图所示EBO和VAO是一一对应的，在unbind VAO之前unbind EBO意味着把VAO对应的EBO设为空。

![vao ebo](https://learnopengl.com/img/getting-started/vertex_array_objects_ebo.png)

有了EBO，在绘制三角形时就应该调用`glDrawElements`:

```cpp
while (!glfwWindowShouldClose(window)) {
    // draw our first triangle
    glUseProgram(shaderProgram);
    glBindVertexArray(vao);  // seeing as we only have a single VAO there's no need to bind it every time, but we'll do so to keep things a bit more organized
    glDrawElements(GL_TRIANGLES, sizeof(indicies), GL_UNSIGNED_INT, nullptr);
    glBindVertexArray(0);
}
```

第一个参数指定了我们绘制的模式，这个和`glDrawArrays`的一样。第二个参数是我们打算绘制顶点的个数，这里填6，也就是说我们一共需要绘制6个顶点。第三个参数是索引的类型，这里是`GL_UNSIGNED_INT`。最后一个参数里我们可以指定EBO中的偏移量（或者传递一个索引数组，但是这是当你不在使用索引缓冲对象的时候），但是我们会在这里填写0。
