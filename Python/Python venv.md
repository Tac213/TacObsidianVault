通常使用venv管理python项目，相关第三方依赖会安装在项目目录，不会安装在操作系统的安装目录。

拿到一个python项目时，通过如下方法创建一个venv:

```bash
python3 -m venv /path/to/venv
```

该命令会创建一个venv目录，路径为`/path/to/venv`，通常venv在项目根目录，起名为`venv`或者`.venv`，具体可以看项目的`.gitignore`把哪个目录作为忽略目录。

创建venv之后需要激活该venv才能使用venv下的解释器和pip。

```bash
# Linux / MacOS
source <venv>/bin/activate

# Win32
<venv>/Scripts/activate.bat

# Powershell
<venv>/Scripts/Activate.ps1
```

激活venv后，可以`which python3`或者`which pip3`看下现在的解释器和pip的路径是否在项目目录的venv目录下。在IDE中，将venv目录下的解释器作为IDE中所使用的解释器， 即可在IDE中使用venv环境调试项目。

接着用pip安装第三方依赖时都会安装在venv目录下。

为了保存第三方依赖信息，可以执行如下命令：

```bash
pip3 freeze > requirements.txt
```

其他人拿到项目之后，同样执行前面的激活venv的步骤，然后执行下面的命令即可一键安装所有第三方依赖。

```shell
pip3 install -r requirements.txt
```

在终端执行下面的命令即可取消venv的激活。

```bash
deactivate
```
