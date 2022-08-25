目前主流的python debug adapter都是通过[pydevd](https://github.com/fabioz/PyDev.Debugger)实现的，比如[debugpy](https://github.com/microsoft/debugpy) / [ptvsd](https://github.com/microsoft/ptvsd) / [pydevd-pycharm](https://pypi.org/project/pydevd-pycharm/)。

这些debug adapter其实都是对pydevd的封装，所以直接理论上来说如果直接调用pydevd的接口就能自己通过代码暂停线程、恢复线程、执行step over或step into等指令。

## 如何import pydevd

正常情况下如果直接跑下面的脚本是会报错的：

```python
import pydevd
```

只有在debug模式下才能跑上面的脚本，因为debug模式下python debug adapter会把pydevd所在的路径加到`sys.path`里，此时就能正常import pydevd。所以可以写如下代码适配：

```python
try:
    import pydevd
except ModuleNotFoundError:
    _is_debugging = False
else:
    _is_debugging = True
```

后续所有的操作过程，只要`_is_debugger`是False就不执行任何操作直接return。

另外pydevd正常启动后会有一个global debugger，通常用`py_db`或者`pydb`这个变量名来表示这个global debugger，通过下面的代码可以拿到pydb:

```python
import pydevd
py_db = pydevd.get_global_debugger()
```

如果当前不处于debug模式，或者debug adapter的连接已经断开了，那么`py_db`就会拿到`None`。

如果需要恢复`py_db`，那么就要通过debug adapter重新进入debug模式，不如debugpy重新调用下面这段脚本：

```python
import debugpy
debugpy.listen(('localhost', 5678))
```

## breakpoint

断点其实就是暂停当前线程，参考`debug.server.api.breakpoint`的实现，代码如下:

```python
import inspect
import pydevd


def breakpoint() -> None:
    """
    Suspend current thread.
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    stop_at_frame = inspect.currentframe().f_back
    while (
        stop_at_frame is not None
        and py_db.get_file_type(stop_at_frame) == py_db.PYDEV_FILE
    ):
        stop_at_frame = stop_at_frame.f_back
    pydevd.settrace(
        suspend=True,
        trace_only_current_thread=True,
        patch_multiprocessing=False,
        stop_at_frame=stop_at_frame,
    )
```

函数执行效果就是会暂停调用这个函数的线程。

## suspend and resume thread

暂停和恢复线程在`pydevd`这个模块里找了很久没找到对应的代码，最后发现了一个`_pydevd_bundle.pydevd_process_net_command_json`这个模块，这个模块就是用来处理debug adapter的操作请求的，比如`PyDevJsonCommandProcessor.on_pause_request`用来处理暂停线程请求，`PyDevJsonCommandProcessor.on_continue_request`用来处理恢复线程请求，那么就参考这2个函数的实现即可。

这两个函数都是调用`_pydevd_bundle.pydevd_api`模块中的相应函数完成对应处理的，所以也可以照着葫芦画瓢：

```python
import threading
import pydevd
from _pydevd_bundle import pydevd_api
from _pydevd_bundle import pydevd_constants


def suspend_thread(thread: threading.Thread = None) -> None:
    """
    Suspend a thread.
    If thread is not given, all threads will be suspended.
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    if not thread:
        thread_id = '*'
    else:
        thread_id = pydevd_constants.get_thread_id(thread)
    pydevd_api.PyDevdAPI().request_suspend_thread(py_db, thread_id)


def resume_thread(thread: threading.Thread = None) -> None:
    """
    Resume a thread.
    If thread is not given, all threads will be resumed.
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    if not thread:
        thread_id = '*'
    else:
        thread_id = pydevd_constants.get_thread_id(thread)
    pydevd_api.PyDevdAPI().request_resume_thread(thread_id)
```

## step command

接着参考`PyDevJsonCommandProcessor`中的`on_xxx_request`函数的相关实现，代码如下：

```python
import threading
import pydevd
from _pydevd_bundle import pydevd_api
from _pydevd_bundle import pydevd_constants
from _pydevd_bundle import pydevd_comm_constants


def step_over(thread: threading.Thread) -> None:
    """
    do command: Step Over
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    step_cmd_id = pydevd_comm_constants.CMD_STEP_OVER
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)


def step_into(thread: threading.Thread, just_my_code=True) -> None:
    """
    do command: Step Into
    Args:
        thread: threading.Thread
        just_my_code: [bool]step to my code
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    if just_my_code:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_INTO_MY_CODE
    else:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_INTO
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)


def step_out(thread: threading.Thread, just_my_code=True) -> None:
    """
    do command: Step Out
    Args:
        thread: threading.Thread
        just_my_code: [bool]step to my code
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    if just_my_code:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_RETURN_MY_CODE
    else:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_RETURN
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)
```

## 测试代码

```python
import debugpy
import threading


def other_thread():
    print('other_thread')
    import time
    time.sleep(5.0)
    print('sleep end')
    import debug_handler
    debug_handler.step_into(threading.main_thread())
    time.sleep(1.0)
    print('sleep end1')
    debug_handler.step_over(threading.main_thread())
    time.sleep(1.0)
    print('sleep end2')
    debug_handler.step_over(threading.main_thread())
    time.sleep(1.0)
    print('sleep end3')
    debug_handler.step_over(threading.main_thread())


def test():
    print('testtttttt')


def main():
    print('Start')
    thread = threading.Thread(target=other_thread)
    thread.start()
    debugpy.listen(('localhost', 39494))
    print('11111111111')
    import debug_handler
    debug_handler.breakpoint()
    test()
    print('333')


if __name__ == '__main__':
    main()
```

执行效果：

```
Start
other_thread
11111111111
sleep end
sleep end1
testtttttt
sleep end2
333
sleep end3
```

整个debug_handler模块的代码如下：

```python
import inspect
import threading
_is_debugging = False
try:
    import pydevd
    from _pydevd_bundle import pydevd_api
    from _pydevd_bundle import pydevd_constants
    from _pydevd_bundle import pydevd_comm_constants
except ModuleNotFoundError:
    _is_debugging = False
else:
    _is_debugging = True


def breakpoint() -> None:
    """
    Suspend current thread.
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    stop_at_frame = inspect.currentframe().f_back
    while (
        stop_at_frame is not None
        and py_db.get_file_type(stop_at_frame) == py_db.PYDEV_FILE
    ):
        stop_at_frame = stop_at_frame.f_back
    pydevd.settrace(
        suspend=True,
        trace_only_current_thread=True,
        patch_multiprocessing=False,
        stop_at_frame=stop_at_frame,
    )


def suspend_thread(thread: threading.Thread = None) -> None:
    """
    Suspend a thread.
    If thread is not given, all threads will be suspended.
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    if not thread:
        thread_id = '*'
    else:
        thread_id = pydevd_constants.get_thread_id(thread)
    pydevd_api.PyDevdAPI().request_suspend_thread(py_db, thread_id)


def resume_thread(thread: threading.Thread = None) -> None:
    """
    Resume a thread.
    If thread is not given, all threads will be resumed.
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    if not thread:
        thread_id = '*'
    else:
        thread_id = pydevd_constants.get_thread_id(thread)
    pydevd_api.PyDevdAPI().request_resume_thread(thread_id)


def step_over(thread: threading.Thread) -> None:
    """
    do command: Step Over
    Args:
        thread: threading.Thread
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    step_cmd_id = pydevd_comm_constants.CMD_STEP_OVER
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)


def step_into(thread: threading.Thread, just_my_code=True) -> None:
    """
    do command: Step Into
    Args:
        thread: threading.Thread
        just_my_code: [bool]step to my code
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    if just_my_code:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_INTO_MY_CODE
    else:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_INTO
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)


def step_out(thread: threading.Thread, just_my_code=True) -> None:
    """
    do command: Step Out
    Args:
        thread: threading.Thread
        just_my_code: [bool]step to my code
    Returns:
        None
    """
    if not _is_debugging:
        return
    py_db = pydevd.get_global_debugger()
    if not py_db:
        return
    thread_id = pydevd_constants.get_thread_id(thread)
    if just_my_code:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_RETURN_MY_CODE
    else:
        step_cmd_id = pydevd_comm_constants.CMD_STEP_RETURN
    pydevd_api.PyDevdAPI().request_step(py_db, thread_id, step_cmd_id)
```
