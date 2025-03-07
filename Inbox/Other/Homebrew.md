
# `tap` 功能

如果你在核心仓库没有找到你需要的软件，那么你就需要安装第三方的仓库去安装你需要的软件，这时候就可以使用 `tap` 命令。

# 安装指定版本

首先，需要在Github上fork `[https://github.com/Homebrew/homebrew-core](https://github.com/Homebrew/homebrew-core)`，然后执行：

```shell
$ brew extract --version 5.6.2 gradle vitahlin/core
$ brew install gradle@5.6.2
```

`--version 5.6.2 gradle` 即指定你要安装的是 `gradle` 的 `5.6.2` 版本， `vitahlin/core` 即对应Homebrew中的 `tap` 

##### 列出所有依赖项目
```shell
$ brew deps --tree --installed autojump
autojump
└── python@3.9
    ├── gdbm
    ├── openssl@1.1
    ├── readline
    ├── sqlite
    │   └── readline
    └── xz
```

# 完美卸载软件及其依赖包

```shell
brew tap beeftornado/rmtree
brew rmtree xxx
brew cleanup
```

# 解决Cask下载太慢的问题

Casks 类软件包，多数是先下载安装文件.dmg或是.pkg，再经过一系列的静默安装过程。将其中下载的环节「外包」给优秀的下载器，就能解决下载中途卡死而无法安装的问题。

查看其下载文件所在位置的指令：`brwe --cache`。键入它，我们得到：/Users/tommy/Library/Caches/Homebrew，这是你用户名下的一个文件夹。这个文件夹里面包含了所有 Homebrew 下载过程中产生的文件。
