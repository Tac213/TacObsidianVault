首先在官网上找到Python的源码包，可以在[这个地址](https://www.python.org/ftp/python/)中找到对应版本的下载地址。

然后通过下面的命令下载并解压源码:

```shell
wget -c https://www.python.org/ftp/python/3.10.6/Python-3.10.6.tar.xz
tar -xf Python-3.10.6.tar.xz
```

先不急着进入这个目录，先源码编译安装openssl，如果没有openssl则pip用不了，没法装包。

## 编译安装openssl

在[这个地址](https://www.openssl.org/source)中找到openssl的源码，下载并安装，通常选3+的版本。

首先还是下载解压，并进入文件夹编译：

```shell
wget -c https://www.openssl.org/source/openssl-3.0.5.tar.gz
tar -zxvf openssl-3.0.5.tar.gz
cd openssl-3.0.5
./config --prefix=/usr/local/openssl
make
sudo make install
```

注意编译完成之后最好进入安装的目录看下openpyxxl的目录结构，主要是确认lib目录的名字是lib还是lib64，以及include目录的名字与结构。

接着修改链接文件：

```shell
#备份原有链接
sudo mv /usr/bin/openssl /usr/bin/openssl.bak

#创建软链接
sudo ln -sf /usr/local/openssl/bin/openssl /usr/bin/openssl
```

最后添加路径至ld.so.conf并设置生效

```shell
sudo sh -c "echo '/usr/local/openssl/lib64' >> /etc/ld.so.conf"
sudo ldconfig -v
```

设置完成后可以重新查看openssl version:

```
$ openssl version
OpenSSL 3.0.5 5 Jul 2022 (Library: OpenSSL 3.0.5 5 Jul 2022)
```

## 编译Python

首先安装依赖:

```shell
sudo apt update
sudo apt install build-essential checkinstall zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev libbz2-dev
```

进入Python源码目录，修改`Module/Setup`文件，搜索`OPENSSL`，取消注释并改成下面这个样子：

```
OPENSSL=/usr/local/openssl
_ssl _ssl.c \ 
    -I$(OPENSSL)/include -L$(OPENSSL)/lib64 \ 
    -lssl -lcrypto
```

也有可能是下面这样子的：

```
# OpenSSL bindings
OPENSSL_INCLUDES=-I/usr/local/openssl/include
OPENSSL_LDFLAGS=-lssl
OPENSSL_LIBS=-L/usr/local/openssl/lib64
_ssl _ssl.c $(OPENSSL_INCLUDES) $(OPENSSL_LDFLAGS) $(OPENSSL_LIBS)
_hashlib _hashopenssl.c $(OPENSSL_INCLUDES) $(OPENSSL_LDFLAGS) -lcrypto
```

之前确认的openssl的目录结构就是为了让这里能写对。

接下来就很简单了，配置并编译Python:

```shell
./configure --enable-optimizations --enable-shared  # enable-shared可以编译出libpython3.xm.so.1.0之类的shared library, 以便于用python开发的应用的打包, 比如PyInstaller打包Python开发的应用
sudo make install
```

注意如果用`--enable-shared`选项的话，会编译出shared library，通常会在`/usr/local/lib`目录下，会根据编译时配置的prefix的不同而不同，默认是这个路径。需要把这个目录加到环境变量里面，否则python无法启动：

```bash
export LD_LIBRARY_PATH=/lib:/usr/lib:/usr/local/lib
```

这样就ok了，如果不详替代原本的python，可以用下面的命令编译：

```shell
sudo make altinstall
```

## 其他

升级Python后可能会遇到终端打不开的问题，此时文本编辑器开启`/usr/bin/gnome-terminal`把第一行改成`#!/usr/bin/python3.10`即可。具体版本事`/usr/bin`目录下的python版本而定。

如果提示`pip`不在`PATH`里，执行这句话即可：

```shell
export PATH=$PATH:$HOME/.local/bin
```
