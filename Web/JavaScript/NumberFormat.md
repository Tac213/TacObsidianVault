[参考链接](https://stackoverflow.com/questions/149055/how-to-format-numbers-as-currency-strings)

```js
let moneyFormatter = new Intl.NumberFormat('en-us', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0,
});
moneyFormatter.format(1_000_000);
```
