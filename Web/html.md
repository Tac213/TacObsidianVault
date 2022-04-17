在vscode，安装live server即可实时通过浏览器预览html, 具体使用方法看live server主页即可，非常简单。

## 注释

```html
<!-- Comment -->
<!-- Multiline
    Comment-->
```

## 基本结构

```html
<!-- 下面这个表示这个是一个html5文档 -->
<!DOCTYPE html>
<html>
    <head>
		<meta charset="UTF-8" />
		<meta name="viewport" content="width=device-width, initial-scale=1.0" />
        <title>Tac Title</title>
    </head>
    <body>
        Mmmmmmmmmmmmmmnnn
    </body>
</html>
```

用tag标识的东西教element。tag就是标识符。

html大小写不敏感。

## 常用标识符

```html
<!-- 段落标识符 -->
<p>这是一个段落</p>

<!-- 预留格式, 这个里面可以写一些带格式的文字 -->
<pre>
后面的这个换行会被识别
后面的这个空格 也会被识别
</pre>

<!-- 标题字, 从小到大, 类似markdown的#，预设最多到h6 -->
<h1>标题</h1>
<h2>标题</h2>
<h3>标题</h3>
<h4>标题</h4>
<h5>标题</h5>
<h6>标题</h6>

<!-- 设置字体颜色 -->
<font color="red" size="50">设置字体颜色</font>

<!-- 换行符, 独目标记 -->
<br>

<!-- 分割线 -->
<hr>

<!-- 有颜色的分割线 -->
<hr color="red">

<!-- bold text -->
<b>加粗文字</b>

<!-- bold text -->
<strong>加粗文字strong</strong>

<!-- italic text -->
<i>斜体文字</i>

<!-- italic text -->
<em>斜体文字em</em>

<!-- small text -->
<small>小字</small>

<!-- insert text -->
<ins>插入字, 有下划线</ins>

<!-- delete text -->
<del>删除字, 中间有横线</del>

<!-- mark text -->
<mark>标记文字</mark>

<!-- subscirpt -->
a<sub>3</sub>是a的下标3
<!-- superscript -->

2<sup>3</sup>是2的三次方
```

## 转义

如果需要在文本里写<或者>，需要进行转义，<是`&lt;`，>是`&gt;`，在html里叫实体符号，以&开头以;结尾

- \<: `&lt;`
- \>: `&gt;`
-  (空格): `&nbsp;`
- utf8字符(emoji也可以): `&#128516;`

## html引用其他文字

```html
<!-- block quote, browsers intent these element -->
<blockquote>我的前面有缩进</blockquote>

<!-- quote, q表示简单的引用, 两端加双引号-->
<p>WWF's goal is to: <q>Build a future where people live in harmony with nature.</q></p>
<!-- Abbreviation, 下标虚线, 鼠标移动有title对应的Tooltip -->
<p>The <abbr title="World Health Organization">WHO</abbr> was founded in 1948.</p>
<!-- 地址信息引用, 斜体 -->
<address>
Written by John Doe.<br>
Visit us at:<br>
Example.com<br>
Box 564, Disneyland<br>
USA
</address>
<!-- The HTML <cite> tag defines the title of a creative work (e.g. a book, a poem, a song, a movie, a painting, a sculpture, etc.). -->
<!-- 其实就是斜体 -->
<p><cite>The Scream</cite> by Edvard Munch. Painted in 1893.</p>
<!-- This text will be written from right to left-->
<bdo dir="rtl">rtl会让这个文字从右到左显示</bdo>
```

## html链接基本样式

```html
<!-- link, 超链接 -->
<a href="https://www.w3schools.com/">Visit W3Schools.com!</a>
```

