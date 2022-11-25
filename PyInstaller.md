# Docker desktop + PyInstaller 打包 Linux 可执行文件

谭家骏 2022/11/15

目录：

1. Docker 准备
2. 虚拟环境准备
3. 程序打包
4. 程序加密
5. 减小程序的体积

## 前言

前置准备：系统开启了 WSL2 并成功安装 Docker desktop

目前打包 Python 脚本成可执行文件主要使用的是 PyInstaller 库，且不支持跨平台打包（Windows 下只能生成 Windows 可执行文件）。为了将 Python 脚本打包成 Linux 系统的可执行文件，我们需要使用 Docker 创建一个 Linux 容器

在文件大小方面，PyInstaller 容易将 Python 环境中的所有库都打包进最终的可执行文件里，需要手动排除不需要的依赖。因此推荐创建一个纯净的 Python 环境，只安装我们需要的库

在进行 Python 3.7 打包测试时的结果如下：

| 系统    | 软件环境                 | 最终大小 |
| ------- | ------------------------ | -------- |
| Windows | anaconda                 | 42M      |
| Linux   | Python 官方镜像 + pipenv | 62M      |
| Linux   | miniconda 官方镜像       | 500+M    |

conda 的官方 docker 镜像中包含很多不必要的依赖，很难创建一个纯净的环境，会导致打包的软件大小很大。最后选择了使用 pipenv 来创建 Python 环境。

## 1. Docker 准备

### 1.1 拉取镜像

这里以 Python 3.7 为例，在控制台拉取官方镜像

```shell
docker pull python:3.7
```

如果下载时遇到网络问题，可以参考文档进行换源

<img src="Pic\下载镜像.png" style="zoom: 67%;" />

### 1.2 创建容器

在镜像下载完成后，在 Docker desktop 进行容器的创建

**Container name** 是自定义的容器名称

**Volumes** 是挂载目录的映射。这里我添加了一个挂载规则，将主机上的 D:\ProgramData\wsl\Python 目录挂载到容器的 /data 目录

通过挂载，可以让主机和容器共享同一个文件夹，可以对文件直接进行复制粘贴等操作

**Environment variables** 是环境变量，暂不需要用到

<img src="Pic\创建容器2.png" style="zoom: 67%;" />

## 2. 虚拟环境准备

### 2.1 pip 换源

创建并运行容器后，可以在 Docker desktop 的 Containers 栏里看到容器的状态

通过控制台，以命令行的方式进入到容器中

使用 Linux 的指令（bash） 即可在容器环境下进行操作

```shell
docker exec -it PyInstaller bash
```

<img src="Pic\进入容器.png" style="zoom:75%;" />

在容器中对 Python 的信息进行查看，可以看到是一个纯净的环境，没有多余的库

<img src="Pic\python信息.png" style="zoom:75%;" />

如果需要的话，使用 pip config 更换国内源，避免网络问题

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### 2.2 安装 pipenv

接下来安装 pipenv，用于创建 Python 虚拟环境

```shell
pip install pipenv
```

<img src="Pic\下载pipenv.png" style="zoom:75%;" />

## 3. 程序打包

### 3.1 搭建虚拟环境

在创建容器时，将主机的 D:\ProgramData\wsl\Python 目录挂载到了容器的 /data 目录，可以在该目录下新开一个 envs 文件夹用于创建 Python 环境

创建环境非常方便，一条指令即可

完成后会在目录下生成一个 Pipfile 用于记录环境的信息，环境本体位于 ~/.local/share/virtualenvs 目录下

```shell
cd /data/envs
pipenv --python 3.7 # 在当前目录创建虚拟环境
```

<img src="Pic\创建env.png" style="zoom:75%;" />

在创建环境的目录下使用 pipenv shell，就会根据 Pipfile 进入到新建的环境中，命令行会有提示

可以使用 pip list 查看到当前环境的包是干净的

```shell
pipenv shell # 进入到虚拟环境
```

<img src="Pic\进入env.png" style="zoom:75%;" />

接下来需要使用 pip 安装 PyInstaller 库

```shell
pip install pyinstaller
```

### 3.2 程序打包

为了打包程序，我们在挂载的目录下新建一个 code 文件夹 D:\ProgramData\wsl\Python\code

然后在文件夹下放入需要镜像打包的脚本文件

<img src="Pic\进入code.png" style="zoom:75%;" />

在虚拟环境中安装必要的库，为了减小最终大小尽量排除无关的库

这里安装的库列表如下

```shell
pip install pandas numpy matplotlib psycopg2 docx
```

<img src="Pic\安装库.png" style="zoom:75%;" />

使用 pyinstaller -F 生成单个可执行文件，使用 --clean 清除缓存

```shell
pyinstaller -F --clean env_report_before.py
```

执行完成时会生成一个 .spec 文件，根据 .spec 再生成一个 build 文件夹和 dist 文件夹

.spec 文件指定了路径变量、排除的库、添加的外部文件等信息，可以手动进行修改

build 文件夹包含了打包的依赖和加工的中间文件

dist 文件夹包含最终打包的程序结果，如果指定了 -F 则输出单个文件，否则会输出一个目录

<img src="Pic\查看结果.png" style="zoom:75%;" />

