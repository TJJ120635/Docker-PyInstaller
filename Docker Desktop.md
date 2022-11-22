# Docker desktop 的安装和使用

谭家骏 2022/11/14

​	由于开发、生产环境的机器使用的是 Linux 系统，如果要进行相关的开发工作，则需要考虑在 Windows 上搭建 Linux 环境。例如在 Linux 中使用 PyInstaller 才能生成 Linux 下的可执行文件，而不是在 Windows 中生成 exe

​	目前有两种主要的方法，第一种是使用 VMware 安装 Linux 虚拟机，第二种是使用 WSL (Windows Subsystem for Linux) 安装 Linux 发行版。WSL2 的空间、性能开销都更小，且更方便在 Windows 本机进行管理

​	使用 WSL + Docker desktop 的方式，我们就可以在本机进入容器进行管理，而不需要使用 "本机进入 Linux 虚拟机再进入容器" 这样的三层嵌套

## 1. 系统准备

系统要求：Win 10 或者 Win 11 至少  21H2 版本

### 1.1 启用虚拟化

首先需要在 BIOS 中开启虚拟化

根据主板品牌，在开机时按键进入 BIOS（例如 Dell 笔记本是 F2）

在高级设置中找到 虚拟化技术/Virtualization Technology 并开启

启用后使用任务管理器查看是否生效

<img src="Pic\虚拟化.png" style="zoom:75%;" />

### 1.2 启用 WSL2

（参考：https://learn.microsoft.com/zh-cn/windows/wsl/install）

**开启 Windows 功能**

在 "控制面版-程序-启动或关闭 Windows 功能 

启用 "适用于 Linux 的 Windows 子系统" 和 "虚拟机平台"

或者使用命令行开启

```shell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

安装过程可能要求重启电脑

<img src="Pic\启用WSL.png" style="zoom:75%;" />

**安装 WSL**

接下来在 cmd 或者 powershell 安装 WSL2

```shell
wsl --install
```

或者下载更新包手动安装 wsl 2

https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi

检查 wsl 是否启用

```shell
wsl -l
```

**设置 WSL 2**

安装完成后，在 cmd 或者 powershell 将 WSL2 设置为默认版本

```shell
wsl --set-default-version 2
```

## 2. Docker Desktop 安装配置

### 2.1 安装 Docker Desktop

下载安装包进行安装，安装时注意选择 **Use WSL 2 instead of Hyper-V**

https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe

安装完成后等待 Docker Desktop 启动，可以看到容器、镜像、挂载目录等管理界面

<img src="Pic\DockerDesktop.png" style="zoom: 67%;" />

### 2.2 安装 Windows Terminal（可选）

我们需要合适的命令行工具来管理 Docker 和容器

可以使用 Windows 自带的 cmd 和 powershell，但是非常不好用。也可以使用第三方的 MobaXterm、Xshell、SecureCRT 等

这里推荐 Windows Terminal 这款神器，对 WSL 的支持非常好，可以在 Windows 系统里混合使用 cmd 命令、PowerShell 命令、bash 命令等。兼具界面美观、操作方便等诸多优点

在 Microsoft 商店中可以搜索安装，也可以在 Github 下载安装包：

https://github.com/microsoft/terminal

### 2.3 换源（可选）

为了避免拉取镜像的网络问题，可以添加国内的镜像源

进入右上角的设置，在配置的 json 中添加镜像源相关的字段

这里使用 163 源和中科大源作为参考

```json
{
    "registry-mirrors": [
        "http://hub-mirror.c.163.com",
    	"https://docker.mirrors.ustc.edu.cn"
    ]
}
```

<img src="Pic\换源.png" style="zoom: 67%;" />

### 2.4 移动文件目录（可选）

（参考：https://www.cnblogs.com/xhznl/p/13184398.html）

WSL 系统的默认文件目录安装在 C 盘的 %LOCALAPPDATA%/Docker/wsl 下，需要手动移动到其他盘中（不能直接复制粘贴）

首先需要关闭 docker，在右下角系统托盘中退出

<img src="Pic\退出.png" style="zoom:75%;" />

接下来打开控制台，关闭所有 WSL 的发行版

```shell
wsl --shutdown
```

将数据导出到自定义位置的压缩包中，例如 D:\Download\docker-desktop-data.tar

```shell
wsl --export docker-desktop-data D:\Download\docker-desktop-data.tar
```

注销原来的 data 目录

```shell
wsl --unregister docker-desktop-data
```

指定新目录导入压缩包

前一个是自定义的新的 data 目录，我使用的是 D:\ProgramData\wsl

后一个是压缩包的目录

```shell
wsl --import docker-desktop-data D:\ProgramData\wsl D:\Download\docker-desktop-data.tar --version 2
```

移动完成后就可以重新启动 Docker Desktop

## 3. 简单使用 Docker

### 3.1 hello-world 测试

在控制台中使用 docker run 来进行一个简单的测试

```shell
docker run hello-world
```

首先 docker 会寻找本地有没有名为 hello-world 的镜像，当发现没有时，就会从镜像源进行下载

在完成下载后，会根据镜像创建一个容器并运行

容器运行的结果就是打印下面的文字在控制台上

<img src="Pic\Helloworld.png" style="zoom:75%;" />

在 docker desktop 的界面可以看到下载的镜像和创建的容器

其中容器的名称由于没有指定，因此是随机生成的

<img src="Pic\镜像.png" style="zoom:75%;" />

<img src="Pic\容器.png" style="zoom:75%;" />

### 3.2 Docker 的基本操作

**下载镜像**

使用控制台的 docker pull [image]:[tag] 来下载指定的镜像，如果不加 tag 则默认为最新版

```shell
docker pull python # python 最新镜像
docker pull python:3.7 # python3.7 镜像
docker pull continuumio/miniconda3 # miniconda 最新镜像
```

**创建容器**

下载镜像后，推荐使用 docker desktop 的镜像管理，选定镜像使用 run 来创建并启动容器，建议指定一个自定义的容器名称

<img src="Pic\创建容器.png" style="zoom:75%;" />

容器完成创建后，可以对容器进行启动/停止/删除等操作

<img src="Pic\容器管理.png" style="zoom:75%;" />

**进入容器**

当容器处于运行状态时，在控制台使用 docker exec -it [container] bash，即可进入容器内部

容器的内部相当于远程终端或虚拟机，使用的是 Linux 的命令行，可以参考：

https://juejin.cn/post/6844903930166509581

https://blog.csdn.net/leilei1366615/article/details/106267225

<img src="Pic\进入容器.png" style="zoom:75%;" />

