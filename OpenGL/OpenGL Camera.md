## 数学库

OpenGL的数学库可以使用[glm](https://github.com/g-truc/glm)，直接git clone最新的tag到include目录即可使用，用法是像下面这样：

```cpp
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>

glm::vec3 = vector;
glm::mat4 = matrix;

// 下面是OpenGL的API用于传矩阵数据到uniform
glUniformMatrix4fv(location, 1, GL_FALSE, glm::value_ptr(matrix));
```

## MVP实现

MVP可以放到shader中实现，比如shader中增加一个传matrix uniform的函数：

```cpp
void Shader::setMatrix(const std::string& name, const GLfloat* value) const {
    int location = glGetUniformLocation(id, name.c_str());
    if (location == -1) {
        std::cerr << "WARNING::SHADER::UNIFORM_LOCATION_NOT_FOUND::" << name << std::endl;
    }
    glUniformMatrix4fv(location, 1, GL_FALSE, value);
}
```

然后再vertext shader中使用uniform并相乘：

```glsl
#version 400 core

layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;

out vec2 texCoord;

uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main() {
   gl_Position = projection * view* model * vec4(aPos.xyz, 1.0);
   texCoord = vec2(aTexCoord.x, aTexCoord.y);
}
```

## Camera实现

header file:

```cpp
#pragma once

#include <glad/glad.h>
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

// Defines several possible options for camera movement. Used as abstraction to stay away from window-system specific input methods
enum CameraMovement {
    FORWARD,
    BACKWARD,
    LEFT,
    RIGHT
};

// Default camera values
const float YAW = 0.0f;    // NOLINT
const float PITCH = 0.0f;  // NOLINT
const float MAX_PITCH = 89.0f;
const float SPEED = 2.5f;
const float SENSITIVITY = 0.1f;
const float FOV = 45.0f;

// An abstract camera class that processes input and calculates the corresponding Euler Angles, Vectors and Matrices for use in OpenGL
class Camera {
public:
    // camera attributes
    glm::vec3 position;
    glm::vec3 front{};
    glm::vec3 up{};
    glm::vec3 right{};
    glm::vec3 worldUp;
    // euler angles
    float yaw;
    float pitch;
    // camera options
    float movementSpeed;
    float mouseSensitivity;
    float zoom;

    // constructor with vectors
    Camera(glm::vec3 inPosition = glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3 inUp = glm::vec3(0.0f, 1.0f, 0.0f), float inYaw = YAW, float inPitch = PITCH)
        : position(inPosition)
        , front(glm::vec3(0.0f, 0.0f, -1.0f))
        , worldUp(inUp)
        , yaw(inYaw)
        , pitch(inPitch)
        , movementSpeed(SPEED)
        , mouseSensitivity(SENSITIVITY)
        , zoom(FOV) {
        updateCameraVectors();
    }

    // constructor with scalar values
    Camera(float posX, float posY, float posZ, float upX, float upY, float upZ, float inYaw, float inPitch)
        : position(glm::vec3(posX, posY, posZ))
        , worldUp(glm::vec3(upX, upY, upZ))
        , yaw(inYaw)
        , pitch(inPitch)
        , movementSpeed(SPEED)
        , mouseSensitivity(SENSITIVITY)
        , zoom(FOV) {
        updateCameraVectors();
    }

    // returns the view matrix calculated using Euler Angles and the LookAt Matrix
    glm::mat4 getViewMatrix() {
        return glm::lookAt(position, position + front, up);
    }

    // processes input received from any keyboard-like input system. Accepts input parameter in the form of camera defined ENUM (to abstract it from windowing systems)
    void processKeyboard(CameraMovement direction, float deltaTime);

    // processes input received from a mouse input system. Expects the offset value in both the x and y direction.
    void processMouseMovement(float xoffset, float yoffset, GLboolean constrainPitch = true);

    // processes input received from a mouse scroll-wheel event. Only requires input on the vertical wheel-axis
    void processMouseScroll(float yoffset);

private:
    void updateCameraVectors();
};

```

cpp file:

```cpp
#include "camera.hpp"

void Camera::processKeyboard(CameraMovement direction, float deltaTime) {
    float velocity = movementSpeed * deltaTime;
    if (direction == FORWARD) {
        position += front * velocity;
    }
    if (direction == BACKWARD) {
        position -= front * velocity;
    }
    if (direction == LEFT) {
        position -= right * velocity;
    }
    if (direction == RIGHT) {
        position += right * velocity;
    }
}

void Camera::processMouseMovement(float xoffset, float yoffset, GLboolean constrainPitch) {
    xoffset *= mouseSensitivity;
    yoffset *= mouseSensitivity;

    yaw += xoffset;
    pitch -= yoffset;

    if (constrainPitch) {
        if (pitch > MAX_PITCH) {
            pitch = MAX_PITCH;
        }
        if (pitch < -MAX_PITCH) {
            pitch = -MAX_PITCH;
        }
    }
    updateCameraVectors();
}

void Camera::processMouseScroll(float yoffset) {
    zoom -= yoffset;
    if (zoom < 1.f) {
        zoom = 1.f;
    }
    if (zoom > FOV) {
        zoom = FOV;
    }
}

void Camera::updateCameraVectors() {
    double x = cos(glm::radians(pitch)) * cos(glm::radians(yaw));
    double y = sin(glm::radians(pitch));
    double z = cos(glm::radians(pitch)) * sin(glm::radians(yaw));
    glm::vec3 newFront(x, y, z);
    front = glm::normalize(newFront);
    right = glm::normalize(glm::cross(front, worldUp));
    up = glm::normalize(glm::cross(right, front));
}
```
