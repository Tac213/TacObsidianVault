Python的import流程在importlib这个标准库中有python源码版本可以看，在知道某个模块的全名时，推荐用下面的方法import:

```python
import importlib
module = importlib.import_module('a.b.c')
```

实际上python import的过程很复杂，import过程中涉及到多个不同类型的对象，import流程带有一些hook可以由开发者自定义。

与import hook相关的对象：

- `sys.meta_path`
- `sys.path_hooks`

这2个对象的作用会在后文解释。

## import_module函数

importlib.import_module是导入模块用的函数，通常只需要传1个参数，意味绝对导入，也就是说`importlib.import_module('a.b.c')`等效于`import a.b.c`。

当需要相对导入时，就需要传入2个参数，第二个参数需要传相对于哪个package导入，也就是说`importlib.import_module('d.e', 'a.b')`等效于在`a.b`这个package里执行`from .d import e`。

这个函数实际上只是对`importlib._bootstrap._gcd_import`函数的封装，详见下文import流程。

## import过程中使用到的对象

- Finder: 又分为BuiltinFinder / FrozenFinder / PathFinder / FileFinder这4种，前2种在import builtin模块和Frozen模块使用，后2种在import普通python源码模块时被使用，PathFinder用于寻找模块可能存在的目录路径，并把这个目录路径传给FileFinder，FilerFinder根据PathFinder传来的目录路径在这个目录下搜索对应的python源码模块
- `sys.meta_path`: 见下文
- `sys.path_hooks`: 见下文
- ModuleSpec: `importlib.machinery.ModuleSpec`的实例，有非常多属性会被module用到，详见下文的import流程
- Loader: `importlib.abc.Loader`的实例，会被ModuleSpec持有，实现create_module方法创建模块实例，实现exec_module方法解析模块内容

## import流程

源码在`importlib._bootstrap`这个文件里，入口函数是`_gcd_import`，该函数带有3个参数:

- name: 导入时传入的模块名，比如`d.e`
- package: 相对导入时传入的包名，默认为None
- level: 相对导入的等级，前面有几个`.`就是几，默认为0

示例:

```python
import a.b.c
from a.b import c
```

上面2个都相当于：

```python
_gcd_import('a.b.c')
```

```python
# current package location: 'a.b'
from .d import e
```

上面的环境和代码相当于：

```python
_gcd_import('b.e', package='a.b', level=1)
```

```python
# current package location: 'a.b'
from .. import f
```

上面的环境和代码相当于：

```python
_gcd_import('f', package='a.b', level=2)
```

这个函数首先把package和level的要素算上，得出绝对import的模块名fullname(这块逻辑很简单详见`importlib._bootstrap._resolve_name`函数)。

接着，根据模块fullname，判断模块的父模块是否有import过(比如`a.b.c`的父模块是`a.b`)，如果父模块没有被import过，则将父模块的fullname递归地传入`_gcd_import`中import父模块。

父模块import完成后，开始当前模块的import，此处开始**进入重点**。

首先遍历`sys.meta_path`中的每个finder对象，调用每个finder对象的`find_spec`函数（旧版本的python为`find_module`），如果`find_spec`函数返回了一个ModuleSpec对象，则终止循环，进入下一步。如果所有的`find_spec`函数都没有返回ModuleSpec对象，则抛出ModuleNotFoundError: No module named xxxxx

有了ModuleSpec对象后，调用spec.loader对象的`create_module`函数创建module，如果这个函数返回None，则通过默认的`type.ModuleType(spec.name)`的方式创建module。

module创建完成后，调用`_init_module_attrs(spec, module)`来初始化module的`__name__`, `__loader__`, `__package__`, `__spec__`, `__path__`, `__file__`, `__cached__`，各属性和spec的关系如下：

- `__name__`: spec.name
- `__loader__`: spec.loader, Loader的实例
- `__package__`: spec.parent, module所在的package名
- `__spec__`: spec本身
- `__path__`: spec.submodule_search_locations, 子模块搜索路径列表
- `__file__`: spec.origin, module所在的路径，有这个属性前提是spec.has_location是True
- `__cached__`: spec.cached, module的pyc所在的路径，有这个属性的前提是spec.has_location并且spec.cached不是None

最后，将module加到`sys.modules`中，调用`spec.exec_module`函数，将模块中的脚本编译并写入模块的命名空间。

整个流程的简化代码如下：

