npm是node package management的意思，是node的包管理软件，随node一起安装。

## package.json

package.json是npm的包描述，用这个文件来描述当前包的相关信息，比如自己写的开源包，就可以在根目录下面写一个package.json。

package.json也可以通过node生成：

```sh
npm init
```

敲下这个命令后会一步一步输入package.json里的配置信息，如果不想输入想直接用默认值，可以用下面这个命令：

```shell
npm init --yes
```

package.json初始长这个样子：

```json
{
    "name": "YourAwesomeProjectName",
    "version": "1.0.0",
    "description": "Your project description",
    "main": "src/main.js",
    "author": "Your Name",
    "keywords": [],
    "repositry": {
        "type": "git",
        "url": "https://github.com/npm/cli.git"
    },
    "license": "MIT",
    "scripts": {
    }
}
```

各字段都比较好理解，详情可以参考[package.json官方文档](https://docs.npmjs.com/cli/v7/configuring-npm/package-json)。其中scripts字段可以参考[scripts文档](https://docs.npmjs.com/cli/v7/using-npm/scripts)。

## 安装包

当前工作目录安装包：

```shell
npm install vue
```

全局安装包：

```shell
npm install vue -g
```

安装包并写入package.json的dependencies:

```shell
npm install vue --save
```

安装包并写入package.json的devDependencies:

```shell
npm install vue --save-dev
```

工作目录安装时会安装到`node_modules`目录并自动生成一个`package-lock.json`文件。

写入package.json的好处是`node_modules`目录可以加到gitignore里，其他电脑git clone这个包之后直接用下面这个命令就可以安装对应的包

```shell
npm install
```

## 卸载包

当前目录卸载：

```shell
npm uninstall vue
```

全局卸载：

```shell
npm uninstall vue -g
```

## 更新包

更新工作目录下全部依赖：

```shell
npm update
```

更新指定包：

```shell
npm update vue
```

更新到指定版本：

```shell
npm update packagename@x.y.z
```
