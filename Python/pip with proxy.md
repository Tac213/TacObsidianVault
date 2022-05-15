```shell
python -m pip install xxx --proxy=127.0.0.1:7890
```

上面的方法很麻烦，可以直接在`~`目录下，新建pip目录，在pip目录下新建pip.ini，里面些如下内容：

```ini
[global]
proxy = http://127.0.0.1:7890
```
