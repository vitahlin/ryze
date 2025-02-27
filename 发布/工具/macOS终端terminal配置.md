---
title: macOS终端terminal配置
slug: make-your-mac-terminal-beautiful
date: 2024-09-02
categories:
  - Tools
tags:
  - Terminal
featuredImage: https://sonder.vitah.me/featured/58d45254fb9ff09247d511693fe307b1.webp
desc: 在MAC的终端（Terminal）中，不仅可以通过原生的功能完成工作，还可以通过一系列的美化和配置，将其打造成一个既美观又高效的开发工具。本文将从字体选择、Terminal 主题美化到 Zsh 配置优化，为您详细介绍如何一步步提升MAC Terminal的使用体验。
---

终端软件很多，比如很流行的终端软件 [iTerm2](https://iterm2.com)，但这里不做介绍，MAC 其实原生自带了终端工具 Terminal，我们来了解一下如何美化。

## 字体

### Powerline

Powerline 是一款 Vim statusline 的插件，它用到了很多特殊的 icon 字符。Powerline fonts 是一个字体集，本质是对一些现有的字体打 patch，把 powerline icon 字符添加到这些现有的字体里去，目前对 30 款编程字体打了 patch。

字体下载：[https://github.com/powerline/fonts](https://github.com/powerline/fonts)

可以按如下命令下载并导入：

```shell
# clone
git clone https://github.com/powerline/fonts.git --depth=1
# install
cd fonts
./install.sh
# clean-up a bit
cd ..
rm -rf fonts
```

### Nerd font

Nerd font 的原理和 Powerline fonts 是一样的，也是针对已有的字体打 patch，把一些 icon 字符插入进去。不过 Nerd font 就比较厉害了，是一个“集大成者”，他几乎把目前市面上主流的 icon 字符全打进去了，包括上面刚刚提到的 powerline icon 字符以及 Font Awesome 等几千个 icon 字符。

![NerdFont.png](https://sonder.vitah.me/blog/2024/6d1bd6b3cb003528884e860c4aaae87f.webp)

比如主题 [powerlevel10k/powerlevel10k]( https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fromkatv%2Fpowerlevel10k%23oh-my-zsh " https://github.com/romkatv/powerlevel10k#oh-my-zsh" )，就推荐使用 Nerd font（虽然也支持 powerline font），显示效果如下：

![](https://sonder.vitah.me/blog/2024/e986a5bfdb98f67368ae507f20a0c158.webp)

Nerd font 对 50 多款编程字体打了 patch（具体请参考 [这里]( https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fryanoasis%2Fnerd-fonts " https://github.com/ryanoasis/nerd-fonts" )，和 Powerline fonts 类似，也会在 patch 后，对名字做一下修改，比如 `Source Code Font` 会修改为 `Sauce Code Nerd Font` (Sauce Code 并非 typo，故意为之)。

安装完之后，也需要修改对应客户端的字体（比如 iTerm/Terminal.app）后，各种主题的效果才会生效。

- Powerline fonts 或者 Nerd fonts 这些字体集，他们对已有的一些 (编程) 字体打了 patch，新增一些 icon 字符。
- Nerd fonts 是 Powerline fonts 的超集，建议直接使用 Nerd font 就好了

## Terminal 主题美化

下载主题美化 Terminal，下载地址：[https://github.com/mbadolato/iTerm2-Color-Schemes](https://github.com/mbadolato/iTerm2-Color-Schemes)

下载到本地后，打开 Terminal，导入所选主题即可，如图所示：

![Terminal主题](https://sonder.vitah.me/blog/2024/76c6585012ba59dd506b94482c5889ca.webp)

这里导入主题 OneHalfDark，效果如下：

![OneHalfDark主题效果](https://sonder.vitah.me/blog/2024/55b0612a43c4a99e84fe48c5893c4339.webp)

## Zsh 配置

主题美化后，我们需要安装一下插件来优化 zsh 操作。
首先安装 zinit，用它来管理 zsh 插件，关于 zinit 的介绍网上很多，主要特点是快并且易于配置，一个 .zshrc 文件即可拷贝全部配置。参考链接：

- [使用 zinit 管理 zsh 插件 完美代替 Antigen](https://einverne.github.io/post/2020/10/use-zinit-to-manage-zsh-plugins.html)
- [加速你的 zsh —— 最强 zsh 插件管理器 zplugin/zinit 教程](https://www.aloxaf.com/2019/11/zplugin_tutorial/)

通用配置：

```shell
# 科学上网需要
alias fq="export ALL_PROXY=127.0.0.1:9999"
alias nfq="unset ALL_PROXY"
alias ip="curl ipinfo.io"

alias vi="mvim"
alias zshrc="vi ~/.zshrc"
alias vimrc="vi ~/.vimrc"
alias rm="trash
```

安装完成后，我们还需要安装插件来优化 zsh。

### 主题优化

#### Zsh 主题 powerlevel 10k

Zshrc 代码：

```shell
# p10k主题
zinit ice depth"1" 
zinit light romkatv/powerlevel10k
```

安装完成后，终端输入 `p10k configure` 来自定义 p 10 k 主题配置。

参考链接：[https://github.com/romkatv/powerlevel10k](https://github.com/romkatv/powerlevel10k)

#### 语法高亮

```shell
# 语法高亮
zinit ice lucid wait='0' atinit='zpcompinit'
zinit light zdharma/fast-syntax-highlighting
```

参考链接：[https://github.com/zdharma-continuum/fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting)

如果需要 `ls` 命令也能高亮颜色的话，还需要如下配置：

```shell
# ls和cd命令展示文件夹颜色
export CLICOLOR=1
export LSCOLORS=ExGxFxdaCxDaDahbadeche
zstyle ':completion:*' list-colors "${(@s.:.)LS_COLORS}"
```

### 操作优化

#### 目录快速跳转 zsh-z

参考链接：[https://github.com/agkozak/zsh-z](https://github.com/agkozak/zsh-z)

```shell
# 快速跳转，需要安装lua，brew install lua
zinit ice lucid wait='1'
zinit light skywind3000/z.lua
alias zb="z -b"
```

该插件需要 `lua` 支持，可以通过 `Homebrew` 安装：`brew install lua`

#### 终端 vim 模式 zsh-vi-mode

参考链接：[https://github.com/jeffreytse/zsh-vi-mode](https://github.com/jeffreytse/zsh-vi-mode)

```shell
终端vim模式
zinit ice depth=1
zinit light jeffreytse/zsh-vi-mod
```
