QML Attached Signal Handlers是一个非常好用的特性，最常用的方法就是连接Component上的信号，这样可以在构造完成和析构的时候都调用一个自定义的js function，相当于可以用js override构造和析构，写一些额外的逻辑：

```qml
Item {
    Component.onCompleted: () => {
        // will be called when contructed
        // 'this' point to the qml object being constructed
    }

    Component.onDestruction: () => {
        // will be called when destructing
    }
}
```
