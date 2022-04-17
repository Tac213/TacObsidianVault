当传递实例方法作为回调参数时，需要使用bind this。

例如下面的代码：

```js
class World
{
    tick()
    {
        requestAnimationFrame(this.tick);
    }
}
```

第一次手动调用tick时是代码是没问题的，但是第二次由requestAnimationFrame来调用tick的时候，就会报错。报错原因是this是undefined。

为了解决这个问题，需要把上述代码改成:

```js
class World
{
    tick()
    {
        requestAnimationFrame(this.tick.bind(this));
    }
}
```

此时传给requestAnimationFrame的回调函数就知道this是哪个对象了，就不会出现undefined的问题。

或者换成把回调函数写成lambda函数也能解决这个问题，例如下面的代码:

```js
class World
{
    tick()
    {
        requestAnimationFrame(() => {this.tick()});
    }
}
```
