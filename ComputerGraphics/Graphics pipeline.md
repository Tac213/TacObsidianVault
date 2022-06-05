![graphics pipeline](../Images/Graphics_pipeline.png)

- Vertex Processing: 首先处理所有的三角形顶点信息，在[[Orthographic and Perspective Projection]]和[[Viewport transformation]]种完成
- Triangle Processing: 把顶点连成三角形
- Rasterization: 完成三角形的光栅化，在[[Rasterization and antialiasing and Z-Buffering]]中完成
- Fragment Processing: 对各个Fragment进行着色，如果使用了MSAA模型的话，Fragment就是一个采样点，否则就是一个单纯的像素，并且会完成[[Rasterization and antialiasing and Z-Buffering#Z-Buffering]]的过程
- Framebuffer Operations: 处理Z-Buffering算法所产生的framebuffer，将其现实到屏幕上

整个这个过程都是在显卡中完成的，其中有2个可编程的过程，分别为Vertex Processing和Fragment Processing，用于编程的程序被成为shader，即编辑shading point的着色过程。

[[Shading#shading frequency]]一节提到，选取shading point时，Gouraud shading选取顶点作为shading point，因此显卡把Vertext Processing这个过程提出来让应用可编程；Phong shading选择像素作为shading point，因此显卡把Fragment Processing这个过程提出来让应用可编程。

一个简单的shader:

```glsl
uniform sampler2D myTexture; // program parameter
uniform vec3 lightDir; // program parameter
varying vec2 uv; // per fragment value (interp. by rasterizer)
varying vec3 norm; // per fragment value (interp. by rasterizer)
void diffuseShader()
{
    vec3 kd;
    kd = texture2d(myTexture, uv); // material color from texture
    kd *= clamp(dot(–lightDir, norm), 0.0, 1.0); // Lambertian shading model
    gl_FragColor = vec4(kd, 1.0); // output fragment color
} 
```
