如果想要在vscode中使用git bash作为默认终端，需要在settings.json中增加如下配置：

```json
{
    "terminal.integrated.profiles.windows": {
        "Git-Bash": {
            "path": "F:\\Program Files\\Git\\bin\\bash.exe",
            "args": [
                "-i",
                "-l"
            ]
        },
    },
    "terminal.integrated.defaultProfile.windows": "Git-Bash",
}
```

其中path为git bash的完整路径，所以这个只能作为个人的settings，不能提交到项目里。

直接配置成git bash之后，在task里面使用诸如`${relativeFile}`之类的带有路径的绝对变量会出现问题，因为vscode默认使用当前系统的路径分隔符，windows下为back slash，比如会返回`a\b\c`之类的路径，此时bash会忽略掉这些back slash，导致出问题。

为了解决这个问题，并照顾到项目中其他使用powershell或者cmd作为默认终端的同学，可以将带有slash的环境变量的两边都加个引号，比如`"\"..\\a\\b\\{$relativeFile}\""`。

git bash中的显示可以在`Git\etc\profile.d\git-prompt.sh`这个文件中进行配置。
