opengl的shader名为glsl(OpenGL Shading Language)，一个shader通常有以下结构：

```glsl
#version version_number
in type in_variable_name;
in type in_variable_name;

out type out_variable_name;
  
uniform type uniform_name;
  
void main()
{
  // process input(s) and do some weird graphics stuff
  ...
  // output processed stuff to output variable
  out_variable_name = weird_stuff_we_processed;
}
```

其中vertex shader用来给顶点着色，可以传递多个顶点属性，不过顶点属性的个数和大小是有限制的，一般来说顶点至少支持16个顶点属性，每个顶点属性做多有4个分量（比如vector4就是4分量的顶点属性）。可以通过下面的代码拿到当前OS的顶点属性上限：

```cpp
int nrAttributes;
glGetIntegerv(GL_MAX_VERTEX_ATTRIBS, &nrAttributes);
std::cout << "Maximum nr of vertex attributes supported: " << nrAttributes << std::endl;
```

## 类型

支持以下类型：

- int
- float
- double
- uint
- bool
- vector
- matrix

vector又有多种(n = 2, 3, 4)：

- vecn: 有n个float的vector
- bvecn: 有n个bool的vector
- ivecn: 有n个int的vector
- uvecn: 有n个unsign int的vector
- dvecn: 有n个double的vector

vector的构造方法有很多种：

```glsl
vec2 someVec;
vec4 differentVec = someVec.xyxx;
vec3 anotherVec = differentVec.zyw;
vec4 otherVec = someVec.xxxx + anotherVec.yxzy;

vec2 vect = vec2(0.5, 0.7);
vec4 result = vec4(vect, 0.0, 0.0);
vec4 otherResult = vec4(result.xyz, 1.0);
```

matrix的类型类似vector，把vec改成mat即可。

## 输入和输出

vertex shader至少需要有gl_Position作为输出类型为vec4，fragment shader至少需要有FragColor作为输出类型为vec4。

vertex shader的out可以用作fragment shader的in，需要名称相同、类型相同。

