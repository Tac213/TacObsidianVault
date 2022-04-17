在vscode中开启ts-check，可以在javascript书写过程中享受如typescript般的体验，开启方式很简单：

```js
// @ts-check
```

但是ts-check没那么智能，有时候会无法推断变量的类型，此时可以手动加入[JSDoc](https://jsdoc.app/)，给ts-check提供帮助。

比如：

```js
/**
 * @param {number} b
 * @returns {void}
 */
function a(b) {}
```

上面的代码说明，形参b是一个number类型，函数a的返回值类型为void，即没有返回值。

对于自定义的类，也可以使用JSDoc

```js
class Actor {}

/**
 * @param {Actor} b
 * @returns {void}
 */
```

**在vscode中输入`/**`然后回车就会自动生成该格式的注释**。

带有泛型的JSDoc:

```js
/**
 * get all bricks
 * @returns {Array<Array<Brick>>}
 */
```

回调函数的JSDoc:

```js
/**
 * iterate all bricks
 * @param {(arg0: Brick) => void} iterationCallback
 * @param {boolean} isCheckVisible
 * @returns {void}
 */
iterateBricks(iterationCallback, isCheckVisible=false) {}
```

类的JSDoc:

```js
/**
 * spawn actor
 * @param {new (arg0: World) => Actor} actorClass
 * @returns {any}
 */
```

单个变量类型注释:

```js
/** @type {HTMLCanvasElement} */
let canvas = document.getElementById('canvas');
```
