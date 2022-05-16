Python reload的过程与[[Python import]] [Python import](Python%20import.md)息息相关，因为reload其实就是把module重新import一次而已。

通过`importlib.reload(module)`函数来reload一个模块。这个函数的流程如下：

首先拿到module的fullname。

然后找到父module，并用父module的__path__作为submodule_search_locations。如果没有父module，submodule_search_locations就是None。

接着和import的过程一样，遍历[[Python import#sys meta_path]]，调用find_spec方法找到module的[[Python import#ModuleSpec]]，与import不同的是，会多传一个target参数，target就是原本的module。

最后调用spec.loader的exec_module函数，更新module的__dict__全局变量空间。

简化后的代码如下：

```python
def reload(module):
    try:
        name = module.__spec__.name
    except AttributeError:
        try:
            name = module.__name__
        except AttributeError:
            raise TypeError('reload() argument must be a module')
    if sys.modules.get(name) is not module:
        raise ImportError('module {} not in sys.modules'.format(name), name=name)
    parent_name = name.rpartition('.')[0]
    if parent_name:
        try:
            parent = sys.modules[parent_name]
        except KeyError:
            raise ImportError('parent {!r} not in sys.modules') from None
        else:
            pkgpath = parent.__path__
    else:
        pkgpath = None
    target = module
    spec = None
    for finder in sys.meta_path:
        spec = finder.find_spec(name, pkgpath, target)  # different from import
        if spec:
            break
    if spec is None:
        raise ModuleNotFoundError('No module named {!r}'.format(name), name=name)
    module.__spec__ = spec
    _init_module_attrs(spec, module, override=True)
    spec.loader.exec_module(module)
    return module
```

需要注意的是，reload过程中并没有调用`create_module`函数重新创建一个新的module，所以__dict__全局变量空间中，旧的全局变量并不会被删除。也就是说如果新模块没有第一在旧版本模块中定义的名称，则将保留旧版本中的定义。