vertex shader的输入为顶点属性，由C++写的CPU程序指定，在[[VBO EBO VAO#通过VBO链接顶点属性]]中有提及，下面为设置2种顶点属性的代码示例：

```cpp
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), nullptr);  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
glEnableVertexAttribArray(0);
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), reinterpret_cast<void*>(3 * sizeof(float)));  // NOLINT(cppcoreguidelines-avoid-magic-numbers, cppcoreguidelines-pro-type-reinterpret-cast)
glEnableVertexAttribArray(1);
```

## uniform

uniform是一个shader program的全局变量，除非被C++或者shader更新，否则它的值在一个shader program中的所有shader中都是一样的。

shader中更新uniform时直接对uniform赋值即可，C++中通过如下办法更新uniform:

```cpp
int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
if (vertexColorLoaction != -1) {
    glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
}
```

首先要通过`glGetUniformLocation`找到显存中uniform的位置，如果没找到会返回-1。

然后要调`glUseProgram`，因为OpenGL实在当前激活的shader program中设置uniform的。

最后再调`glUniformnx`，n为数字比如1, 2, 3, 4，x为类型后缀(因为OpenGL是一个C库所以不支持类型重载)，后缀意义：

- f: float
- i: int
- ui: unsigned int
- fv: float vector

## 常用内建函数

- 最大值：`max(a, b)`
- 乘方：`pow(a, 2.0)`
- 点乘：`dot(a, b)`
- 叉乘：`cross(a, b)`
- 标准向量：`normalize(a)`
- 向量长度：`length(someVector)`

## shader struct

glsl也可以定义结构体：

```glsl
#version 400 core

struct Material {
    vec3 ambient;
    vec3 diffuse;
    vec3 specular;
    float shininess;
}; 

uniform Material material;
```

结构体类似于一个命名空间，可以通过下面的方式给结构体中的每一个分量赋值：

```cpp
lightingShader.setVec3("material.ambient",  1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.diffuse",  1.0f, 0.5f, 0.31f);
lightingShader.setVec3("material.specular", 0.5f, 0.5f, 0.5f);
lightingShader.setFloat("material.shininess", 32.0f);
```

## macro

glsl也可以定义宏：

```glsl
#define NR_POINT_LIGHTS 4
```

## array

glsl也可以定义数组：

```glsl
uniform PointLight pointLights[NR_POINT_LIGHTS];
```

可以通过下面的方式给数组赋值：

```cpp
lightingShader.setVec3("pointLights[0].position", pointLightPositions[0]);
lightingShader.setVec3("pointLights[0].ambient", 0.05f, 0.05f, 0.05f);
lightingShader.setVec3("pointLights[0].diffuse", 0.8f, 0.8f, 0.8f);
lightingShader.setVec3("pointLights[0].specular", 1.0f, 1.0f, 1.0f);
lightingShader.setFloat("pointLights[0].constant", 1.0f);
lightingShader.setFloat("pointLights[0].linear", 0.09f);
lightingShader.setFloat("pointLights[0].quadratic", 0.032f);
```

# function

glsl也可以定义函数，函数被调用前必须被声明，可以声明整个函数也可以只声明函数原型。

声明函数原型：

```glsl
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir);
```

定义函数：

```glsl
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir)
{
    vec3 lightDir = normalize(-light.direction);
    // diffuse shading
    float diff = max(dot(normal, lightDir), 0.0);
    // specular shading
    vec3 reflectDir = reflect(-lightDir, normal);
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    // combine results
    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
    return (ambient + diffuse + specular);
}
```

## Shader类封装

为了方便使用，可以将shader program封装为一个类，从文件中读取shader代码。

根据[glslang](https://github.com/KhronosGroup/glslang)，shader文件的后缀通常为：

- vertex shader: `.vert`
- fragment shader: `.frag`

头文件：

```cpp
#pragma once

#include <glad/glad.h>

#include <fstream>

#include <iostream>
#include <sstream>
#include <string>

class Shader {
public:
    unsigned int id;

    Shader(const char* vertexPath, const char* fragmentPath);
    ~Shader();
    Shader(Shader const&) = delete;
    Shader(Shader&&) = delete;
    Shader& operator=(Shader const&) = delete;
    Shader& operator=(Shader&&) = delete;

    void use();
    void setBool(const std::string& name, bool value) const;
    void setInt(const std::string& name, int value) const;
    void setFloat(const std::string& name, float value) const;
    void setVec3(const std::string& name, const glm::vec3& value) const;
    void setVec3(const std::string& name, float x, float y, float z) const;
    void setMatrix(const std::string& name, const GLfloat* value) const;
};
```

实现文件：

```cpp
#include "shader.hpp"

Shader::Shader(const char* vertexPath, const char* fragmentPath) {
    // 1. retrieve the vertex/fragment source code from filePath
    std::string vertexCode;
    std::string fragmentCode;
    std::ifstream vShaderFile;
    std::ifstream fShaderFile;
    // ensure ifstream objects can throw exceptions
    vShaderFile.exceptions(std::ifstream::failbit | std::ifstream::badbit);
    fShaderFile.exceptions(std::ifstream::failbit | std::ifstream::badbit);
    try {
        // open files
        vShaderFile.open(vertexPath);
        fShaderFile.open(fragmentPath);
        std::stringstream vShaderStream, fShaderStream;
        // read file's buffer contents into streams
        vShaderStream << vShaderFile.rdbuf();
        fShaderStream << fShaderFile.rdbuf();
        // close file handlers
        vShaderFile.close();
        fShaderFile.close();
        // convert stream into string
        vertexCode = vShaderStream.str();
        fragmentCode = fShaderStream.str();
    } catch (std::ifstream::failure& e) {
        std::cerr << "ERROR::SHADER::FILE_NOT_SUCCESFULLY_READ" << std::endl;
    }
    const char* vShaderCode = vertexCode.c_str();
    const char* fShaderCode = fragmentCode.c_str();

    // 2. compile shaders
    unsigned int vertex, fragment;
    const int LOG_SIZE = 512;
    int success;
    char infoLog[LOG_SIZE];  // NOLINT(cppcoreguidelines-avoid-c-arrays)

    // vertex shader
    vertex = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vertex, 1, &vShaderCode, nullptr);
    glCompileShader(vertex);
    // get compile errors
    glGetShaderiv(vertex, GL_COMPILE_STATUS, &success);
    if (!success) {
        glGetShaderInfoLog(vertex, LOG_SIZE, nullptr, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
        std::cerr << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n"
                  << infoLog << std::endl;  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
    }

    // fragment shader
    fragment = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fragment, 1, &fShaderCode, nullptr);
    glCompileShader(fragment);
    // get compile errors
    glGetShaderiv(fragment, GL_COMPILE_STATUS, &success);
    if (!success) {
        glGetShaderInfoLog(fragment, LOG_SIZE, nullptr, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
        std::cerr << "ERROR::SHADER::FRAGMENT::COMPILATION_FAILED\n"
                  << infoLog << std::endl;  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
    }

    // shader program
    id = glCreateProgram();
    glAttachShader(id, vertex);
    glAttachShader(id, fragment);
    glLinkProgram(id);
    // get link errors
    glGetProgramiv(id, GL_LINK_STATUS, &success);
    if (!success) {
        glGetProgramInfoLog(id, LOG_SIZE, nullptr, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
        std::cerr << "ERROR::SHADER::PROGRAM::LINK_FAILED\n"
                  << infoLog << std::endl;  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
    }

    // delete the shaders as they're linked into our program now and no longer necessary
    glDeleteShader(vertex);
    glDeleteShader(fragment);
}

Shader::~Shader() {
    glDeleteProgram(id);
}

void Shader::use() {
    glUseProgram(id);
}

void Shader::setBool(const std::string& name, bool value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniform1i(location, static_cast<int>(value));
}

void Shader::setInt(const std::string& name, int value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniform1i(location, value);
}

void Shader::setFloat(const std::string& name, float value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniform1f(location, value);
}

void Shader::setVec3(const std::string& name, const glm::vec3& value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniform3fv(location, 1, &value[0]);
}

void Shader::setVec3(const std::string& name, float x, float y, float z) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniform3f(location, x, y, z);
}

void Shader::setMatrix(const std::string& name, const GLfloat* value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniformMatrix4fv(location, 1, GL_FALSE, value);
}
```
