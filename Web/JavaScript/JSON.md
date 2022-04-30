JSON是一个全局变量，用于将object转为string或者将string转为object，类似python的json标准库的功能。

下面的代码将object转为string:

```js
JSON.stringify(obj)
```

下面的代码将string转为object:

```js
JSON.parse(jsonString);
```

因为转过来的只是一个不同object没法使用vscode的type checker，所以如果想用到这个功能的话，可以在目标类上实现一个类似这样的函数：

```js
class Transaction
{
    /**
     * @param {string} name
     * @param {number} amount 
     */
    constructor (name, amount)
    {
        this.id = Math.floor(Math.random() * 100000000);
        this.name = name;
        this.isIncome = amount > 0;
        this.amount = Math.abs(amount);
    }

    /**
     * @param {{ name: string; amount: number; id: number; isIncome: boolean; }} obj
     * @returns {Transaction}
     */
    static createWithJson(obj)
    {
        let transaction = new Transaction(obj.name, obj.amount);
        transaction.id = obj.id;
        transaction.isIncome = obj.isIncome;
        return transaction;
    }
}
```

这个createWithJson就是将object转变为对应的类的实例，调用了这个函数之后就可以愉快使用vscode的type checker。
