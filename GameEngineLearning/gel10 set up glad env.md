glad是通过脚本下载下来的，将其放到Engine/External/Source/glad这个目录下。

这里学习了Unreal，在不同的平台通过不同的脚本入口文件，setup开发环境，windows写bat，linux写sh，mac写command。

因为环境设置过程比较复杂，所以统一用python脚本来写，所以开发环境多了一个：需要安装python3和pip。

## windows

bat文件比较简单，就是设置一些环境变量，并调用python脚本：

```shell
set TAMASHIIROOT=..\..
set UNIVERSALSCRIPTSROOT=%TAMASHIIROOT%\Scripts\Universal
set PYTHONIOENCODING=utf8
cls
py -3 %UNIVERSALSCRIPTSROOT%\setup.py
```

python脚本很简单，首先写了一个path_handler用于处理各种路径：

```python
import os

CWD = os.getcwd()
TAMASHII_ROOT = os.path.realpath(os.environ['TAMASHIIROOT'])
PY_SCRIPTS_ROOT = os.path.realpath(os.environ['UNIVERSALSCRIPTSROOT'])

ENGINE_DIR = os.path.join(TAMASHII_ROOT, 'Engine')
BUILD_DIR = os.path.join(TAMASHII_ROOT, 'Build')

EXTERNAL_DIR = os.path.join(ENGINE_DIR, 'External')
```

接着写setup glad:

```python
def setup() -> None:
    senv.logger.info('Setup %s.', GLAD_NAME)
    glad_path = os.path.join(
        path_handler.EXTERNAL_DIR,
        'Source',
        GLAD_NAME,
    )
    if not os.path.isdir(glad_path):
        senv.logger.info(
            'Path: \'%s\' does not exist, try to make dir',
            glad_path
        )
        os.makedirs(glad_path)
    try:
        importlib.import_module(GLAD_NAME)
    except ModuleNotFoundError:
        senv.logger.info(
            'Failed to import \'%s\', try to install it with pip',
            GLAD_NAME
        )
        returncode = subprocess.Popen(
            [
                sys.executable,
                '-m',
                'pip',
                'install',
                '--upgrade',
                'pip',
            ]
        ).wait()
        if returncode:
            senv.logger.error(
                'Failed to upgraded pip. '
                'Please install pip first.'
            )
            return
        subprocess.Popen(
            [
                sys.executable,
                '-m',
                'pip',
                'install',
                GLAD_NAME,
            ]
        ).wait()
    subprocess.Popen(
        [
            GLAD_NAME,
            '--generator=c',
            '--spec',
            'gl',
            f'--out-path={glad_path}',
        ]
    ).wait()
    subprocess.Popen(
        [
            GLAD_NAME,
            '--generator=c',
            '--spec',
            'wgl',
            f'--out-path={glad_path}',
        ]
    ).wait()
    subprocess.Popen(
        [
            GLAD_NAME,
            '--generator=c',
            '--spec',
            'glx',
            f'--out-path={glad_path}',
        ]
    ).wait()
    senv.logger.info('Finished.')
```

代码看起来很复杂，实际上等同于在执行下面的shell：

```shell
py -3 -m pip install --upgrade pip
py -3 -m pip install glad
glad --generator=c --spec gl --out-path=$glad_path
glad --generator=c --spec wgl --out-path=$glad_path
glad --generator=c --spec glx --out-path=$glad_path
```

## linux

python脚本没有变化，翻译以下shell即可：

```shell
export TAMASHIIROOT="`dirname "$0"`"/../..
export UNIVERSALSCRIPTSROOT=$TAMASHIIROOT/Scripts/Universal
export PYTHONIOENCODING=utf8
clear
python3 $UNIVERSALSCRIPTSROOT/setup.py
```

## mac

mac的command直接调用Linux的shell即可：

```shell
bash "`dirname "$0"`"/../Linux/Setup.sh
```

不过mac安装过程可能会报ssl的错误，此时应该运行`/Applications/Python 3.10/Install Certificates.command`这个脚本，执行完之后就能成功下载glad。

不过由于这个脚本很简单，而且不太好通过Python脚本找到这个脚本的绝对路径(主要是不知道用户把python装在哪里)，就直接把这个脚本抄了过来，当glad下载出错时，再执行自己写的这段脚本，执行完成后重新执行glad:

```python
returncode = subprocess.Popen(
    [
        GLAD_NAME,
        '--generator=c',
        '--spec',
        'gl',
        f'--out-path={glad_path}',
    ]
).wait()
if returncode and platform.system() == 'Darwin':
    import install_certificates
    install_certificates.setup()
    subprocess.Popen(
        [
            GLAD_NAME,
            '--generator=c',
            '--spec',
            'gl',
            f'--out-path={glad_path}',
        ]
    ).wait()
```
