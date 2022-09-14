要使用embind的功能的话需要在编译时增加`-lembind`的编译选项：

```cmake
target_link_libraries(${TARGET_NAME} "-lembind")
```

如果从static library导出，那么编译命令如下:

```bash
emcc -lembind -o library.js -Wl,--whole-archive library.a -Wl,--no-whole-archive
```

## 导出函数

```cpp
#include <emscripten/bind.h>

using namespace emscripten;

float lerp(float a, float b, float t) {
    return (1 - t) * a + t * b;
}

EMSCRIPTEN_BINDINGS(my_module) {
    function("lerp", &lerp);
}
```

返回值类型和参数类型会自动识别。

js加载模块后这样调用即可：

```js
var result = Module.lerp(1, 2, 3);
```

### 函数重载

overload的函数可以通过下面的方法导出：

```cpp
#include <emscripten/bind.h>

struct HasOverloadedMethods {
    void foo();
    void foo(int i);
    void foo(float f) const;
};

EMSCRIPTEN_BINDING(overloads) {
    class_<HasOverloadedMethods>("HasOverloadedMethods")
        .function("foo", select_overload<void()>(&HasOverloadedMethods::foo))
        .function("foo_int", select_overload<void(int)>(&HasOverloadedMethods::foo))
        .function("foo_float", select_overload<void(float)const>(&HasOverloadedMethods::foo))
        ;
}
```

## 导出类

```cpp
#include <emscripten/bind.h>

class MyClass {
public:
  MyClass(int x, std::string y)
    : x(x)
    , y(y)
  {}

  void incrementX() {
    ++x;
  }

  int getX() const { return x; }
  void setX(int x_) { x = x_; }

  static std::string getStringFromInstance(const MyClass& instance) {
    return instance.y;
  }

private:
  int x;
  std::string y;
};

// Binding code
EMSCRIPTEN_BINDINGS(my_class_example) {
  class_<MyClass>("MyClass")
    .constructor<int, std::string>()
    .function("incrementX", &MyClass::incrementX)
    .property("x", &MyClass::getX, &MyClass::setX)
    .class_function("getStringFromInstance", &MyClass::getStringFromInstance)
    ;
}
```

`class_`这个标识符是`emscripten/bind.h`中实现的一个类，后面调用的`constructor`这种方法都会返回`*this`所以支持链式调用。struct也可以通过`class_`导出。

- constructor: 导出类的构造函数，尖括号中写参数类型
- function: 导出成员方法，还可以多传若干个`Policy`参数，比如`allow_raw_pointers()`  `pure_virtual()`之类的
- property: 导出成员属性，`property("x", &MyClass::getX, &MyClass::setX)`导出的是可读可写的属性，`property("x", &MyClass::getX)`导出的是只读属性，`property("x", &MyClass::x)`导出的也是可读可写(如果属性是const则只读)但是读写逻辑无法定义；
- class_function: 导出静态方法，需要通过类调用对应的方法，同样可以多传`Policy`参数；
- class_property: 导出静态属性，只能通过`class_property("y", &MyClass::y)`来导出；

### 使用智能指针构造函数

```cpp
#include <emscripten/bind.h>

EMSCRIPTEN_BINDINGS(my_class_example)
{
    class_<MyClass>("MyClass")
        .smart_ptr_constructor("MyClass", &std::make_shared<MyClass, int, std::string>);
}
```

注意暂时不能包含多个构造函数。

### 允许subclass的类

```cpp
#include <emscripten/bind.h>

class MyClassWrapper : public wrapper<MyClass>
{
public:
    EMSCRIPTEN_WRAPPER(MyClassWrapper);
};

EMSCRIPTEN_BINDINGS(my_class_example)
{
    class_<MyClass>("MyClass")
        .allow_subclass<MyClassWrapper, int, std::string>("MyClassWrapper", constructor<int, std::string>());
}
```

如果使用无参构造函数，则`allow_subclass`的第二个参数可以不传。

`EMSCRIPTEN_WRAPPER`宏的作用是给Wrapper类定义构造函数，宏定义如下：

```cpp
#define EMSCRIPTEN_WRAPPER(T)                                           \
template<typename... Args>                                          \
T(::emscripten::val&& v, Args&&... args)                            \
    : wrapper(std::forward<::emscripten::val>(v), std::forward<Args>(args)...) \
{}
```

注意可以被subclass调用的构造函数必须`public`，所以前面还要加`public`，struct默认public则不用加。

subclass的方法如下：

