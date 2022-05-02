svg是w3c xml的分支语言之一，用标记语言绘制适量图形，目前在主流浏览器中都有实现，比如下面的代码可以画一条线：

```html
<svg height="250" width="200" class="figure-container">
    <line x1="60" y1="20" x2="140" y2="20" />
</svg>
```

画好线之后可以在css里面定义线的一些绘制属性：

```css
.figure-container {
    fill: transparent;
    stroke: #fff;
    stroke-width: 4px;
    stroke-linecap: round;
}
```

详情阅读[该文档](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial)。
