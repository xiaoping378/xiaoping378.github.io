---
tags: ["Ops"]
title: "Windows Terminal终端 入坑指南"
linkTitle: "Windows Terminal终端"
weight: 9
description: >
 介绍windows terminal从安装到配置指南
---

{{% pageinfo %}}
为了这个东西，重新安装了系统，目前是在windows LTSC 2021版本下的使用指南。
在IT界Terminal和Console差不多是一个意思，同属于界面层面的，不少人老和Shell搞混了，特此说明下Shell一般是指的Bash、zsh、PowerShell、cmd等。
{{% /pageinfo %}}


## 安装

为了使用Windows Terminal，在春节期间，重新安装了LTSC 2021版本的系统（之前的LTSC 2019）。

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

![](/images/windows-terminal-2022-01-30-13-32-26.png)

本人环境`VSCode`和`git-bash`都是绿色版本了，免去每次重装系统，都进行各种重复的配置操作，这里介绍下git-bash改造的大致步骤。

### 绿色改造

在powershell中设置环境变量，git-bash每次启动可以根据此变量，决定加载配置的路径。
```powershell
[Environment]::SetEnvironmentVariable("HOME", "D:\xxp", "User")
```

我这里是`D:\xxp`目录，这样的话，那些`ssh、git、vscode、bash`的自定义配置，都可以免去重装再来一次的痛苦了。

还可以把日常用到`exe`小工具，也放到`$HOME/bin`目录下，再加到`PATH`环境变量里。
```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";D:\xxp\bin","User")
```

### oh-my-zsh主题改造

现在有人直接把zsh装上，然后再安装github上的oh-my-zsh主题，但是启动速度会慢一些，本来bash就比cmd启动慢了，，，

我是在bash的基础上改造下主题，修改`Git\etc\profile.d\git-prompt.sh`文件：

```bash
if test -f /etc/profile.d/git-sdk.sh
then
	TITLEPREFIX=SDK-${MSYSTEM#MINGW}
else
	TITLEPREFIX=$MSYSTEM
fi

if test -f ~/.config/git/git-prompt.sh
then
	. ~/.config/git/git-prompt.sh
else
	PS1='\[\033]0;Bash\007\]'      # 窗口标题
	PS1="$PS1"'\n'                 # 换行
	PS1="$PS1"'\[\033[32;1m\]'     # 高亮绿色
	PS1="$PS1"'➜  '               # unicode 字符，右箭头
	PS1="$PS1"'\[\033[33;1m\]'     # 高亮黄色
	PS1="$PS1"'\W'                 # 当前目录
	if test -z "$WINELOADERNOEXEC"
	then
		GIT_EXEC_PATH="$(git --exec-path 2>/dev/null)"
		COMPLETION_PATH="${GIT_EXEC_PATH%/libexec/git-core}"
		COMPLETION_PATH="${COMPLETION_PATH%/lib/git-core}"
		COMPLETION_PATH="$COMPLETION_PATH/share/git/completion"
		if test -f "$COMPLETION_PATH/git-prompt.sh"
		then
			. "$COMPLETION_PATH/git-completion.bash"
			. "$COMPLETION_PATH/git-prompt.sh"
			PS1="$PS1"'\[\033[31m\]'   # 红色
			PS1="$PS1"'`__git_ps1`'    # git 插件
		fi
	fi
	PS1="$PS1"'\[\033[36m\] '      # 青色
fi

MSYS2_PS1="$PS1"

# Evaluate all user-specific Bash completion scripts (if any)
if test -z "$WINELOADERNOEXEC"
then
	for c in "$HOME"/.bash_completion.d/*.bash
	do
		# Handle absence of any scripts (or the folder) gracefully
		test ! -f "$c" ||
		. "$c"
	done
fi
```
