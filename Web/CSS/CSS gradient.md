gradient是一种渐变效果，分为3种：

- linear-gradient: 线性渐变，沿着一个线性方向进行颜色渐变
- radial-gradient: 径向渐变，沿着圆心向外椭圆形状向外扩展渐变
- conic-gradient: 圆锥渐变，绕着圆心方向渐变

[gradient mdn doc](https://developer.mozilla.org/en-US/docs/Web/CSS/gradient)
[linear-gradient mdn doc](https://developer.mozilla.org/en-US/docs/Web/CSS/gradient/linear-gradient)
[radial-gradient mdn doc](https://developer.mozilla.org/en-US/docs/Web/CSS/gradient/radial-gradient)
[conic-gradient mdn doc](https://developer.mozilla.org/en-US/docs/Web/CSS/gradient/conic-gradient)

[网上关于conic-gradient的中文笔记](https://www.cnblogs.com/coco1s/p/7079529.html)

linear-gradient示例：

```css
body {
    background-image: linear-gradient(0deg, rgba(247, 247, 247, 1) 23.8%, rgba(252, 221, 221, 1) 92%);
}
```

conic-gradient示例：

```css
.gradient-circle {
    background: conic-gradient(#55b7a4 0%, #4ca493 40%, #fff 40%, #fff 60%, #336d62 60%, #2a5b52 100%);
}
```
