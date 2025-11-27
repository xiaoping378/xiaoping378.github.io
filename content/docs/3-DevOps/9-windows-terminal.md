---
tags: ["Ops"]
title: "Windows Terminal终端入坑指南"
linkTitle: "Windows Terminal终端"
weight: 9
description: >
 介绍windows terminal安装和oh-my-zsh的配置
---

{{% pageinfo %}}
为了这个东西，重新安装了系统，目前是在windows LTSC 2021版本下的使用指南。
在IT界Terminal和Console差不多是一个意思，同属于界面层面的，不少人老和Shell搞混了，特此说明下Shell一般是指的Bash、zsh、PowerShell、cmd等。
{{% /pageinfo %}}


## 安装

为了使用Windows Terminal，在春节期间，重新安装了LTSC 2021版本的系统（之前一直用的LTSC 2019）。

> 它对操作系统内部版本的最低要求为 `18362.0`，通过`Win+R`输入`winver`可以确认本机系统是否支持。

目前有三种办法安装（本人选用的第二种）:

- 是在应用商店中搜索`Windows Terminal`，安装即可。

- 通过[Github release](https://github.com/microsoft/terminal/releases)页面下载安装包，

```powershell
Add-AppxPackage Microsoft.WindowsTerminal_<versionNumber>.msixbundle
```
- 通过命令行[winget](https://github.com/microsoft/winget-cli)、[Chocolatey ](https://chocolatey.org/)、[Scoop ](https://scoop.sh/)安装，下面以winget为例：

```powershell
winget install --id=Microsoft.WindowsTerminal -e
```

## 配置

现在基本可以图形界面配置了，按照自己的习惯图形操作即可，网上一坨坨的教程，此处不表。

![](/images/windows-terminal-2022-01-30-12-53-50.png)

默认配置保存在了`%LOCALAPPDATA%\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json`

## 技巧

1. 快捷键
   - 新建终端 -- `Ctrl+Shift+t`
   - 切换终端 -- `Alt + Num` , （我这里修改过了）
2. Quake窗口
   - 快捷键是**Win + \`** , 可以快速从屏幕上半区换出终端窗口


## GitBash

![](/images/windows-terminal-2022-01-30-23-03-08.png)

本人环境`VSCode`和[git-bash](https://git-scm.com/download/win)都是绿色版本了，免去了每次重装系统，都进行各种重复的配置操作.

> 没有环境的可以自行通过上面的连接下载GitBash，后面有时间会尝试下`WSL`和WSL2。

### 中文乱码

需要添加环境变量到`~/.bashrc`或者`~/.zshrc`中。

```bash
export LANG=zh_CN.UTF-8
```

### 绿色改造

绿色改造的核心，一个是安装时不默认安装在C盘，另一个就是设置`HOME`的系统环境变量，Git-Bash每次启动是可以根据`HOME`变量，决定加载配置的路径的。

设置系统环境变量，两种办法：

- 图形界面操作： `Win+x` -> 系统 -> 高级系统设置 -> 环境变量, 自行添加`HOME`变量。

- 在PowerShell命令行中设置环境变量，执行完即可生效。
```powershell
[Environment]::SetEnvironmentVariable("HOME", "D:\xxp", "User")
```

我这里是`D:\xxp`目录，这样的话，那些`ssh、git、vscode、bash、zsh`的自定义配置，都可以免去重装再来一次的痛苦了。

还可以把日常用到`exe`小工具，也放到`$HOME/bin`目录下，再加到`PATH`环境变量里。
  
```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";D:\xxp\bin","User")
```

### oh-my-zsh主题改造

~~现在有人直接把zsh装上，然后再安装github上的oh-my-zsh主题，但是启动速度会慢一些，本来bash就比cmd启动慢了，，，~~

~~我是在bash的基础上改造下主题，修改`Git\etc\profile.d\git-prompt.sh`文件，详见[这里](https://gist.github.com/xiaoping378/8d636ddfdcf68982b93b65acbd5dcd83)~~

之前一直是Bash的基础上，修改了下主题凑活用着。其实可以直接使用zsh的，记录下大致操作。

> 之前在网上看过的的教程大多是在`bashrc`里再启动`zsh`，会慢上加慢的，我就一直没弄，后来觉得是可以做个`Git-zsh`环境的。

1. [下载](https://mirror.msys2.org/msys/x86_64/zsh-5.8-5-x86_64.pkg.tar.zst)zsh二进制

现在msys2上的安装包，都变成zst格式（Facebook家出的）的压缩包了，还需要下载解压工具，我平常使用的就是7z，这里找了个[7z with ZS](https://github.com/mcmilk/7-Zip-zstd/releases)的工具。

解压到GitBash安装的根目录上。主要是`/etc/zsh`和`/usr`目录。

2. 安装oh-my-zsh主题

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

3. terminal和vscode中使用zsh.exe

- 直接指定`zsh.exe`的路径
- 还需要在`~/.zshrc`的`PATH`里添加`/mingw64/bin`，不然会提示找不到git，

如下是Terminal的配置： 
![](/images/windows-terminal-2022-01-30-23-18-29.png)

如下是vscode中的配置：
```json
    "terminal.integrated.profiles.windows": {
        "git-bash": {
          "path": "D:\\Softwares\\Git\\usr\\bin\\zsh.exe",
          "args": []
        }
      },
    "terminal.integrated.defaultProfile.windows": "git-bash",
```