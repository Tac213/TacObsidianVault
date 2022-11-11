首先需要一个conf文件：

```ini
[Controls]
Style=Material

[Universal]
Theme=System
Accent=Red

[Material]
Theme=Dark
Accent=Red
```

然后需要一个qrc的xml文件：

```xml
<!DOCTYPE RCC><RCC version="1.0">
<qresource prefix="/">
    <file>qtquickcontrols2.conf</file>
</qresource>
</RCC>
```

接着通过下面的命令行生成rc加载脚本：

```shell
pyside6-rcc style.qrc > style_rc.py
```

然后在python import这个加载的脚本即可：

```python
import style_rc  # pylint: disable=unused-import
```
