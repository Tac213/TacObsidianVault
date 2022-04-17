有时候我们需要一个盒子铺满整个页面，可能是横向铺满或者纵向铺满，但是不同设备的页面大小不一样，我们不能准确得到像素数值。

此时可以用vw和vh单位，其实就是view width和view height的意思。

比如下面的代码：

```css
div {
	height: 100vh;
}
```

意思是高度为100%的view height，即铺满整个页面。

比如下面的代码：

```css
div {
	width: 50vw;
}
```

意识是宽度为50%的view width, 即页面宽度的一半。