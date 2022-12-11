[参考文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions)

通常用字面量来创建正则表达式，两个slash中间的部分会被识别为正则表达式，比如：

```js
const letterKeyRe = /^Key[A-Z]$/;
```

字面量创建的正则表达式会在编译时确定，如果正则表达式在运行时才能确定，则可以通过`RegExp`来创建，比如：

```js
const letterKeyRe = new RegExp('^Key[A-Z]$');
```

## flag

JavaScript的正则表达式还可以定义flag，比如下面的正则表达式：

```js
const fooRe = /foo/g;
```

第二个slash后面的`g`就是这个正则表达式的flag。

一个正则表达式还可以定义多个flag，比如：

```js
const fooRe = /foo/gim;
```

通过`RegExp`创建时同样可以定义flag，比如:

```js
const fooRe = new RegExp('foo', 'gim');
```

各flag的意义如下表。

flag | 意义
--- | ---
g | global, 识别所有
i | ignore case, 忽略大小写
s | dot all, .代表所有字符, 没有这个flag时.只能代表除`\n`, `\r` 之外的其他字符
d | has indices, 如果匹配结果是否包含匹配位置的下标
m | multi line, 是否识别多行字符串
y | sticky, 是否从[lastIndex](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/lastIndex)开始识别，而且下一个匹配必须从lastIndex开始
u | unicode, 是否识别unicode, [unicode参考文章](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_Expressions/Unicode_Property_Escapes)

## Character classes

character classes代表一个character, 常用的有:

character | 意义
--- | ---
`.` | 除`\n`, `\r` 之外的其他字符
`\d` | 任何数字, 相当于`[0-9]`
`\D` | `\d`的反集，不是数字的任何character, 相当于`[^0-9]`
`\w` | 任何character, 相当于`[A-Za-z0-9_]`
`\W` | `\w`的反集, 相当于`[^A-Aa-z0-9]`
`\s` | 任何空白, 如空格 制表符之类的
`\S` | `\s`的反集
`\t` | 水平制表符
`\r` | 回车
`\n` | 换行符

## Qualifiers

Qualifiers代表一个character的重复次数, 常用的有:

qualifiers | 意义
--- | ---
`x*` | x的克林闭包，x重复0或任何次
`x+` | x重复n次，n不能为0
`x?` | x重复0或1次
`x{n}` | x重复n次，n为固定次数，比如`x{2}`代表x重复2次
`x{n,}` | x重复n到无穷多次
`x{n,m}` | x重复n到m次

## Groups and ranges

Groups and ranges代表了一组character的集合, 常用的有:

expression | 意义
--- | ---
x\|y | 匹配x或者匹配y
`[xyz]` | 匹配x到z这3个字符的任意一个
`[a-z]` | 匹配a到z这26个字符的任意一个
`[^xyz]` | `[xyz]`的反集
`(x)` | 匹配x并记录当前匹配的值，被下面的一些expression调用
`\1` `\2` `\3` 等等 | 也就是back slash后面加一个数字，是正则表达式的变量，代表从左到右第n个(x)匹配时所记录的值，比如用`/apple(,)\sorange\1`匹配"apple, orange, cherry, peach"时会得到"apple, orange,"，第1个(x)匹配的值是,所以\\1是,
`(?<Name>x)` | `(x)`的高级用法，将`(x)`匹配的值存在名为`Name`的变量中，比如用`/(?\<title>\w+)/匹配"abc"时，匹配结果是"abc"，此时匹配过程中就有了一个中间变量，变量名为title, 值为"abc"
`\k<Name>` | back slash后面加数字的高级用法，也就是直接用变量名来取前面匹配的值，比如`/(?<title>\w+), yes \k<title>/`匹配"Do you copy? Sir, yes Sir!"时匹配结果为"Sir, yes Sir"

## Assertion

Assertion用来表示边界，常用的有:

assertion | 意义
--- | ---
`^` | 匹配字符串开头，multi line模式也匹配line break的后面
`$` | 匹配字符串结尾, multi line模式也匹配line break的前面
`\b` | 匹配单词边界，比如`/\bm\w/`在匹配"momn"时结果为"mo"而不是"mn", 因为"momn"被认为是一个完整的单词，单词边界既可以是左边也可以是右边
`\B` | `\b`的反集，即不是单词的边界，比如`/\Bon/`匹配"noon"时结果为"on"
`x(?=y)` | x的后面必须紧跟着y才匹配x，否则不匹配x
`x(?!y)` | `x(?=y)`的反集，x的后面紧跟着的必须不是y才匹配x
`(?<=y)x` | x的前面必须是y才匹配x，否则不匹配x
`(?<!y)x` | `(?<=y)x`的反集合，x的前面必须不是y才匹配x
