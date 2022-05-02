通过关键帧来定义自定义动画，比如：

```css
@keyframes bounce {
    0%,
    100% {
        transform: translateY(0);
    }
    50% {
        transform: translateY(-10px);
    }
}
```

上面的意思是定义bounce动画的关键帧，0%时间时属性无变化，50%时间时Y轴负方向移动10px，100%时间时回到原来的位置。

第一动画后，即可通过animation字段使用自定义动画：

```css
/* circle 是个div 这里把div画成一个圈 */
.circle {
    background-color: #fff;
    width: 10px;
    height: 10px;
    border-radius: 50%;
    margin: 5px;
    animation: bounce 0.5s ease-in infinite;
}
```
