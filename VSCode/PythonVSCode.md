## launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "python",
            "request": "launch",
            "console": "integratedTerminal",
            "justMyCode": false,
            "program": "${workspaceFolder}/launch.py"
        },
        {
            "name": "Attach",
            "type": "python",
            "request": "attach",
            "console": "integratedTerminal",
            "justMyCode": false,
            "connect": {
                "host": "localhost",
                "port": 56778
            }
        }
    ]
}
```

有2种调试方式，一种是启动调试，一种是远程调试。

当request为launch时，就是启动调试；当request为attach时，就是远程调试。

无论启动调试还是远程调试，使用的都是[debugpy](https://github.com/microsoft/debugpy)。

### 启动调试

启动调试时，命令行类似下面:

```shell
python 'c:\Users\Tac\.vscode\extensions\ms-python.python-2022.4.1\pythonFiles\lib\python\debugpy\launcher' '9110' '--' 'path/to/your/py_file.py'
```

实际上是调用debugpy模块的功能来调试，如果能拿到vscode的端口号的话，也可以这样启动:

```shell
python -m debugpy --connect 'localhost:9110' 'path/to/your/py_file.py'
```

### 远程调试

远程调试有很多方法，一种是通过命令行启动debugpy adapter的同时启动你的目标python进程。

比如启动文件是`myfile.py`，则像下面这样启动:

```shell
python -m debugpy --listen localhost:5678 myfile.py
```

这样会启动一个adapter子进程作为服务器，vscode作为客户端连接adapter进程，连上了之后就能查看python进程的相关信息，也能打断点。

如果需要在adapter子进程启动后将整个python进程卡住等待vscode连接，可以这样启动:

```shell
python -m debugpy --listen localhost:5678 --wait-for-client myfile.py
```

通过python脚本import debugpy的方式来启动adapter进程:

```python
import debugpy
debugpy.configure(python=python_exe_path)  # 默认用sys.executable, 需要用python启动adapter
debugpy.listen(('localhost', 5678))
debugpy.wait_for_client()  # 不需要等待客户端连接可以把这行注释掉
```

## settings.json
使用pep8格式化代码，flake8检测代码，和Pylance作为languageServer:

```json
{
    "python.linting.flake8Enabled": true,
    "python.formatting.provider": "autopep8",
    "python.languageServer": "Pylance"
}
```

一行120个字:

```json
{
    "python.linting.flake8Args": [
        "--max-line-length=120"
    ],
    "python.formatting.autopep8Args": [
        "--max-line-length=120"
    ],
}
```

忽略flake8错误，比如下面忽略了不能用tab作为缩进:

```json
{
    "python.linting.flake8Args": [
        "--ignore=E128, W191"
    ],
}
```

配置PYTHONPATH可以通过下面这个字段，但注意只能是相对路径或者绝对路径，不能识别`${workspaceFolder}`等环境变量:

```json
{
    "python.analysis.extraPaths": [
        "Script",
        "Script/Lib",
        "SDK",
    ],
}
```

保存时使用autopep8自动格式化：

```json
{
    "[python]": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "ms-python.python"
    },
}
```

## extension.json

```json
{
    "recommendations": [
        "ms-python.python",
        "ms-python.vscode-pylance"
    ]
}
```