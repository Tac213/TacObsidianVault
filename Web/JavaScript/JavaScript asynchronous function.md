JavaScript支持async函数，但比起Python相对简单，直接调用async函数就能发起异步任务，而不用像python那样。

如果直接调用async函数，返回的是一个Promise对象，Promise对象可以被await，await表达式的结果就是async函数的返回值。

比如下面这个函数：

```js
async function fetchRandomUser()
{
    /**@type {Response} */
    const res = await fetch('https://randomuser.me/api');
    // for more infomation, visit: https://randomuser.me
    const userData = await res.json();
    return userData;
}
```

如果直接调用：

```js
const result = fetchRandomUser();
```

这样result就只是一个Promise对象而已，拿不到这个函数真正的返回值，如果要拿到这个函数真正的返回值，需要await，而且下面这段代码必须要在另一个async函数中：

```js
const result = await fetchRandomUser();
```

async函数的[[JSDoc]]应该这样写，在尖括号里面写真正的返回值类型：

```js
/**
 * @returns {Promise<any>}
 */
async function fetchRandomUser()
{
    /**@type {Response} */
    const res = await fetch('https://randomuser.me/api');
    // for more infomation, visit: https://randomuser.me
    const userData = await res.json();
    return userData;
}
```

## Promise对象

TypeScript中的Promise对象定义如下：

```ts
/**
 * Represents the completion of an asynchronous operation
 */
interface Promise<T> {
    /**
     * Attaches callbacks for the resolution and/or rejection of the Promise.
     * @param onfulfilled The callback to execute when the Promise is resolved.
     * @param onrejected The callback to execute when the Promise is rejected.
     * @returns A Promise for the completion of which ever callback is executed.
     */
    then<TResult1 = T, TResult2 = never>(onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | undefined | null, onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | undefined | null): Promise<TResult1 | TResult2>;

    /**
     * Attaches a callback for only the rejection of the Promise.
     * @param onrejected The callback to execute when the Promise is rejected.
     * @returns A Promise for the completion of the callback.
     */
    catch<TResult = never>(onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | undefined | null): Promise<T | TResult>;
}
```

所以在同步函数中，其实也可以像下面这样写：

```js
fetch('some/url').then(res => {
    const data = res.json();
})
```

这样fulfill时传进去的匿名函数就会被调用，匿名函数带有一个形参，就是async函数的返回值。

也可以自行new一个Promise对象，比如：

```js
const promise = new Promise((resolve) = {
    // define how to call resolve here
})
```

需要传一个函数参数，该函数参数接受一个名为resolve的函数参数作为形参，resolve是await结束时要执行的回调，也就是then函数里面传的值。所以我们应该在这个函数参数里面定义什么时候调用resolve。

比如可以用下面这个方法在async函数里面await固定毫秒：

```js
async function foo() {
    doSomething();
    await new Promise((resolve) => setTimeout(resolve, 2000));
    doSomethingAfterFewSeconds();
}
```
