C++函数定义：

```cpp
#include <emscripten.h>
#include <stdio.h>

extern "C" EMSCRIPTEN_KEEPALIVE void
sayHi()
{
    printf("Hi!\n");
    printf("oh no\n");
    emscripten_log(EM_LOG_ERROR, "const char *format, ...");
}

extern "C" EMSCRIPTEN_KEEPALIVE int daysInWeek()
{
    return 7;
}
```

注意必须要有`extern "C"`。

然后在编译参数增加下面2个参数：

```cmake
target_link_libraries(Tamashii "-s MODULARIZE=1")
target_link_libraries(Tamashii "-s EXPORT_NAME=\"initializeTamashiiJs\"")
```

node加载(node其实只需要MODULARIZE即可)：

```js
var factory = require('./api_example.js');

factory().then((instance) => {
  instance._sayHi(); // direct calling works
  console.log(instance._daysInWeek()); // values can be returned, etc.
});
```

浏览器加载：

```js
/**
 * 
 * @param {string} source 
 * @returns {Promise<void>}
 */
const loadScript = (source) => {
  return new Promise((resolve, reject) => {
    if (typeof document === 'undefined') {
      reject(
        new Error('loadScript is only supported in a browser environment.')
      );
      return;
    }

    const { body } = document;
    if (!body) {
      reject(
        new Error('loadScript is only supported in a browser environment.')
      );
      return;
    }

    const scriptElement = document.createElement('script');
    scriptElement.type = 'text/javascript';
    scriptElement.src = source;
    scriptElement.onload = () => resolve();
    scriptElement.onerror = error => reject(error);
    scriptElement.onabort = error => reject(error);

    body.appendChild(scriptElement);
  });
};

loadScript('./api_example.js').then(() => {
  const initializeTamashiiJs = global.initializeTamashiiJs;
  console.log(initializeTamashiiJs);
  initializeTamashiiJs().then(instance => {
    console.log(instance);
    console.log(instance._sayHi);
    console.log(instance._daysInWeek);
    instance._sayHi();
    global.tamashii = instance;
    console.log(instance._daysInWeek());
    console.log(instance._test(22))
  }
  )
});
```

## 传递字符串

比如下面这个C++函数：

```cpp
#include <emscripten.h>

extern "C" EMSCRIPTEN_KEEPALIVE void printString(const char* a)
{
    emscripten_log(EM_LOG_INFO, a);
}
```

js调用时需要先把js的string转为`char *`，类似这种调法：

```js
tamashii._printString(tamashii.allocateUTF8("aaabcc"));
```

不过allocateUTF8会在堆空间开辟内存，所以需要在c++里面`#include <malloc.h>`之后调用`free`或者在js里面调用`tamashii._free`。

如果C++函数返回了`char *`那么js也需要讲`char *`转为string:

```js
var a = tamashii.UTF8ToString(tamashii.testString());
```

这几个函数都是[preamble.js](https://emscripten.org/docs/api_reference/preamble.js.html#preamble-js)的函数，默认情况下导出的胶水代码没有这几个函数，如果需要的话要额外增加编译选项：

```cmake
target_link_libraries(Tamashii "-s EXPORTED_FUNCTIONS=['_malloc','_free']")
target_link_libraries(Tamashii "-s EXPORTED_RUNTIME_METHODS=['allocateUTF8','allocateUTF8OnStack','UTF8ToString']")
```

## 传递指针

指针在js里面是number类型，所以可以直接传：

```cpp
class Tac
{
public:
    int a = 1;
    const char* b = "emm";
};

extern "C" EMSCRIPTEN_KEEPALIVE Tac* testTac()
{
    return new Tac {};
}

extern "C" EMSCRIPTEN_KEEPALIVE void printTac(Tac* t)
{
    emscripten_log(EM_LOG_INFO, t->b);
}
```

js:

```js
const tac = tamashii._testTac();
tamashii._printTac(tac);
```
