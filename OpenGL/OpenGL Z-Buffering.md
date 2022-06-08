OpenGL已经实现了[[Rasterization and antialiasing and Z-Buffering#Z-Buffering]]算法，只需要在应用初始化时调用：

```cpp
// configure global opengl state
// -----------------------------
glEnable(GL_DEPTH_TEST);
```

在帧循环中清除z-buffer：

```cpp
glClear(GL_DEPTH_BUFFER_BIT);
```
