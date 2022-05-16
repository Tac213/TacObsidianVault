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
`decode_source(bytes_source)`: 传入bytes类型的源代码，decode为str类型，bytes类型的源代码可以通过`io.open_code(file_path)`或者`open(file_path, 'rb')`得到
`SourceFileLoader(name, path)`: 可以加载`SOURCE_SUFFIXES`的Loader类
`SourcelessFileLoader(name, path)`: 可以加载`BYTECODE_SUFFIXES`的Loader类
`ExtensionFileLoader(name, path)`: 可以加载`SOURCE_SUFFIXES`的Loader类
`PathFinder`: 普通Python模块的Finder

### 上述变量、函数、类的常规import方式

```python
from importlib.machinery import ModuleSpec, BuiltinImporter, FrozenImporter, PathFinder, FileFinder, SourceFilerLoader, SourcelessFileLoader, ExtensionFileLoader, SOURCE_SUFFIXES, BYTECODE_SUFFIXES, EXTENSION_SUFFIXES, all_suffixes

from importlib.util import module_from_spec, spec_from_loader, spec_from_file_location, decode_source
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

这3个loader的基类实现了一些必要函数，例如：

- `exec_module(self, spec)`: 调用`get_code`函数拿到module的code对象，调用exec将code内容写入module的变量命名空间(调用方法为`exec(code, module.__dict__)`)
- `is_package(self, fullname)`: 根据模块的绝对路径尾部是否带有__init__来判断模块是不是package
- `get_filename(self, fullname)`: 返回模块的绝对路径
- `path_stats(self, path)`: 返回字典`{'mtime': stats_mtime, 'size': stats_size}`
- `get_data(self, path)`: 返回文件中的bytes数据
- `set_data(self, path, data)`: 将某些数据写入某个文件中
- `get_source(self, fullname)`: 调用get_data得到bytes数据，然后将bytes数据传入到`decode_source`中得到bytes对应的string代码
- `source_to_code(self, data, path)`: 将bytes源码编译为code对象(调用方法为`compile(bytes_data, path, 'exec', dont_inherit=True, optimize=-1)`)
- `get_code(self, fullname)`: 核心函数，获取模块的code对象

其中py和pyd文件都是通过`io.open_code(path)`来拿到bytes数据的，`io.open_code(path)`等效于`open(path, 'rb')`，而pyc文件是通过`io.FileIO(path, 'r')`来拿bytes数据的。

下面分别看下这3个Loader的实现。

### SourceFileLoader

这个loader没有override exec_module，所以在get_code函数中实现了所有加载逻辑。

如果熟悉Python都会直到，源py会被cache到pyc文件中，所以要了解这个Loader，需要先了解pyc中存储的字节数据。

#### pyc中的字节数据

pyc中存储的是字节码，前16个字节是pyc文件的header，后面的所有字节才是源文件中的所有信息。

源文件的所有信息是通过下面这种方式得到的：

```python
import io
import marshal

with io.open_code(source_py_file_path) as py_src_file:
    source_bytes = py_src_file.read()

source_code = compile(source_bytes, source_py_file_path, 'exec', dont_inherit=True, optimize=-1)
return marshal.dumps(code)
```

存在2种pyc文件，一种是带有源文件hash信息的pyc文件，另一种是带有源文件时间戳的pyc文件。

带有源文件hash信息的pyc文件的意义是限定了：在使用pyc时，这个pyc文件必须时被当前源py文件编译出来的才能用这个pyc。

带有源文件hash信息的pyc文件是这样dump的:

```python
def _code_to_hash_pyc(code, source_hash, checked=True):
    "Produce the data for a hash-based pyc."
    data = bytearray(MAGIC_NUMBER)
    flags = 0b1 | checked << 1
    data.extend(_pack_uint32(flags))
    assert len(source_hash) == 8
    data.extend(source_hash)
    data.extend(marshal.dumps(code))
    return data
```

由此可见，前16字节header信息为：

- 1 - 4字节：MAGIC_NUMBER，每个Python版本都有一个MAGIC_NUMBER，可以通过`sys.modules['importlib._bootstrap_external'].MAGIC_NUMBER`这个表达式获取，通过`(3429).to_bytes(2, 'little') + b'\r\n'`这种方式定义，对于3.x.y版本的Python，如果x相同，则MAGIC_NUMBER是相同的，pyc文件互通，否则不行；
- 4 - 8字节: 是一个flags标签，在解析文件的时候用到；
- 9 - 16字节: 源文件的哈希值

flags标签目前只有2位有效信息，第一位表示是否使用hash信息的pyc，第二位表示是否check_source。在读取pyc数据时，如果4 - 8位的信息超过2位（第3位开始有存在不是0的位），就会转而使用时间戳pyc。

其中源文件的哈希值是通过下面这种方式得到的：

```python
import _imp
import io

with io.open_code(source_py_file_path) as py_src_file:
    source_bytes = py_src_file.read()

