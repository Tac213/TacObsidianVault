在QtQuick界面实际开发的过程中不可避免的会遇到qml前端和python后端之间的通信，此时需要通过把Python对象或类暴露给QML来完成这个过程。总的来说方法大概是：

- 逻辑从qml到python: qml端调用python对象的槽函数
- 逻辑从python到qml: python发射Signal，qml端通过槽函数处理

可以通过下面的方式在Python定义一个QObject的派生类，并将这个实例共享给qml:

```python
import random

from PySide6.QtCore import QObject, Signal, Slot

class NumberGenerator(QObject):
    def __init__(self):
        QObject.__init__(self)
    
    nextNumber = Signal(int, arguments=['number'])

    @Slot()
    def giveNumber(self):
        self.nextNumber.emit(random.randint(0, 99))


engine = QQmlApplicationEngine()  # QQmlEngine should be initialized after QGuiApplication
number_generator = NumberGenerator()
engine.rootContext().setContextProperty("numberGenerator", number_generator)
```

其中第一个参数是qml中的变量名，第二个参数变量的值。

在Qml中，需要使用`Connections`来连接该对象：

```qml
import QtQuick
import QtQuick.Window
import QtQuick.Controls

Window {
    id: root
    
    width: 640
    height: 480
    visible: true
    title: qsTr("Hello Python World!")
    
    Flow {
        Button {
            text: qsTr("Give me a number!")
            onClicked: numberGenerator.giveNumber()
        }
        Label {
            id: numberLabel
            text: qsTr("no number")
        }
    }

    Connections {
        target: numberGenerator
        function onNextNumber(number) {
            numberLabel.text = number
        }
    }
}
```

比如上面的示例中，Button被点击时调用了`numberGenerator`的`giveNumber`槽函数，`numberGenerator`的`nextNumber`信号发射时`numberLabel`的text会被改变。

如果不使用`QQmlEngine`而使用`QQuickView`，则通过以下方法：

```python
view = QtQuick.QQuickView()
view.setInitialProperties({'myModel': my_model})
# or
view.rootContext().setContextProperty("numberGenerator", number_generator)
```

也可以直接把一个python类暴露给qml，相当于在python里定义qml对象，里面的函数被调用时会直接跑python逻辑：

```python
QtQml.qmlRegisterType(NumberGenerator, 'Generators', 1, 0, 'NumberGenerator')
```

第1个参数为python定义的class对象，第2个参数为qml中import该对象时的路径，第3个参数为import该对象的主版本号，第4个参数为import该对象的副版本号，第5个参数为qml中实例化该对象时所使用的变量名。

比如上面这个类应该在qml中这样使用：

```qml
import Generators 1.0  // 1.0 is optional

NumberGenerator {
}
```

也可以不调用这个函数，通过import的方式来注册类，比如像下面这样写一个python模块：

```python
from PySide6.QtCore import QObject, Property
from PySide6.QtQml import QmlElement

# To be used on the @QmlElement decorator
# (QML_IMPORT_MINOR_VERSION is optional)
QML_IMPORT_NAME = "examples.adding.people"
QML_IMPORT_MAJOR_VERSION = 1


@QmlElement
class Person(QObject):

    def __init__(self, parent=None):
        super().__init__(parent)
        self._name = ''
        self._shoe_size = 0

    @Property(str)
    def name(self):
        return self._name

    @name.setter
    def name(self, n):
        self._name = n

    @Property(int)
    def shoe_size(self):
        return self._shoe_size

    @shoe_size.setter
    def shoe_size(self, s):
        self._shoe_size = s

```

然后只需要在Python执行`import xxx`这个语句之后，qml中就可以使用`Person`这个类，使用方法：

```qml
import examples.adding.people

Person {
    name: "Bob Jones"
    shoe_size: 12
}

```

在python实现的一些model也可以通过上面的这些方法暴露给qml使用，比如我们在python实现一个model的类暴露给qml，qml直接使用这个类名来实例化这个model；也可以在python实例化model之后，再把model的实例通过context传给qml。
