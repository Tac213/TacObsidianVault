[参考文章](https://juejin.cn/post/6990929456382607374)

ESLint作为**代码规范检查工具**来使用，比如是否应该使用console，应该使用单引号还是双引号，能否定义未使用的变量等等。

Prettier作为**代码格式化工具**来使用，比如每行代码超过120个字符时自动格式化代码，帮助缩进整齐、统一缩进等等。

使用这2个工具之前都需要`npm init`当前项目，因为这2个工具都是npm的包，并且应将其安装为devDependencies。

## ESLint

安装ESLint可以全局安装也可以将其安装为devDependencies，比如全局安装：

```shell
npm install eslint@latest -g
```

安装完成后，在当前项目根目录执行eslint初始化：

```shell
npx eslint --init
```

接着会出现很多选项，比如:

- How would you like to use ESLint? · problems
- What type of modules does your project use? · esm
- Which framework does your project use? · none
- Does your project use TypeScript? · No / Yes
- Where does your code run? · browser
- What format do you want your config file to be in? · JavaScript

选项选完之后就会在项目根目录创建`.eslintrc`。如果最后一个问题选择了json就会创建`.eslintrc`并使用json语法配置ESLint的规则，如果最后一个问题选择了JavaScript就会创建`.eslintrc.js`并使用JavaScript语法配置ESLint的规则。

比如`.eslintrc.js`:

```js
/* eslint no-undef: 0 */

module.exports = {
    env: {
        browser: true,
        es2021: true,
    },
    extends: 'eslint:recommended',
    parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module',
    },
    rules: {
        'no-console': 1,
        'no-unused-vars': 0,
        quotes: [2, 'single', { avoidEscape: true, allowTemplateLiterals: true }],
    },
};

```

配置好之后在项目根目录下执行：

```shell
eslint .
```

就能检查所有js文件的问题。

rules字段可以配置相关检查的规则，key就是ESLint所报的错误，value可以配置0 / 1 / 2分别是ignore / warning / error。

ESLint默认是使用双引号，如果想用单引号，可以像上面那样配置`quotes: [2, 'single', { avoidEscape: true, allowTemplateLiterals: true }]`，具体参考[官方文档](https://cn.eslint.org/docs/rules/quotes)。

另外，`.eslintrc.js`总是会报`module.exports`有错误，报的是`no-undef`，为了解决这个问题，可以在文件的第一行加上`/* eslint no-undef: 0*/`把这个文件的`no-undef`设为ignore就可以了。

### VS Code拓展

可以安装[[WebVSCode]]的[ESLint拓展](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)，这样打开js文件时ESLint拓展就会自动检查问题并把问题通过黄线和红线标注出来。

安装后可以增加如下配置：

```json
{
    "eslint.codeAction.showDocumentation": {
        "enable": true
    },
    "eslint.validate": [
        "javascript",
        "javascriptreact",
        "vue"
    ]
}
```

如果是在项目中使用，可以把该配置加到`.vscode/settings.json`中，并设置extensions.json:

```json
{
    "recommendations": ["dbaeumer.vscode-eslint"]
}
```

## Prettier

安装Prettier到devDepencies:

```shell
npm install prettier -D
```

然后执行下面的命令就可以对某个文件执行格式化：

```shell
npx prettier --write file.js
```

格式化也可以通过rc文件进行配置，同样支持json格式和js文件，比如`.prettierrc.js`:

```js
/* eslint no-undef: 0 */

module.exports = {
    semi: true,
    singleQuote: true,
    tabWidth: 4,
    useTabs: false,
    printWidth: 120,
};

```

- semi: 是否用分号
- singleQuote: 是否使用单引号
- tabWidth: tab大小
- useTabs: 是否使用制表符
- printWidth: 单行长度

更多配置参考[官网文档](https://prettier.io/docs/en/options.html)。

### VS Code拓展

可以安装[[WebVSCode]]的[Prettier拓展](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)，在保存文件时自动格式化代码。

安装后可以增加如下配置：

```json
{
    "[javascript]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[json]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[jsonc]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[css]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    },
    "[html]": {
        "editor.defaultFormatter": "esbenp.prettier-vscode",
        "editor.formatOnSave": true
    }
}
```

也就是对指定格式的文件的格式化工具指定为Prettier拓展并在保存时执行格式化。

如果是在项目中使用，可以把该配置加到`.vscode/settings.json`中，并设置extensions.json:

```json
{
    "recommendations": ["esbenp.prettier-vscode"]
}
```