进入 dist 可以看到最终生成的可执行程序大小为 62.1 M

<img src="Pic\查看可执行程序.png" style="zoom:75%;" />

## 4. 程序加密

我们已经将 Python 脚本打包成了一个可执行文件，做到了源程序不可见。但是如果使用 pyinstxtractor 工具进行反编译，可以恢复出主程序的源码

如果需要加强程序的安全性，需要使用 --key 参数来进行加密

### 4.1 分离主程序和核心程序

PyInstaller 支持对依赖库进行加密，但是主程序不会被加密。因此为了安全性需要将数据库连接方式、算法逻辑等核心内容放在单独的 py 文件中作为依赖。主程序只作为入口调用核心库的函数

这里使用两个脚本进行测试

```Python
# test.py
import testlib

if __name__ == '__main__':
	testlib.hello()
	print('Hello from main')
```

```python
# testlib.py
import pandas
import numpy

def hello():
	print('Hello World from testlib')
	print(numpy.eye(4))
	print(pandas.Series([1,2,3]))
```

### 4.2 使用 PyInstaller 打包并加密

参考：https://zhuanlan.zhihu.com/p/109266820

在打包时加入 --key [密码] 进行加密

```shell
pyinstaller -F --clean -key 123456 test.py
```

<img src="Pic\安装tinyaes.png" style="zoom:75%;" />

在 Linux 系统下使用 --key 参数会提示报错。需要额外安装 tinyaes 库来加密。如果是 Windows 平台则是安装 pycrypto 包，安装需要调用 VS 编译器等等步骤，比较麻烦。

```shell
pip install tinyaes
```

安装 tinyaes 之后，重新使用 pyinstaller 进行加密

加密生成的 test 可执行文件，可以正常调用得到结果，且加强了程序的安全性

在体积方面，加密的程序体积只增加 0.1M 左右，可以忽略

<img src="Pic\使用加密.png" style="zoom:75%;" />

## 5 减小程序的体积（效果不好）

按照网络上的经验，第一种方式是使用 UPX 工具进行压缩，第二种方式是更改引入（import）的方式。但是目前尝试两种方式的效果都不好，这里仅做介绍

### 5.1 UPX 压缩工具

UPX 是一款压缩可执行文件的工具，PyInstaller 内置了对 UPX 的支持，但是需要自行下载 UPX

首先在 Github 下载最新的压缩包 https://github.com/upx/upx/releases

这里我将压缩包放到了 D:\ProgramData\wsl\Python 目录下

然后在容器中解压压缩包，并将文件夹重命名为 upx

解压后的文件夹里包含了说明文件以及名为 upx 的可执行文件

```shell
tar -xcJf upx-upx-4.0.1-amd64_linux.tar.xz
mv upx-4.0.1-amd64_linux upx
```

<img src="Pic\解压UPX.png" style="zoom:75%;" />

接下来有两种使用 UPX 的方式

第一种是在使用 PyInstaller 时指定 UPX 的路径

或者将 upx 文件夹加入系统 PATH 环境变量中，使用时就不需要单独指定目录

```shell
pyinstaller -F --clean --upx-dir /data/upx test2.py
```

<img src="Pic\使用UPX.png" style="zoom:75%;" />

第二种是在 PyInstaller 打包完成后再进行压缩

其中参数 --best 表示自动选择最高压缩级别（从 -1 到 -9，默认为 -7）

如果使用 --brute 和 --ultra-brute，则会尝试使用更多压缩级别和参数形成最小体积

但是从实际结果来看，三种压缩参数的效果都不好，只是从 62.1 MB 压缩到了 61.6 MB

```shell
pyinstaller -F --clean test2.py
/data/upx/upx --best dist/test2
```

<img src="Pic\使用UPX2.png" style="zoom:75%;" />

## 5.2 更改引入的方式

按照网上的说法，不引入不必要的包，以及使用 from ... import ... 的方式，都能够减少体积

这里以一个测试样例为例子

```python
# test2.py
import time # from time import localtime
import calendar # from calendar import month
import datetime # from datetime import date
import warnings # from warnings import showwarning
import imp # from imp import reload

import pandas # from pandas import Series
import numpy # from numpy import array
import matplotlib # from matplotlib.pyplot import figure
import psycopg2 # from psycopg2 import connect
import docx # from docx import Document

print('Hello World')
```

上半部分是 Python 官方库，下半部分是第三方库

同时采用了两种 import 的方式，第一种是引入整个库，第二种是引入其中的一部分

| 引入库/引入方式   | import  | from ... import |
| ----------------- | ------- | --------------- |
| 无（只有 print）  | 9.58 MB | 9.58 MB         |
| 官方库            | 9.58 MB | 9.58 MB         |
| 第三方库          | 62.1 MB | 62.1 MB         |
| 官方库 + 第三方库 | 62.1 MB | 62.1 MB         |

可以看到就算只有一条 print('Hello World') 语句，最终的可执行程序也会将整个 Python 的环境进行打包，其中包括了官方的基础库

最终决定打包大小的是 import 了哪些第三方库，使用 from ... import 的方法不能减小体积，但是减少 import 的库可以减小体积

因此最好的减小体积的方法是尽量不导入不必要的库