```python
import sys
import types

def _gcd_import(name, package=None, level=0) -> types.ModuleType:
    """
    Args:
        name: [str]import时传入的模块名，比如'd.e'
        package: [str]相对import时传入的package名, None为绝对import
        level: [int]相对import时的相对level，name前面有几个.就是几
    Returns:
        type.ModuleType
    """
    fullname = _resolve_name(name, package, level)  # 全部转为绝对import
    if fullname in sys.modules:
        return sys.modules[fullname]

    path_list = None  # 模块搜索路径列表
    parent_module_name = fullname.rpartition('.')[0]
    if parent_module_name:
        if parent_module_name not in sys.modules:
            _gcd_import(parent_module_name)
        parent_module = sys.modules[parent_module_name]
        path_list = parent_module.__path__

    spec = None
    for finder in sys.meta_path:
        spec = finder.find_spec(fullname, path_list)
        if spec:
            break
    if spec is None:
        raise ModuleNotFoundError('No module named {!r}'.format(fullname), name=fullname)

    module = None
    if spec.loader is not None and hasattr(spec.loader, 'create_module'):
        module = spec.loader.create_module(spec)
    if module is None:
        module = types.ModuleType(spec.name)

    _init_module_attrs(spec, module)  # 具体哪些属性见上文

    spec._initializing = True
    try:
        sys.modules[spec.name] = module
        try:
            if spec.loader is None:
                raise ImportError('missing loader', name=spec.name)
            else:
                spec.loader.exec_module(module)
        except:
            del sys.modules[spec.name]
            raise
    finally:
        spec._initializing = False
    return module
```

### `_bootstrap.py`中可以参考的其他变量或函数或类

`_module_repr(module)`: 相当于`types.ModuleTye.__repr__`
`spec_from_loader(name, loader, *, orgin=None, is_package=None)`: 通过Loader创建ModuleSpec实例
`ModuleSpec`: ModuleSpec的类
`module_from_spec(spec)`: 通过ModuleSpec创建module实例
`BuiltinImporter`: builtin模块的Finder
`FrozenImporter`: frozen模块的Finder

### `_bootstrap_external.py`中可以参考的其他变量或函数或类

`_PYCACHE`: `'__pycache__'`
`SOURCE_SUFFIXES`: `['.py']`, 如果是windows还多一个`'pyw'`
`EXTENSION_SUFFIXES`: `['.cp310-win_amd64.pyd', '.pyd']`, 第一个根据python版本不同而不同
`BYTECODE_SUFFIXES`: `['.pyc']`
`cache_from_source(path)`: 返回py源码文件路径对应的pyc文件路径
`source_from_cache(path)`: 返回pyc文件路径所对应的py源码文件路径
`spec_from_file_location(name, location=None, *, loader=None, submodule_search_locations=_POPULATE)`: 根据py文件路径得到对应的ModuleSpec对象
`SourceFileLoader(name, path)`: 可以加载`SOURCE_SUFFIXES`的Loader类
`SourcelessFileLoader(name, path)`: 可以加载`BYTECODE_SUFFIXES`的Loader类
`ExtensionFileLoader(name, path)`: 可以加载`SOURCE_SUFFIXES`的Loader类
`PathFinder`: 普通Python模块的Finder

### 上述变量、函数、类的常规import方式

```python
from importlib.machinery import ModuleSpec, BuiltinImporter, FrozenImporter, PathFinder, FileFinder, SourceFilerLoader, SourcelessFileLoader, ExtensionFileLoader, SOURCE_SUFFIXES, BYTECODE_SUFFIXES, EXTENSION_SUFFIXES, all_suffixes

from importlib.util import module_from_spec, spec_from_loader, spec_from_file_location
```

## `sys.meta_path`

是一个list, 每一项是一个Finder对象，其功能是根据import的模块的全名比如'a.b.c'搜索该模块可能存在的路径，python启动时默认有3个，按顺序是builtin模块的Finder、frozen模块的Finder、普通Python文件的PathFinder。这3个对象的python源码类定义分别是：

- builtin模块的Finder: `importlib._bootstrap.BuiltinImporter`, 可以通过`sys.modules['importlib._bootstrap'].BuiltinImporter`拿到这个对象，这个表达式会返回True: `sys.meta_path[0] is sys.modules['importlib._bootstrap'].BuiltinImporter`
- frozen模块的Finder: `importlib._bootstrap.FrozenImporter`, 可以通过`sys.modules['importlib._bootstrap'].FrozenImporter`拿到这个对象，这个表达式会返回True: `sys.meta_path[1] is sys.modules['importlib._bootstrap'].FrozenImporter`
- 普通Python文件的PathFinder: `importlib._bootstrap_external.PathFinder`, 可以通过`sys.modules['importlib._bootstrap_external'].PathFinder`拿到这个对象，这个表达式会返回True: `sys.meta_path[2] is sys.modules['importlib._bootstrap_external'].PathFinder`

### BuiltinImporter

