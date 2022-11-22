在QML中定义的对象可以通过js动态实例化，方法类似下面这样：

```js
var component;

function createImageObject() {
    component = Qt.createComponent("dynamic-image.qml");
    if (component.status === Component.Ready || component.status === Component.Error) {
        finishCreation();
    } else {
        component.statusChanged.connect(finishCreation);
    }
}

function finishCreation() {
    if (component.status === Component.Ready) {
        var image = component.createObject(root, {"x": 100, "y": 100});
        if (image === null) {
            console.log("Error creating image");
        }
    } else if (component.status === Component.Error) {
        console.log("Error loading component:", component.errorString());
    }
}
```

`createObject`方法的第二个参数可以传入实例化该对象时的一些属性的值，比如`component.createObject(root, {"x": 100, "y": 100})`表示其父对象为root，x和y属性都为100。

上面Component的创建过程是同步或异步的，Qt自行决定，但实例化的过程是同步的，我们也可以通过孵化器incubator将其设置为异步的：

```js
function finishCreate() {
    if (component.status === Component.Ready) {
        var incubator = component.incubateObject(root, {"x": 100, "y": 100});
        if (incubator.status === Component.Ready) {
            var image = incubator.object; // Created at once
        } else {
            incubator.onStatusChanged = function(status) {
                if (status === Component.Ready) {
                    var image = incubator.object; // Created async
                }
            };
        }
    }
}
```

在QML中定义的对象实际上也可以通过python脚本来实例化，方法跟用js创建一样，需要先创建component，再通过component创建qml对象:

```python
# Create a QML engine
engine = QtQml.QQmlEngine()
# Create a component factory and load the QML script
component = QtQml.QQmlComponent(engine)
component.loadUrl('person.qml')

# Create an instance of the component
person = component.create()
if person is not None:
    # qml object is instantiated successfully
    pass
else:
    for error in component.errors():
        print(error.toString())
```

如果想要异步创建实例，则需要在create时传入`QQmlIncubator`:

```python
# Define a custom QQmlIncubator
class MyIncubator(QtQml.QQmlIncubator):

    def statusChanged(self, status) -> None:
        """
        Will be called every time when status changed
        """
        super().statusChanged(status)
        if status == QtQml.QQmlIncubator.Status.Ready:
            # qml object is instantiated successfully
            print(self.object())


# Create a QML engine
engine = QtQml.QQmlEngine()
# Create a component factory and load the QML script
component = QtQml.QQmlComponent(engine)
component.loadUrl('person.qml')

# Create an incubator to instantiate asynchronously
incubator = MyIncubator(QtQml.QQmlIncubator.IncubationMode.Asynchronous)
# Create an instance of the component
person = component.create(incubator)
# person is None
print(person is None)
```

`component.create`方法除了传incubator，还可以再传一个QQmlContext，实例化QQmlContext的时候需要传入父对象的qmlcontext和父对象的qobject，类似于js中的第一个参数，然后可以调用`setContextProperty`或者`setContextProperties`定义实例化时的相关属性。

如果希望创建`QQmlComponent`的过程是异步的，则需要在实例化`QQmlComponent`时，指定qml的url和mode，比如：

```python
component = QtQml.QQmlComponent(engine, 'main.qml', QtQml.QQmlComponent.CompilationMode.Asynchronous)

def on_created(status):
    if status.QtQml.QQmlComponent.Status.Ready:
        # component is ready
        pass

component.statusChanged.connect(on_created)
```

此时该component就会走异步模式创建，和incubator不同，`QQmlComponent.statusChanged`是个signal，需要用connect来定义一个槽函数获取异步创建的结果。