```js
var DerivedClass = Module.MyClass.extend("DerivedClass", {  // the first parameter is the name of subclass
    // __construct and __destruct are optional.  They are included
    // in this example for illustration purposes.
    // If you override __construct or __destruct, don't forget to
    // call the parent implementation!
    __construct: function(x, y) {
        this.__parent.__construct.call(this, x, y);
        this.z = false;
    },
    __destruct: function() {
        this.__parent.__destruct.call(this);
    },
});

var subclassInstance = new DerivedClass(1, "emm");
```

也可以通过下面的方法实现虚函数：

```js
var x = {
    invoke: function(str) {
        console.log('invoking with: ' + str);
    }
};
var interfaceObject = Module.Interface.implement(x);
```

带有纯虚函数的类导出：

```cpp
struct Interface {
    virtual void invoke(const std::string& str) = 0;
};

struct InterfaceWrapper : public wrapper<Interface> {
    EMSCRIPTEN_WRAPPER(InterfaceWrapper);
    void invoke(const std::string& str) {
        return call<void>("invoke", str);
    }
};

EMSCRIPTEN_BINDINGS(interface) {
    class_<Interface>("Interface")
        .function("invoke", &Interface::invoke, pure_virtual())
        .allow_subclass<InterfaceWrapper>("InterfaceWrapper")
        ;
}
```

带有虚函数的类导出：

```cpp
struct Base {
    virtual void invoke(const std::string& str) {
        // default implementation
    }
};

struct BaseWrapper : public wrapper<Base> {
    EMSCRIPTEN_WRAPPER(BaseWrapper);
    void invoke(const std::string& str) {
        return call<void>("invoke", str);
    }
};

EMSCRIPTEN_BINDINGS(interface) {
    class_<Base>("Base")
        .allow_subclass<BaseWrapper>("BaseWrapper")
        .function("invoke", optional_override([](Base& self, const std::string& str) {
            return self.Base::invoke(str);
        }))
        ;
}
```

`optional_override`是必须的。

带有基类的类导出：

```cpp
EMSCRIPTEN_BINDINGS(base_example) {
    class_<BaseClass>("BaseClass");
    class_<DerivedClass, base<BaseClass>>("DerivedClass");
}
```

### subclass注意事项

首先js必须显示删除cpp对象，通过下面2个函数的任意一个完成：

```js
subclassInstance.delete();
subclassInstance.deleteLater();
```

subclass的实例可以clone，clone造成引用计数增加，不会复制cpp对象

```js
async function myLongRunningProcess(x, milliseconds) {
    // sleep for the specified number of milliseconds
    await new Promise(resolve => setTimeout(resolve, milliseconds));
    x.method();
    x.delete();
}

const y = new Module.MyClass;          // refCount = 1
myLongRunningProcess(y.clone(), 5000); // refCount = 2
myLongRunningProcess(y.clone(), 3000); // refCount = 3
y.delete();                            // refCount = 2

// (after 3000ms) refCount = 1
// (after 5000ms) refCount = 0 -> object is deleted
```

通过`isAliasOf`方法可以知道某个对象是不是另一个对象的clone:

```js
a.isAliasOf(b);
```

该函数实现如下：

```js
function ClassHandle_isAliasOf(other) {
  if (!(this instanceof ClassHandle)) {
    return false;
  }
  if (!(other instanceof ClassHandle)) {
    return false;
  }

  var leftClass = this.$$.ptrType.registeredClass;
  var left = this.$$.ptr;
  var rightClass = other.$$.ptrType.registeredClass;
  var right = other.$$.ptr;

  while (leftClass.baseClass) {
    left = leftClass.upcast(left);
    leftClass = leftClass.baseClass;
  }

  while (rightClass.baseClass) {
    right = rightClass.upcast(right);
    rightClass = rightClass.baseClass;
  }

  return leftClass === rightClass && left === right;
}
```

通过`$$`变量可以拿到subclass实例的一些内部信息：

- count: 引用计数
- prt: cpp对象指针地址
- smartPrt: cpp对象智能指针地址
- ptrType.registeredClass: subclass实例对应的RegisteredClass，可以调`downcast`  `upcast`等函数进行cast操作

## Enum导出

```cpp
enum OldStyle {
    OLD_STYLE_ONE,
    OLD_STYLE_TWO
};

enum class NewStyle {
    ONE,
    TWO
};

EMSCRIPTEN_BINDINGS(my_enum_example) {
    enum_<OldStyle>("OldStyle")
        .value("ONE", OLD_STYLE_ONE)
        .value("TWO", OLD_STYLE_TWO)
        ;
    enum_<NewStyle>("NewStyle")
        .value("ONE", NewStyle::ONE)
        .value("TWO", NewStyle::TWO)
        ;
}
```

## const导出

```cpp
EMSCRIPTEN_BINDINGS(my_constant_example) {
    constant("SOME_CONSTANT", SOME_CONSTANT);
}
```