这个类既是builtin模块的Finder又是builtin模块的loader。find_spec的实现如下：

```python
    @classmethod
    def find_spec(cls, fullname, path=None, target=None):
        if path is not None:
            return None
        if _imp.is_builtin(fullname):
            return spec_from_loader(fullname, cls, origin=cls._ORIGIN)
        else:
            return None
```

实现很简单，就是通过builtin的_imp模块判断模块名是否是builtin模块，如果是的话就调用spec_from_loader返回builtin模块的spec，这个spec的loader还是cls本身。也因此，该类也实现了loader的create_module和exec_module:

```python
    @staticmethod
    def create_module(spec):
        """Create a built-in module"""
        if spec.name not in sys.builtin_module_names:
            raise ImportError('{!r} is not a built-in module'.format(spec.name),
                              name=spec.name)
        return _call_with_frames_removed(_imp.create_builtin, spec)

    @staticmethod
    def exec_module(module):
        """Exec a built-in module"""
        _call_with_frames_removed(_imp.exec_builtin, module)
```

都是通过调用_imp模块的c函数去创建并执行builtin模块的。

### FrozenImporter

这个类既是Frozen模块的Finder又是Frozen模块的loader。find_spec实现如下：

```python
    @classmethod
    def find_spec(cls, fullname, path=None, target=None):
        if _imp.is_frozen(fullname):
            return spec_from_loader(fullname, cls, origin=cls._ORIGIN)
        else:
            return None
```

`importlib._bootstrap`这个模块本身就是一个frozen模块，调用_imp.is_frozen判断模块名是否是frozen模块，如果是的话就调用spec_from_loader返回frozen模块的spec，这个spec的loader还是cls本身。因此，该类也实现了Loader的create_module和exec_module:

```python
    @staticmethod
    def create_module(spec):
        """Use default semantics for module creation."""

    @staticmethod
    def exec_module(module):
        name = module.__spec__.name
        if not _imp.is_frozen(name):
            raise ImportError('{!r} is not a frozen module'.format(name),
                              name=name)
        code = _call_with_frames_removed(_imp.get_frozen_object, name)
        exec(code, module.__dict__)
```

和builtin模块不一样，frozen模块用python实现，create_module不用返回值，用默认的创建模块方式创建即可。同样的，exec_module也是通过python源码生成code对象来exec的。

另外由于frozen模块的特殊性，该loader还额外实现了get_code和is_package方法：

```python
    @classmethod
    @_requires_frozen
    def get_code(cls, fullname):
        """Return the code object for the frozen module."""
        return _imp.get_frozen_object(fullname)

    @classmethod
    @_requires_frozen
    def get_source(cls, fullname):
        """Return None as frozen modules do not have source code."""
        return None

    @classmethod
    @_requires_frozen
    def is_package(cls, fullname):
        """Return True if the frozen module is a package."""
        return _imp.is_frozen_package(fullname)
```

### PathFinder

PathFinder的工作流程是这3个里面最复杂的，它只是external模块的Finder，不充当Loader的角色，它的find_spec流程也很复杂，篇幅限制不在此处贴(后面会贴简化代码)，大致上是做了下面的这些事。

首先如果path是None，就把path设为sys.path，也就是从PYTHONPATH里面找想要的模块，符合认知。

接着遍历path，根据每个path生成FileFinder。FileFinder才是真正返回ModuleSpec的对象。FileFinder会被存到`sys.path_importer_cache`中，这个对象是一个dict，key为搜索路径，value为FileFinder对象。在生成FileFinder的过程中，会去遍历`sys.path_hooks`找到合适的FileFinder。

找到FileFinder后，调用FileFinder的find_spec得到ModuleSpec，并返回。

简化后的find_spec代码如下：

```python
    @classmethod
    def find_spec(cls, fullname, path=None, target=None):
        if path is None:
            path = sys.path
        spec = None
        namespace_path = []
        for entry in path:
            file_finder = sys.path_importer_cache.get(entry)
            if file_finder is None:
                for hook in sys.path_hooks:
                    try:
                        file_finder = hook(entry)
                        break
                    except ImportError:
                        continue
                sys.path_importer_cache[entry] = file_finder
            if file_finder is None:
                continue
            spec = file_finder.find_spec(fullname, target)
            if spec is None:
                continue
            if spec.loader is not None:
                break
            namespace_path.extend(spec.submodule_search_locations)
        else:
            if namespace_path:
                spec = ModuleSpec(fullname, None)
                spec.origin = None
                spec.submodule_search_locations = namespace_path
                return spec
            return None
        return spec
```

#### FileFinder

