[参考文章](https://zhuanlan.zhihu.com/p/28679304)

这些矩形都是由CPU绘制的。

## windows

在windowProc函数中处理事件时，可以绘制矩形，代码如下：

```cpp
switch (message) {
case WM_PAINT:
    {
        PAINTSTRUCT ps;
        HDC hdc = BeginPaint(hwnd, &ps);
        RECT rect = {20, 20, 60, 80};  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
        HBRUSH brush = static_cast<HBRUSH>(GetStockObject(DKGRAY_BRUSH));

        FillRect(hdc, &rect, brush);

        EndPaint(hwnd, &ps);
        break;
    }
}
```

代码很简单，就是创建一个矩形和一个刷子，用刷子给矩形上色。

## Linux

在tick函数中，遇到XCB_EXPOSE事件时，可以绘制矩形，代码如下：

```cpp
switch (event->response_type & ~0x80) {  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
case XCB_EXPOSE: {
    xcb_rectangle_t rect = {20, 20, 60, 80};  // NOLINT(cppcoreguidelines-avoid-magic-numbers)
    xcb_gcontext_t foreground = xcb_generate_id(mpConn);
    uint32_t mask = XCB_GC_FOREGROUND | XCB_GC_GRAPHICS_EXPOSURES;
    std::array<uint32_t, 2> values{mpScreen->black_pixel, 0};
    xcb_create_gc(mpConn, foreground, mWindow, mask, &values);
    xcb_poly_fill_rectangle(mpConn, mWindow, foreground, 1, &rect);
    xcb_flush(mpConn);
    break;
}
}
```

代码也很简单，同样是创建一个矩形，但时用一个context来填充他，这个context的代码与应用初始化时基本相同，颜色取自`mpScreen->black_pixel`，调`xcb_poly_fill_rectangle`填充矩形后，调`xcb_flush`即可刷新界面。
