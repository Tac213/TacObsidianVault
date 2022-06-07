[原文链接](https://learnopengl.com/Getting-started/Hello-Triangle)

[代码链接](https://learnopengl.com/code_viewer_gh.php?code=src/1.getting_started/2.2.hello_triangle_indexed/hello_triangle_indexed.cpp)

## compile shaders

opengl没有默认的vertex shader和fragment shader，需要开发者自行提供，以下为shader初始化与编译方法：

```cpp
const LOG_SIZE = 512;
const char* vertexShaderSource = "#version 330 core\n"
                                 "layout (location = 0) in vec3 aPos;\n"
                                 "void main()\n"
                                 "{\n"
                                 "   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
                                 "}\0";
const char* fragmentShaderSource = "#version 330 core\n"
                                   "out vec4 FragColor;\n"
                                   "void main()\n"
                                   "{\n"
                                   "   FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0);\n"
                                   "}\0";

bool initializeShader(unsigned int& shader, GLenum shaderType, const char* shaderSource, char* infoLog) {
    shader = glCreateShader(shaderType);
    glShaderSource(shader, 1, &shaderSource, nullptr);
    glCompileShader(shader);
    int success;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &success);
    if (!success) {
        glGetShaderInfoLog(shader, LOG_SIZE, nullptr, infoLog);
        std::cerr << "ERROR::SHADER::" << (shaderType == GL_VERTEX_SHADER ? "VERTEX" : "FRAGMENT") << "::COMPILATION_FAILED\n"
                  << infoLog << std::endl;  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
        return false;
    }
    return true;
}

char infoLog[LOG_SIZE];  // NOLINT(cppcoreguidelines-avoid-c-arrays)
// build and compile our shader program
// ------------------------------------
// vertex shader
unsigned int vertexShader;
initializeShader(vertexShader, GL_VERTEX_SHADER, vertexShaderSource, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
// fragment shader
unsigned int fragmentShader;
initializeShader(fragmentShader, GL_FRAGMENT_SHADER, fragmentShaderSource, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
```

## link shaders

shader编译完了之后还要链接到程序里，使用的是shader program的概念，与编译shader类似，同样可以检测链接是否成功：

```cpp
int success;
// link shaders
unsigned int shaderProgram;
shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);
glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
if (!success) {
    glGetProgramInfoLog(shaderProgram, LOG_SIZE, nullptr, infoLog);  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
    std::cerr << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n"
              << infoLog << std::endl;  // NOLINT(cppcoreguidelines-pro-bounds-array-to-pointer-decay)
}
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);
```

注意链接完成后应当把shader删除，无需再次使用。

为了使shader生效，需要在render frame使用shader program:

```cpp
while (!glfwWindowShouldClose(window)) {
    glUseProgram(shaderProgram);
}
glDeleteProgram(shaderProgram);
```

注意退出render frame之后应当将shader program删除。
