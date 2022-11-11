一个简单的加载QtQuick应用的py脚本如下：

```python
import os
import sys

from PySide6 import QtCore, QtQuick, QtGui

if __name__ == '__main__':
    # Set up the application window
    app = QtGui.QGuiApplication(sys.argv)
    app.setApplicationDisplayName('QmlLearning')
    app.setApplicationName('QmlLearning')
    app.setDesktopFileName('QmlLearning')
    app.setOrganizationName('Nothing')
    view = QtQuick.QQuickView()
    view.setResizeMode(QtQuick.QQuickView.ResizeMode.SizeRootObjectToView)

    # Load the QML file
    qml_file = os.path.join(os.path.dirname(__file__), 'view.qml')
    view.setSource(QtCore.QUrl.fromLocalFile(qml_file))

    # Show the window
    if view.status() == QtQuick.QQuickView.Status.Error:
        sys.exit(-1)
    view.show()

    # execute and cleanup
    app.exec()
    del view

    sys.exit(0)
```
