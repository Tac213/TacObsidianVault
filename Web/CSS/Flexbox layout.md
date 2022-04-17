[Flexbox-mdn web docs](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox)

Flex box 类似于Qt里面的水平布局或者垂直布局。

像下面这样操作，就给body使用了一个水平布局：

```css
body {
    display: flex;
    flex-direction: row;  /* this is optional, row is default */
    align-items: center;  /* this makes the subelement align to center*/
    justify-content: center;
}
```

像下面那这样操作，就给body使用了一个垂直布局：

```css
body {
    display: flex;
    flex-direction: column;
    align-items: center;  /* this makes the subelement align to center*/
    justify-content: center;
}
```

for more information, read the mdn web doc.