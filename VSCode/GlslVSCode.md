## 拓展

- language support: [slevesque.shader](https://marketplace.visualstudio.com/items?itemName=slevesque.shader)
- linter: [dtoplak.vscode-glsllint](https://marketplace.visualstudio.com/items?itemName=dtoplak.vscode-glsllint)

## linter配置

linter需要使用[glslangValidator](https://github.com/KhronosGroup/glslang)。

直接[下载编译好的glslangValidator](https://github.com/KhronosGroup/glslang/releases)，可执行文件放到任意喜欢的位置，然后像下面这样配置linter路径即可：

```json
{
    "glsllint.glslangValidatorPath": "F:\\Program Files\\Git\\usr\\bin\\glslangValidator.exe"
}
```