_imp.source_hash(source_bytes)
```

带有时间戳信息的pyc文件是这样dump的：

```python
def _code_to_timestamp_pyc(code, mtime=0, source_size=0):
    "Produce the data for a timestamp-based pyc."
    data = bytearray(MAGIC_NUMBER)
    data.extend(_pack_uint32(0))
    data.extend(_pack_uint32(mtime))
    data.extend(_pack_uint32(source_size))
    data.extend(marshal.dumps(code))
    return data
```

由此可见，前16字节header信息为：

- 1 - 4字节：MAGIC_NUMBER
- 4 - 8字节：flags全是0
- 9 - 12字节：源文件时间戳
- 13 - 16字节：源文件大小

那么什么时候使用带有源文件hash信息的pyc文件，什么时候使用时间戳的pyc文件呢？`_imp`模块有一个`_imp.check_hash_based_pycs`，其值是一个string，有default / never / always 3种值，默认是default。always就会一直使用源文件hash信息的pyc，never就会一直使用时间戳的pyc文件，default的话，如果第2字节的第3位开始有存在不是0的位，就会使用时间戳pyc，否则就会根据pyc文件第2个字节第2位的数据的值，如果第2个字节第2位是1，则也会使用hash信息的pyc文件，去检测这个pyc文件是否为当前源文件编译出来的py文件。

不过在python3.10的`importlib._bootstrap_external`模块中，SourceFileLoader的get_code函数局部变量hash_based被设为了True，也就是说目前暂时都不会用到hash_based_pycs。

#### 源py文件load流程

首先调用`cache_from_source`拿到pyc文件的路径。

接着，调用`get_data`获取pyc文件中的字节数据。

如果没有pyc文件数据，即文件不存在，则调用`get_data`获取py文件种的字节数据，然后调用`source_to_code`得到code对象，如果`sys.dont_write_bytecode`不是True的话，就会调用`_code_to_xxx_pyc`得到pyc的字节数据，调用`set_data`将pyc字节数据写入。写入pyc的bytes数据时，是`_write_atomic`函数通过`io.FileIO(path, 'wb')`来写入的。

如果有pyc数据，即文件存在，则根据pyc文件类型调用`_validate_xxx_pyc`函数检测当前pyc是否有效(比如时间戳pyc就是pyc时间戳和py时间戳不相等就invalid)，如果无效pyc就走没有pyc的流程，如果pyc有效就调`_complie_bytecode`得到字节码对应的code对象。

字节码对应的code对象是通过下面这种方法得到的：

```python
import marshal
import io
import _imp

with io.FileIO(pyc_path, 'r') as pyc_file:
    data = pyc_file.read()

bytes_data = memoryview(data)[16:]
code = marshal.loads(bytes_data)
if isinstance(code, _code_type):  # _code_type是任意一个python函数的__code__字段的值
    if source_path is not None:
        _imp._fix_co_filename(code, source_path)
    return code
```

去掉了hash pyc逻辑的简化代码如下：

```python
class SourceFileLoader:
    def get_code(self, fullname):
        source_path = self.get_filename(fullname)
        source_mtime = None
        source_bytes = None
        try:
            bytecode_path = cache_from_source(source_path)
        except NotImplementedError:
            bytecode_path = None
        else:
            try:
                st = self.path_stats(source_path)
            except OSError:
                pass
            else:
                source_mtime = int(st['mtime'])
                try:
                    data = self.get_data(bytecode_path)
                except OSError:
                    pass
                else:
                    exc_details = {
                        'name': fullname,
                        'path': bytecode_path,
                    }
                    try:
                        bytes_data = memoryview(data)[16:]
                        _validate_timestamp_pyc(data, source_mtime, st['size'], fullname, exc_details)
                    except (ImportError, EOFError):
                        pass
                    else:
                        return _compile_bytecode(
                            bytes_data,
                            name=fullname,
                            bytecode_path=bytecode_path,
                            source_path=source_path
                        )
        if source_bytes is None:
            source_bytes = self.get_data(source_path)
        code_object = self.source_to_code(source_bytes, source_path)
        if not sys.dont_write_bytecode and bytecode_path is not None and source_mtime is not None:
            data = _code_to_timestamp_pyc(code_object, source_mtime, len(source_bytes))
            self._cache_bytecode(source_path, bytecode_path, data)
        return code_object
```

### SourceLessFileLoader

这个相对于SourceFileLoader就简单很多了，首先因为没有源码，所以override了`get_source`函数返回None。

`get_code`函数是实现也很简单，调用`_classify_pyc`判断pyc文件的合法性，然后还是调用`_compile_bytecode`来得到code对象。

### ExtensionFileLoader

这个Loader要加载pyd，只能使用`_imp`模块中的c函数来完成，简化后的代码如下面所示：

```python
class ExtensionFileLoader:
    def create_module(self, spec):
        return _imp.create_dynamic(spec)

    def exec_module(self, module):
        _imp.exec_dynamic(module)

    def is_package(self, fullname):
        file_name = os.path.basename(self.path)
        return any(file_name == '__init__' + suffix for suffix in EXTENSION_SUFFIXES)

    def get_code(self, fullname):
        return None

    def get_filename(self, fullname):
        return None
```