当某个具体的目录被分发到FileFinder后，就会调用FileFinder的find_spec函数来找到最终的ModuleSpec。逻辑很简单，其实就是找目录或者找文件，找到了就调spec_from_file_location返回对应的spec。简化后的代码如下：

```python
class FileFinder:
    def __init__(self, path, *loader_details):
        """
        Args:
            path: [str]寻找文件的目录路径
            loader_details: [tuple]每一项是一个二元组(loader_class, suffixes)即Loader的类支持哪些suffixes拓展名列表
        """
        loaders = []
        for loader_class, suffixes in loader_details:
            loaders.extend((suffix, loader_class) for suffix in suffixes)
        self._loaders = loaders
        self.path = path or '.'
        if not os.path.isabs(self.path):
            self.path = os.path.join(os.getcwd(), self.path)
        self._path_cache = set(os.listdir(self.path))

    def find_spec(self, fullname, target=None):
        tail_module = fullname.rpartition('.')[2]
        # Check if the module is the name of a directory (and thus a package).
        if tail_module in self._path_cache:
            base_path = os.path.join(self.path, tail_module)
            for suffix, loader_class in self._loaders:
                init_filename = '__init__' + suffix
                full_path = os.path.join(base_path, init_filename)
                if os.path.isfile(full_path):
                    return self._get_spec(loader_class, fullname, full_path, [base_path], target)
        # Check for a file w/ a proper suffix exists.
        for suffix, loader_class in self._loaders:
            try:
                full_path = os.path.join(self.path, tail_module + suffix)
            except ValueError:
                return None
            if tail_module + suffix in self._path_cache:
                if os.path.isfile(full_path):
                    return self._get_spec(loader_class, fullname, full_path, None, target)
        return None

    def _get_spec(self, loader_class, fullname, path, submodule_search_locations, target):
        loader = loader_class(fullname, path)
        return spec_from_file_location(fullname, path, loader=loader, submodule_search_locations=submodule_search_locations)
```

## `sys.path_hooks`

是一个list, 每一项是一个`Callable[[str], FileFinder]`，也就是一个可调用对象，调用时会传入路径，并返回FileFinder对象。

PathFinder的find_spec函数会遍历这个对象获取FileFinder，详见PathFinder的find_spec函数。

## Finder

Finder是`sys.meta_path`列表中的对象，理论上来说只需要一个find_spec函数即可，find_spec函数返回一个ModuleSpec，不过如果为了兼容老版本的Python(调用find_module函数返回loader)，可以继承`importlib.abc.MetaPathFinder`类来做，这个基类包含了兼容老版本Python的功能。

```python
from importlib import abc, machinery


class MyMetaPathFinder(abc.MetaPathFinder):
    def find_spec(self, fullname, path_list, target=None) -> machinery.ModuleSpec | None:
        """
        实现自己的MetaPathFinder
        返回None表示这个MetaPathFinder找不到, 交给下一个sys.meta_path中的MetaPathFinder继续找
        Args:
            fullname: [str]模块完整路径
            path_list: [list]查找模块的路径列表
            target: [types.ModuleType]reload时有值，是原本的module对象
        Returns:
            importlib.machinery.ModuleSpec
        """
```

## ModuleSpec

ModuleSpec是import过程中非常重要的一个中间对象，描述了模块的相关属性，是`importlib.machinery.ModuleSpec`的实例。

相关的属性有：

- name: 模块的名字
- loader: 模块的loader对象
- origin: 模块文件的绝对路径，也可能是`'builtin'` / `'frozen'`
- submodule_search_locations: 子模块搜索路径列表，构造时传入is_package=True才会是list否则是None
- parent: 只读，父模块名
- has_location: 模块是否对应一个操作系统的文件

## Loader

Loader是import过程中加载模块的中间对象，用于加载模块，实际上只需要实现create_module和exec_module这2个函数即可。不过如果为了兼容老版本的Python(调用load_module返回module实例)，可以继承`importlib.abc.Loader`类来做，这个基类包含了兼容老版本Python的功能。而且如果module就是types.ModuleType的实例，甚至可以不实现create_module，因为这个基类已经实现了这个函数返回None，只需要实现exec_module即可:

```python
from importlib import abc

class MyModuleLoader(abc.Loader):
    def exec_module(self, module) -> None:
        """
        在这个函数里设置module的全局变量
        Args:
            module: module实例
        Returns:
            None
        """
```

`importlib._bootstrap_external`这个模块实现了3个基础Loader:

- py文件Loader: SourceFileLoader
- pyc文件Loader: SourcelessFileLoader
- pyd文件Loader: ExtensionFileLoader

下面分别看下这3个Loader的实现。

### SourceFileLoader

### SourceLessFileLoader

### ExtensionFileLoader
