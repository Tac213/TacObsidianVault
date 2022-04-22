其实就是把若干个不同的目录都放到vscode的workspace里面，xxx.code-workspace也是一个json文件，相当于.vscode的集合体，里面可以配置launch / settings / extensions ...

每个目录仍然可以保留自己的.vscode，目录的.vscode会override code-workspace文件中的配置。

但在Multi-root Workspace中，部分环境变量不能正常使用，比如`${workspaceFolder}`。假设有2个子目录名字为A和B，那么可以用`{workspaceFolder:A}`表示A目录的路径，`{workspaceFolder:B}`表示B目录的路径。
