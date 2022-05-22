## `:not`

`:not`选择器可以对当前选择器进行过滤，过滤部分满足当前选择器规则的element。

比如下面这个选择器，可以选择到classList包含seat的element被鼠标hover的情况，但是classList既包含seat又包含occupied的element则不会被选择到。

```css
.seat:not(.occupied):hover {}
```
