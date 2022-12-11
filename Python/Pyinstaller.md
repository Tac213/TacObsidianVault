通过spec文件配置打包信息，对python项目进行打包。

spec文件本质上是个python模块，再打包的过程中被Pyinstaller导入并执行，根据[官方文档](https://pyinstaller.org/en/stable/spec-files.html)的介绍，主要需要提供4个不同类型的变量：

- a: Analysis的实例，主要作用是配置一些输入信息，比如入口脚本、隐式import、PYTHONPATH、额外data数据等等
- pyz: PYZ的实例，用于注入Analysis实例中的python模块信息
- exe: EXE的实例，用来配置executable的相关信息，比如名字、icon、是否debug模式、是否启用控制台
- coll: COLLECT的实例，用来配置最终输出的目录中要有哪些信息
- bundle: mac平台上特有的BUNDLE对象，用于将coll额外输出成`xxx.app`目录，即mac平台上常见可运行的app(注意会同时输出bundle和coll)

以上类可以通过下面的方式import:

```python
from PyInstaller.building.build_main import Analysis
from PyInstaller.building.api import PYZ, EXE, COLLECT
from PyInstaller.building.osx import BUNDLE
```

通常通过指令自动生成一个spec文件，然后在此基础上修改，比如：

```bash
pyi-makespec launcher.py --name AppName --windowed
```

以上命令会在当前目录下生成一个`AppName.spec`，然后根据自己的需求修改spec文件即可。

打包app时除了打包还要做很多其他额外的操作，比如编译Qt资源，因此适合把打包写成一个py脚本，在python中通过下面的方式调用PyInstaller进行打包：

```python
import PyInstaller.__main__

PyInstaller.__main__.run([
    os.path.join(os.path.dirname(__file__), 'spec', f'{sys.platform}.spec'),
    '--distpath',
    args.distpath,
    '--workpath',
    args.workpath,
    '--noconfirm',
])
```
