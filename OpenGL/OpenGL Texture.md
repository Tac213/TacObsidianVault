## Texture创建与绑定

和其他OpenGL对象相似，创建Texture对象后需要将其绑定到context上才能对Texture进行配置：

```cpp
unsigned int texture;
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
```

## 配置Texture

### Texture Wrapping

Texture Wrapping有4种方式：

- GL_REPEAT: 对纹理的默认行为。重复纹理图像。
- GL_MIRRORED_REPEAT: 和GL_REPEAT一样，但每次重复图片是镜像放置的。
- GL_CLAMP_TO_EDGE: 纹理坐标会被约束在0到1之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果。
- GL_CLAMP_TO_BORDER: 超出的坐标为用户指定的边缘颜色。

![Texture wrapping](https://learnopengl.com/img/getting-started/texture_wrapping.png)

如果使用GL_CLAMP_TO_DORDER，要像下面这样设置边缘颜色：

```cpp
float borderColor[] = { 1.0f, 1.0f, 0.0f, 1.0f };
glTexParameterfv(GL_TEXTURE_2D, GL_TEXTURE_BORDER_COLOR, borderColor);
```

### Texture Filtering

[[Texture mapping and antialiasing]]中提到的过滤方法在OpenGL中都有实现，其中纹理放大有2种方式：

- GL_NEAREST: 直接根据采样点所在的texel取颜色，造成方格效果
- GL_LINEAR: 通过bilinear方式做线性插值

![texture magnifacation filter](https://learnopengl.com/img/getting-started/texture_filtering.png)

纹理放大则通过mipmap，mipmap需要手动创建，通过下面的方式可以创建mipmap(在绑定纹理数据后调用`glGenerateMipmap`):

```cpp
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
glGenerateMipmap(GL_TEXTURE_2D);
```

mipmap的采样也提供了4种方式：

-   GL_NEAREST_MIPMAP_NEAREST: 使用最邻近的多级渐远纹理来匹配像素大小，并使用邻近插值进行纹理采样
-   GL_LINEAR_MIPMAP_NEAREST: 使用最邻近的多级渐远纹理级别，并使用线性插值进行采样
-   GL_NEAREST_MIPMAP_LINEAR: 在两个最匹配像素大小的多级渐远纹理之间进行线性插值，使用邻近插值进行采样
-   GL_LINEAR_MIPMAP_LINEAR: 在两个邻近的多级渐远纹理之间使用线性插值，并使用线性插值进行采样

## 加载Texture

加载纹理可以使用[stb_image.h](https://raw.github.com/nothings/stb/master/stb_image.h)，用法很简单：

```cpp
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

int width, height, nrChannels;
unsigned char *data = stbi_load("container.jpg", &width, &height, &nrChannels, 0);
```

## 生成Texture

创建并绑定纹理后，就可以通过下面这个函数生成纹理数据：

```cpp
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, data);
glGenerateMipmap(GL_TEXTURE_2D);
```

-   第一个参数指定了纹理目标(Target)。设置为GL_TEXTURE_2D意味着会生成与当前绑定的纹理对象在同一个目标上的纹理（任何绑定到GL_TEXTURE_1D和GL_TEXTURE_3D的纹理不会受到影响）。
-   第二个参数为纹理指定多级渐远纹理的级别，如果你希望单独手动设置每个多级渐远纹理的级别的话。这里我们填0，也就是基本级别。
-   第三个参数告诉OpenGL我们希望把纹理储存为何种格式。我们的图像只有`RGB`值，因此我们也把纹理储存为`RGB`值。
-   第四个和第五个参数设置最终的纹理的宽度和高度。我们之前加载图像的时候储存了它们，所以我们使用对应的变量。
-   下个参数应该总是被设为`0`（历史遗留的问题）。
-   第七第八个参数定义了源图的格式和数据类型。我们使用RGB值加载这个图像，并把它们储存为`char`(byte)数组，我们将会传入对应值。
-   最后一个参数是真正的图像数据。

## 使用纹理数据

首先应该把(u, v)坐标传到vertex shader里面：

![vertex format](https://learnopengl.com/img/getting-started/vertex_attribute_pointer_interleaved_textures.png)

```cpp
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(6 * sizeof(float)));
glEnableVertexAttribArray(2);  
```

然后在vertex shader把纹理坐标直接传给fragment shader:

```glsl
#version 400 core

layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aTexCoord;

out vec3 ourColor;
out vec2 TexCoord;

void main()
{
    gl_Position = vec4(aPos, 1.0);
    ourColor = aColor;
    TexCoord = aTexCoord;
}
```

fragment shader读取纹理颜色：

```glsl
#version 400 core

out vec4 FragColor;
  
in vec3 ourColor;
in vec2 TexCoord;

uniform sampler2D ourTexture;

void main()
{
    FragColor = texture(ourTexture, TexCoord);
}
```

也可以跟别的颜色融合：

```glsl
FragColor = texture(ourTexture, TexCoord) * vec4(ourColor, 1.0);  
```

可以看到，纹理数据会被直接放到一个uniform上面。

当然在render frame，cpu在bind vao之前也要重新bind纹理：

```cpp
glBindTexture(GL_TEXTURE_2D, texture);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

## Texture Units

一个物体还可以使用多个纹理(最多16个，分别对应`GL_TEXTURE0`到`GL_TEXTURE15`)。

当只有`GL_TEXTURE0`时(默认时这个)，不需要给shader program设置uniform，当texture数量多于1时就需要给每个texture是指对应的uniform，render frame时也要指定激活哪一个:

```cpp
ourShader.use(); // don't forget to activate the shader before setting uniforms!  
glUniform1i(glGetUniformLocation(ourShader.ID, "texture1"), 0); // set it manually
ourShader.setInt("texture2", 1); // or with shader class
  
while(...) 
{
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, texture1);
    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, texture2);

    glBindVertexArray(VAO);
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0); 
}
```

shader里可以通过下面的方法混合2个纹理：

```glsl
FragColor = mix(texture(texture1, TexCoord), texture(texture2, TexCoord), 0.2);
```

0.2是一个插值，表示在以texture1为起点texture2为终点的线段上，0.2所在的坐标位置，也就是80%用texture1 20%用texture2。