链接可以配置的属性：`target`可以配置链接在哪个标签页打开, `_self`就是在当前页, `_blank`就是新建一个标签页打开链接, 链接里面可以写绝对路径也可以写相对路径，也可以使用按钮、图片链接，详情参考[这里](https://www.w3schools.com/html/html_links.asp)。

通过下面这样给标题加个id可以设定一个bookmark书签：

```html
<h2 id="C4">Chapter 4</h2>
```

有了bookmark就可以通过url定位到这个书签所在的位置：

```html
<a href="html_demo.html#C4">Jump to Chapter 4</a>
```

## 图片

```html
<!-- 设置宽高时只设置一个就可以等比缩放 -->
<img src="path/to/image" width="100px" title="图片悬停显示" alt="图片加载失败显示文字"/>
```

## 列表

无序列表：

```html
<ul>
    <li>a</li>
    <li>b</li>
    <li>c</li>
</ul>
```

效果：

- a
- b
- c

有序列表：

```html
<ol>
    <li>a</li>
    <li>b</li>
    <li>c</li>
</ol>
```


效果：

1. a
2. b
3. c

## 表单

就是类似qt里面的QFromLayout。

可以在表单里套一个table，让表单看起来比较整齐

```html
<form action="http://localhost:8080/register">
    <table>
        <tr>
            <td>user name</td>
            <td><input type="text" name="username"></td>
        </tr>
        <tr>
            <td>pass word</td>
            <td><input type="password" name="password"></td>
        </tr>
        <tr>
            <td>confirm pass word</td>
            <td><input type="password"></td>
        </tr>
        <tr>
            <td>gender</td>
            <td>
                <input type="radio" name="gender" value="0" checked>male
                <input type="radio" name="gender" value="1">female
            </td>
        </tr>
        <tr>
            <td>multi selection</td>
            <td>
                <input type="checkbox" name="multiselection" value="selection1" checked>selection1
                <br>
                <input type="checkbox" name="multiselection" value="selection2">selection2
                <br>
                <input type="checkbox" name="multiselection" value="selection3">selection3
                <br>
            </td>
            </td>
        </tr>
        <tr>
            <td>drop down selection</td>
            <td>
                <select name="dropdown">
                    <option value="a">a</option>
                    <option value="b" selected>b</option>
                    <option value="c">c</option>
                </select>
            </td>
        </tr>
        <tr>
            <td>drop down multi selection</td>
            <td>
                <select name="dropdown" multiple="multiple">
                    <option value="a">a</option>
                    <option value="b" selected>b</option>
                    <option value="c">c</option>
                </select>
            </td>
        </tr>
        <tr>
            <td>select file</td>
            <td>
                <input type="file" name="select file">
            </td>
        </tr>
        <tr>
            <td>indroduction</td>
            <td>
                <textarea rows="10" cols="60" name="indroduction"></textarea>
            </td>
        </tr>
        <tr>
            <td rowspan="2">
                <input type="submit" value="click me to submit">
            </td>
        </tr>
        <tr>
            <td rowspan="2">
                <input type="reset" value="click me to reset">
            </td>
        </tr>
    </table>
</form>
```

点击`<input type="submit">`按钮会提交表单数据，标签的name属性作为字段名，标签当前输入的内容作为字段的值，比如提交的url:

```
http://localhost:8080/register?username=aaa&password=222&gender=0&multiselection=selection1&multiselection=selection2&dropdown=b&indroduction=asdfasddddddddddddddddddddddddddddddddd
```

http协议规定这样了格式：前面是`<form>`标签的action属性指定的接受表单数据的服务器地址，后面接个问号`?`，然后就是`name=value&name=value&name=value`。

默认情况下会在浏览器的地址栏显示表单所提交的数据，但是如果表单中有密码等敏感数据则不是很安全，此时可以给form指定`method="post"`属性就不会显示出来，但是服务器仍然会收到上述的数据。默认情况下`method="get"`，会显示表单提交的数据。

表单中可以通过下面的方式增加一个对用户不可见的字段：

```html
<form>
    <input type="hidden" name="userid" value="1000">
</form>
```

也可以给input标签增加readonly属性使字段只读不能编辑。

如果给input标签增加disabled属性也是只读的，但是字段不会被提交。
